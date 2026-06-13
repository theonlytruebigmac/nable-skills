---
description: Report license/usage limits versus actual device consumption per org unit for capacity planning and billing reconciliation. Triggers on "license usage", "capacity report", "how many devices licensed", "license limits", "am I over capacity", "usage by customer", "license headroom".
---

# N-central License & Capacity Report

Compare licensing/usage limits against actual device counts across the partner to find near-capacity and over-capacity org units.

Uses the **N-central MCP** (classic N-central REST API).

## What's available
- `get_org_unit_limits` — licensing/usage limits for an org unit (the capacity ceiling). Service Org or customer level.
- `report_customer_site_summary` — customers + sites + device counts in one report (fastest source for "used").
- `list_devices_by_org_unit` — fallback for exact device counts; use `all: true` and take the array length.
- `report_org_hierarchy` — flat SO -> Customer -> Site with IDs/names; use to resolve `orgUnitId` and label rows.
- `list_service_orgs` / `list_customers` — enumerate the org units to audit.

This skill is **read-only**. Do not call `update_org_unit_limits` — capacity changes are out of scope.

## Step 1 — Resolve org units to audit
Pull the hierarchy first so every row has a name and ID. Scope to a single customer if the user named one; otherwise audit the whole partner.

```json
{ "format": "json" }
```
`report_org_hierarchy` returns SO/customer/site IDs. Collect the `customerId` (and `soId`) values you will iterate.

## Step 2 — Get the limit (ceiling) per org unit
Call `get_org_unit_limits` once per SO and customer you are auditing.

```json
{ "orgUnitId": 101 }
```
Returns `{ message, data: [ { limitName, value, maxValue } ] }` — a LIST of named limits, NOT a single number. `value` = the configured limit; `maxValue` = the ceiling it can be raised to. **Neither is current consumption** — get actual usage from device counts in Step 3. For device capacity, read the `MaxDevices*` rows (`MaxDevicesProfessional`, `MaxDevicesEssential`, `MaxDevicesMobile`); other rows cover features (`MaxAVDefenderNodes`, `MaxDevicesDiskEncryption`, `MaxBackupManager*`, `MaxProbes`, etc.). If a relevant limit row is absent or `value` is 0/unset, mark it `n/a` and skip the utilization math for that row.

## Step 3 — Get the actual used count per org unit
Prefer the summary report — one call covers every customer with device counts:

```json
{ "format": "json" }
```
`report_customer_site_summary` gives device counts per customer. For an org unit not covered or when you need an exact live count, fall back to:

```json
{ "orgUnitId": 101, "all": true, "format": "json" }
```
`list_devices_by_org_unit` with `all: true` auto-paginates; the device count is the array length.

## Step 4 — Compute utilization and flags
The device count from Step 3 is a single total, so reconcile it against a single total device-capacity limit: **sum the three per-class rows** — `limit = MaxDevicesProfessional + MaxDevicesEssential + MaxDevicesMobile` (treat a missing/unset row as 0; if all three are absent, mark the org unit `n/a`). Compare the total device count against that summed limit.

For each org unit with a numeric limit:
- `% = round(used / limit * 100)`
- `headroom = limit - used`
- Flag `WARN` when `% >= 85` and `% < 100`.
- Flag `OVER` when `% >= 100` (headroom is zero or negative).
- Rows with no limit are `n/a` — list them but do not flag.

Sort all rows by `%` descending so the tightest org units surface first.

## Output format
**License & Capacity Report — [Partner/Customer] — 2026-06-13**

Lead with any flagged rows called out in a one-line banner, e.g. `2 over capacity, 3 near capacity`.

| Org Unit | Limit | Used | % | Headroom | Flag |
|---|---|---|---|---|---|
| Acme Corp | 250 | 263 | 105% | -13 | OVER |
| Globex | 100 | 91 | 91% | 9 | WARN |
| Initech | 500 | 220 | 44% | 280 | — |

- Sort by `%` descending. `OVER` rows always appear at the top of the sort.
- Show `n/a` in Limit/%/Flag for org units with no configured limit.
- Use `—` for the Flag column when the org unit is under 85%.

Close with a one-line partner roll-up: total licensed vs total used and aggregate headroom. **Sum over leaf org units (customers) only — exclude SO-level rows from the totals**, since an SO's limit and device pool already aggregate its child customers, so adding both double-counts. (Still show the SO rows in the table; just don't add them into the roll-up.) e.g. `Partner total: 850 licensed / 574 used (276 headroom) across 3 customers — 1 over, 1 near.`
