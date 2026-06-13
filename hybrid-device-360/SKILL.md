---
description: Merge both MCPs into one complete picture of a single device, correlated by name/hostname since IDs differ between systems. Triggers on "everything about [device]", "full picture of [device]", "device 360", "deep dive [device]", "complete device report".
---

# Device 360 (Both MCPs)

One device's complete picture by merging live and historical data from both systems. Uses **both** the N-able MCP (GraphQL) and the N-central MCP (classic REST).

> **ID-space warning:** The two MCPs use DIFFERENT ID spaces. A GraphQL `asset` id is NOT the N-central numeric `deviceId`. NEVER pass an id from one MCP to the other. Correlate ONLY on stable human attributes: device NAME and/or hostname (OS as tiebreaker). Tag every fact in the output with its source MCP.

This skill is read-only — no mutations.

## Key tools/types
- GraphQL: `assetSearch` (resolve id, agent, OS, tags) → `get_single_asset_details_and_metrics` (CPU/mem), `vulnerabilityDetectionSearch` (open CVEs), `patchInstallationSearch` (per-patch), `list_script_tasks_for_asset` (automation). Always `validate` before `execute`; name every operation.
- N-central: `list_devices` / `list_devices_by_org_unit` (resolve `deviceId`), `get_device`, `get_device_status` (live failing services), `list_device_custom_properties`, `get_device_lifecycle`, `get_maintenance_windows`, `list_device_notes`.

## Step 1 — Resolve IDs in both systems by name
Find the device on each side independently, then match on name/hostname.

GraphQL — validate then execute:
```graphql
query Find360Asset($name: String!) {
  assetSearch(first: 5, where: { name: { contains: $name } }) {
    nodes { id name systemInfo { hostname } customer { name }
      operatingSystemInfo { name } }
  }
}
```

N-central — list devices and match the same name client-side. `select` is FIQL/RSQL equality (`field==value`), not wildcard/glob, and not all fields are queryable — so pull the page and match `longName`/`uri` in code rather than relying on a wildcard predicate:
```json
{ "tool": "list_devices", "all": true, "format": "json" }
```

State the match explicitly, e.g. "GraphQL asset `abc123` ⇄ N-central deviceId `4471` (matched on hostname SERVER01)." If the device appears in only ONE system, FLAG it — it may be unmanaged on one side or named differently — and confirm with the operator before merging.

## Step 2 — GraphQL: agent, tags, security, patch, automation
Agent status, OS, tags (validate → execute):
```graphql
query Asset360($id: ID!) {
  asset(id: $id) { name agentConnection { status statusChangedAt }
    operatingSystemInfo { name } reboot { isRequired } lastBootedAt
    patchManagement { status lastPatchScanTime } tags { nodes { name } } }
}
```
Open CVEs for this asset:
```graphql
query Asset360Vulns($name: String!) {
  vulnerabilityDetectionSearch(where: { status: { in: [UNRESOLVED] }, asset: { name: { equals: $name } } },
    orderBy: [{ field: RISK_SCORE, direction: DESC }]) {
    nodes { vulnerability { cveIdentifier severity riskScore hasExploit hasRansomwareCampaignUse } firstDetectedAt patchableStatus }
  }
}
```
Per-patch install/failure detail:
```graphql
query Asset360Patches($name: String!) {
  patchInstallationSearch(where: { asset: { name: { equals: $name } } }) {
    nodes { patch { name severity classification } status failureCount errorDetails { message occurredAt } lastUpdatedAt }
  }
}
```
Live metrics and recent automation. `get_single_asset_details_and_metrics` takes `id` (the GraphQL assetId, NOT `assetId`) plus a `currentTimestamp` you set to now:
```json
{ "tool": "get_single_asset_details_and_metrics", "id": "<graphql-id>", "currentTimestamp": "2026-06-13T00:00:00Z" }
```
```json
{ "tool": "list_script_tasks_for_asset", "assetId": "<graphql-id>", "where": { "task": { "type": { "equals": "TASK_SUBTYPE_RUN_SCRIPT" } } } }
```

## Step 3 — N-central: live monitoring, metadata, lifecycle, schedule, notes
Live monitored SERVICES (GraphQL lacks these). Returns `{ data: [...], totalItems }`; each row has `moduleName` (service) and `stateStatus` (`Normal` or non-normal like `Failed`/`Warning`/`Stale`). List the ones where `stateStatus` != `Normal`:
```json
{ "tool": "get_device_status", "deviceId": "4471" }
```
```json
{ "tool": "get_device", "deviceId": "4471" }
```
MSP metadata, warranty/age, patch window, runbook notes:
```json
{ "tool": "list_device_custom_properties", "deviceId": "4471" }
```
```json
{ "tool": "get_device_lifecycle", "deviceId": "4471" }
```
```json
{ "tool": "get_maintenance_windows", "deviceId": "4471" }
```
```json
{ "tool": "list_device_notes", "deviceId": "4471" }
```

## Step 4 — Merge into one card
Combine both sides. Where both systems report agent/OS state, show both and note any disagreement. Tag each line `[N-able]` or `[N-central]`.

## Output format
**Device 360 — [Device Name] — [Date]**

**Identity** `[N-central get_device + N-able asset]` — name, hostname, customer/site, OS, deviceId (N-central) + asset id (N-able), match basis.

**Live Monitoring**
| Source | Signal | State |
|---|---|---|
| `[N-central]` | failing services (`get_device_status`) | name + status |
| `[N-able]` | agent (`agentConnection`) | status + statusChangedAt |
Reboot required and lastBootedAt `[N-able]`.

**Security (CVEs)** `[N-able]`
| CVE | Severity | Risk | Exploit | Ransomware | First seen | Patchable |
|---|---|---|---|---|---|---|
Sort by Risk desc.

**Patch** — per-patch install/failure `[N-able]`; maintenance window/schedule `[N-central]`.
| Patch | Class | Status | Failures | Last update |
|---|---|---|---|---|

**Metrics** `[N-able]` — CPU %, memory %, latest sample time.

**Lifecycle / Warranty** `[N-central]` — warrantyExpiry, purchaseDate, expectedReplacement, cost, location, assetTag. Flag if warranty expired or past replacement date.

**Metadata** — custom properties `[N-central]` + tags `[N-able]`.

**Notes** `[N-central]` — most recent first.

**Recent Automation** `[N-able]` — script task name, status, last run.

End with a one-line summary: `Device [name]: N open CVEs, N failing services, N patch failures — present in [both systems | N-able only | N-central only].`
