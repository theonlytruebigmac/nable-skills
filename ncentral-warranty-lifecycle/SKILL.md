---
description: Forecast hardware refresh from asset-lifecycle and warranty data, bucketing devices by warranty/replacement date against today and estimating refresh budget. Triggers on "warranty report", "hardware refresh", "asset lifecycle", "warranties expiring", "aging hardware", "replacement planning", "lease expiry".
---

# N-central Warranty & Hardware-Refresh Forecast

Forecast hardware refresh from per-device asset-lifecycle and warranty data, then optionally backfill missing dates. Uses the **N-central MCP** (classic N-central REST API).

**Today is 2026-06-13 — compute every date delta against it.**

## Key tools
- `list_devices_by_org_unit` / `report_devices_by_so` — enumerate devices under a customer, site, or service org.
- `get_device_lifecycle` — per-device warranty/lease/replacement/purchase dates, cost, location, assetTag.
- `get_device_assets` — hardware inventory (model, serial, age) for enrichment. Probes 404 — skip them.
- `patch_device_lifecycle` — PATCH partial lifecycle fields (WRITE). Dates ISO-8601 / YYYY-MM-DD.
- `update_device_lifecycle` — PUT full replace (requires ALL fields); prefer `patch_*` for backfill.

## Step 1 — Enumerate devices in scope
Resolve the org unit first via `report_org_hierarchy` if you only have a name. Then pull every device:
```json
{ "orgUnitId": 12345, "all": true, "format": "json" }
```
For a whole service org use `report_devices_by_so` `{ "soId": 50, "format": "json" }`. Note each device's `deviceId`, `customerName`/`siteName`, and `longName`.

## Step 2 — Pull lifecycle data per device
Call `get_device_lifecycle` for each `deviceId`:
```json
{ "deviceId": "987654" }
```
Capture `warrantyExpiryDate`, `leaseExpiryDate`, `expectedReplacementDate`, `purchaseDate`, `cost`, `location`, `assetTag`. A device with all of these null/empty has **NO lifecycle data** — flag it (Step 5), do not bucket it as expired.

## Step 3 — Enrich with model / serial / age (optional)
For each device, pull hardware detail to populate the Model column and sanity-check age:
```json
{ "deviceId": "987654" }
```
`get_device_assets` returns model, serial, and BIOS/purchase age. Skip on 404 (probe/agentless). Fall back to `get_device` `longName` if no model.

## Step 4 — Bucket against 2026-06-13
For each device pick the **soonest** of `warrantyExpiryDate` and `expectedReplacementDate` (use `leaseExpiryDate` if those are absent) as its refresh trigger. Compute days-left = triggerDate − 2026-06-13, then bucket:

| Bucket | Condition (days left) |
|---|---|
| Expired | < 0 |
| < 30 days | 0–29 |
| < 60 days | 30–59 |
| < 90 days | 60–89 |
| < 1 year | 90–364 |
| > 1 year | ≥ 365 |
| No data | all dates null |

Sum `cost` across devices due within the planning horizon for a rough refresh-budget total. Treat null `cost` as 0 but note the count of devices missing cost so the total reads as a floor, not exact.

## Step 5 — (Optional WRITE) Backfill missing lifecycle dates
Only when the user asks to populate missing data. Use `patch_device_lifecycle` to set just the fields supplied — never blank existing values with `update_device_lifecycle`.

**Preview + confirm gate (required before any write):** present the exact changes and stop for explicit approval.
```
Proposed lifecycle updates (3 devices):
  SRV-DC01 (deviceId 987654)  warrantyExpiryDate: (empty) -> 2027-03-01
  WS-FIN-04 (deviceId 987701) expectedReplacementDate: (empty) -> 2028-01-01
  WS-FIN-05 (deviceId 987702) purchaseDate: (empty) -> 2024-01-15
Apply these 3 updates? (yes/no)
```
Only after the user replies "yes", issue one PATCH per device:
```json
{ "deviceId": "987654", "warrantyExpiryDate": "2027-03-01" }
```

**Verification (after write):** re-call `get_device_lifecycle` for each patched `deviceId` and confirm the new value persisted. Report any device whose value did not change as a failed write.

## Output format
**Hardware Refresh Forecast — [Org/Customer] — 2026-06-13**

Bucket summary first:
| Bucket | Devices | Est. cost |
|---|---|---|
| Expired | n | $… |
| < 30 days | n | $… |
| < 60 days | n | $… |
| < 90 days | n | $… |
| < 1 year | n | $… |
| > 1 year | n | $… |
| No lifecycle data | n | — |

Then the device detail table, **sorted soonest-first** (expired at top, no-data rows last):
| Device | Customer/Site | Model | Warranty exp | Replacement due | Days left | Cost |
|---|---|---|---|---|---|---|

Mark **No lifecycle data** devices clearly in the Days-left column and exclude them from the budget total.

Close with one line: `N devices, M expiring within 12 months, rough refresh budget ~$X (floor — K devices missing cost).`
