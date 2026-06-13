---
description: Triage live N-central monitoring alerts for a customer, site, or service org and produce a worst-first brief of failing services. Triggers on "active issues for [client]", "live alerts", "what's alerting in n-central", "monitoring alerts", "what's down right now", "service failures".
---

# N-central Active Issues

Pull live monitoring-alert state straight from N-central's own monitoring engine — the real alerts the GraphQL API has no type for. Read-only triage; no writes.

Uses the **N-central MCP** (classic N-central REST API).

## What's available
- `list_active_issues` — active monitoring issues for an org unit. ONLY accepts a **customer** or **site** org unit, NOT a service-org. To cover a whole SO, iterate its customers.
- `get_device_status` — live status of a device's service-monitoring tasks (which services are failing/warning).
- `list_job_statuses` — failed/running job context for an org unit (optional).
- `list_customers` / `list_sites` — resolve the customerId/siteId to feed `list_active_issues`.
- KNOWN BUG: `list_active_issues` `_extra.deviceClassValue`/`deviceClassLabel` are always null — do NOT rely on those fields.

This is **live N-central monitoring state**. It is distinct from the GraphQL `alert-triage` skill, which infers issues from disconnected agents, vulns, and patch errors.

## Step 1 — Resolve the org unit
`list_active_issues` rejects service-org IDs. Get the customerId (and siteIds, if scoping a site).

```json
{ "soId": 50, "all": true }
```
Call `list_customers` with the SO id (or omit `soId` for all). For a single named client, match `customerName`. Capture each `customerId`. If scoping to one site, call `list_sites` with the `customerId` and capture `siteId`.

## Step 2 — Pull active issues per customer/site
Call `list_active_issues` once per customer (or site). For a whole SO, loop over every customerId from Step 1.

```json
{ "orgUnitId": 1234, "format": "json" }
```
The response is a **bare JSON array** (no `data` wrapper). Each element is an active issue with these verified top-level fields: `deviceId`, `serviceName` (e.g. "NetPath Status"), `serviceType` (e.g. "Maint"), and `notificationState`. (`serviceId` may also be present but is unverified — don't rely on it.) Each element also carries an `_extra` object holding `deviceName` (the device label — there is NO top-level device name), `transitionTime` (when it entered this state — use as "Since"), and `customerTree` (e.g. `["System","Cove","Asian Pacific Tech"]`). Ignore `_extra.deviceClassValue`/`deviceClassLabel` (null bug). Collect rows across all customers, tagging each with its customer name from `customerTree`.

## Step 3 — Drill into the worst devices
For devices with the most/severest issues, get the live per-service breakdown to name the exact failing monitors. This feeds the "Failing service(s)" detail column only — it is not the per-row severity source (Step 2's `notificationState` is, since `get_device_status` is not called for every device).

```json
{ "deviceId": "98765" }
```
Call `get_device_status` (deviceId is a STRING). It returns `{ data: [...], totalItems }`; each task has `moduleName` (the service, e.g. "CPU", "Disk", "Patch Status v2", "EDR Status") and `stateStatus` (the live state — `Normal`, plus non-normal values like `Failed`/`Warning`/`Stale`/`Misconfigured`). **List the tasks where `stateStatus` != `Normal`**, naming them by `moduleName`, to fill the "Failing service(s)" column precisely. `lastScanTime` shows freshness.

## Step 4 — (Optional) Failed-job context
If issues look job-driven (patch runs, scripts), pull job status for the affected org unit.

```json
{ "orgUnitId": 1234, "format": "json" }
```
Call `list_job_statuses` and note any FAILED jobs near the alerting devices. Include as context only; do not pad the main table with it.

## Triage rules
- Severity order, worst-first by the `notificationState` from Step 2's `list_active_issues` (present on every row, so every device can be sorted): most-severe states first, less-severe last. Within the same state, more affected services first. Use the Step 3 `stateStatus` (`Failed` > `Warning` > `Stale`/`Misconfigured` > others) only as a finer-grained tiebreaker on the devices you actually drilled into.
- Group rows by customer (from `_extra.customerTree`).
- "Since" = `_extra.transitionTime` from `list_active_issues`. If absent, leave blank.
- One row per affected device; comma-join multiple failing services in the cell.

## Output format
**Active Issues Triage Brief — [Scope: Customer / Site / "All SOs"] — 2026-06-13**

Group the table by customer (one sub-header per customer), rows sorted worst-first. The Severity/state column is the row's `notificationState` from Step 2 (available for every device); Failing service(s) comes from Step 3 `stateStatus` only on the devices you drilled into.

### [Customer Name]
| Device | Customer | Failing service(s) | Severity/state | Since |
|--------|----------|--------------------|----------------|-------|
| SRV-DC01 | Acme Corp | DNS Service, Disk C: | FAILED | 2026-06-13 02:14 |
| WS-FINANCE-7 | Acme Corp | Antivirus Status | WARNING | 2026-06-12 22:40 |

If a customer has zero active issues, omit it from the table.

Optionally add a short **Failed jobs** note beneath a customer if Step 4 surfaced relevant failures.

Close with a one-line summary count, e.g.:

> 3 customers affected · 11 active issues · 7 failing services · 2 WARNING — live N-central monitoring state as of 2026-06-13 09:00.
