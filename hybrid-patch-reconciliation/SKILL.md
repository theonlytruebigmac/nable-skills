---
description: Reconcile GraphQL patch state against N-central maintenance windows to determine whether unpatched devices are actually scheduled to patch. Triggers on "patch reconciliation", "are unpatched devices scheduled", "patch vs maintenance windows", "reconcile patch status", "why isn't [device] patching".
---

# Patch Reconciliation (Both MCPs)

Answer "are our unpatched devices actually scheduled to patch?" by joining patch state from GraphQL against maintenance-window and service-health facts from N-central.

Uses **both** the N-able MCP (GraphQL) and the N-central MCP (classic REST).

## ID-space warning (read first)

The two MCPs use **different ID spaces**. A GraphQL asset/organization id is NOT an N-central numeric `deviceId`/`customerId`/`orgUnitId`. **Never pass an id from one MCP to the other.** Correlate only on stable human attributes — customer NAME and device NAME/hostname (OS as tiebreaker). Tag every fact below with its source: **[GraphQL]** or **[N-central]**.

## What's available

- **[GraphQL]** `patchInstallationSearch` — unpatched/failed records (`installationStatus` AVAILABLE = missing, ERROR = failing).
- **[GraphQL]** `assetSearch` — `patchManagement.status` (ACTIVE/INACTIVE) per asset.
- **[N-central]** `get_maintenance_windows {deviceId}` — is there an `enabled` patch window (cron schedule) covering this device?
- **[N-central]** `get_device_status {deviceId}` — is the patch monitoring service healthy?
- **[N-central]** `list_devices` / `list_customers` / `report_org_hierarchy` — resolve names to numeric ids.
- **[N-central]** `generate_patch_comparison_report` (async, optional) — catalog-wide approved-vs-installed view.

This skill is **read-only** — no mutations, no confirmation gate required.

## Step 1 — Pull unpatched/failing devices [GraphQL]

`validate` then `execute`. Never `execute` an unvalidated query. Name every operation.

```graphql
query UnpatchedDevices {
  patchInstallationSearch(where: {
    or: [
      { installationStatus: { equals: AVAILABLE } }
      { installationStatus: { equals: ERROR } }
    ]
  }) {
    nodes {
      asset { id name }
      customer { name }
      patch { name severity classification }
      status
      failureCount
      errorDetails { message occurredAt }
      lastUpdatedAt
    }
  }
}
```

The `installationStatus` filter only supports `equals`/`notEquals` (no `in`) — `or` two `equals` to span AVAILABLE + ERROR. The status is selected on the node as `status` (the filter input is named `installationStatus`; the output field is `status`). Collapse to one row per asset: count missing (AVAILABLE) and failed (ERROR) patches. Keep `asset.name` and `customer.name` for correlation.

## Step 2 — Pull patch-management status per asset [GraphQL]

```graphql
query PatchMgmtStatus($org: ID!) {
  assetSearch(first: 200, inOrganizations: [$org]) {
    nodes {
      name
      systemInfo { hostname }
      operatingSystemInfo { name }
      customer { name }
      patchManagement { status lastPatchScanTime }
    }
  }
}
```

Resolve `$org` with `organizationSearch` by exact customer name. Record `patchManagement.status` (ACTIVE/INACTIVE) per asset name.

## Step 3 — Resolve IDs in both systems by name

The GraphQL `asset.id` is useless to N-central. Resolve the N-central `deviceId` from the same NAME:

```json
{ "tool": "list_customers", "args": { "all": true } }
```

```json
{ "tool": "list_devices", "args": { "all": true, "select": "customerId==12345" } }
```

Match each unpatched GraphQL asset to an N-central device by exact `name` (then hostname, then OS as tiebreaker) to obtain its numeric `deviceId`. **If a device appears in only one system, do NOT guess a match** — flag it: it may be unmanaged on one side or named differently. Confirm the name match with the operator before merging.

## Step 4 — Check maintenance window + service health [N-central]

For each matched `deviceId`:

```json
{ "tool": "get_maintenance_windows", "args": { "deviceId": "987654" } }
```

```json
{ "tool": "get_device_status", "args": { "deviceId": "987654" } }
```

A non-empty `data` array with an `enabled` recurring schedule (`cron` + `duration`) = the device is covered by a patch window. There is no "active right now" field in the response — if you need to know whether a window is open at this moment, evaluate `cron` + `duration` against `get_server_time` rather than expecting an active-state flag. Inspect `get_device_status` for the patch monitoring task — failed/stale = service problem.

## Step 5 — (Optional) catalog cross-check [N-central]

```json
{ "tool": "generate_patch_comparison_report", "args": { "startDate": "2026-06-01", "installStatuses": ["FAILED", "PENDING"], "patchApprovals": ["NOT_APPROVED"] } }
```

Returns a `reportId` (async). Poll with `get_report {reportId}` for the approved-vs-installed catalog view when you need fleet-level confirmation.

## Step 6 — Diagnose each unpatched device

Apply in order:

- **(a) Will catch up — watch it:** missing patches + active window **[N-central]** + `patchManagement.status` ACTIVE **[GraphQL]**.
- **(b) No schedule — root cause:** missing patches + NO active window **[N-central]**. Action: create/assign a maintenance window.
- **(c) Agent/policy problem:** `patchManagement.status` INACTIVE **[GraphQL]** (regardless of window). Action: fix agent/policy.
- **(d) Real failure — investigate:** `installationStatus` ERROR **[GraphQL]** + window present **[N-central]**. Pull `errorDetails` and `get_device_status`.

## Output format

**Patch Reconciliation — [Customer or Fleet] — [Date]**

Group rows by diagnosis bucket (a, b, c, d — buckets b/c/d first, a last). Within each bucket sort by failed-then-missing patch count descending.

| Device | Customer | Missing/Failed (GraphQL) | Maintenance window? (N-central) | Patch mgmt active? (GraphQL) | Diagnosis / Action |
|--------|----------|--------------------------|---------------------------------|------------------------------|--------------------|

List name-mismatch / single-system devices in a separate **Unmatched — confirm manually** table (Device | Found in | Note).

Close with one line of counts per bucket, e.g.: `(b) No schedule: 7 | (c) Inactive mgmt: 3 | (d) Failing: 2 | (a) Will catch up: 14 | Unmatched: 1`.
