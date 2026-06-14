---
description: Produce a complete, spreadsheet-ready device export at any scope — whole fleet, org unit, or service org — with optional asset/property/monitor enrichment. Triggers on "export all devices", "device spreadsheet", "full fleet csv", "devices by service org", "export fleet for [SO]", "device list export".
---

# N-central Fleet Export

Generate a spreadsheet-ready CSV of devices at any scope and hand back the file. Read-only. Uses the **N-central MCP** (classic N-central REST API).

## What's available
- `list_devices` — every device for the user. Whole-fleet export.
- `list_devices_by_org_unit` — devices under one `orgUnitId` (any level: SO, customer, or site).
- `report_devices_by_so` — full device report for one service org by `soId`.
- `report_devices_bulk` — fans a per-device call across an org unit; `dataType` ∈ `assets` | `custom-properties` | `monitor-status`. Use to append enrichment columns.
- `report_org_hierarchy` / `list_service_orgs` / `list_customers` — resolve `soId` / `customerId` / `orgUnitId` from a name.

Export rules:
- Always pass `format:"csv"` for spreadsheet output (`list_*` default json; `report_*` default csv).
- On `list_devices` / `list_devices_by_org_unit`, pass `all:true` for a complete export. `report_devices_by_so` takes only `soId`+`format` (no `all`/`select`/`pageSize`) and already fetches every device.
- Global call conventions (all/format/pageSize/select): docs/ncentral-mcp-reference.md#global-call-conventions.

## Step 1 — Resolve the scope ID

If the user named a service org or customer, resolve its ID first. Call `report_org_hierarchy` (flat SO -> Customer -> Site with IDs/names) or `list_service_orgs` / `list_customers` to map the name to a `soId` / `customerId` / `orgUnitId`. Confirm the matched org before exporting.

```json
{ "format": "csv" }
```
(`report_org_hierarchy` takes only `format`.)

## Step 2 — Export the device rows

Whole fleet:
```json
{ "all": true, "format": "csv" }
```
Call `list_devices` with the above.

By org unit (customer or site):
```json
{ "orgUnitId": 12345, "all": true, "format": "csv" }
```
Call `list_devices_by_org_unit`.

By service org:
```json
{ "soId": 50, "format": "csv" }
```
Call `report_devices_by_so`.

Pre-filter rows with `select` when the user asks for a subset (e.g. only one customer's devices, or a class):
```json
{ "all": true, "format": "csv", "select": "customerId==4021" }
```

## Step 3 — Enrich (optional)

If the user wants hardware, MSP metadata, or live monitor columns appended, fan out across the org unit:

```json
{ "orgUnitId": 12345, "dataType": "assets", "format": "csv" }
```
Call `report_devices_bulk`. Swap `dataType` for the columns wanted:
- `assets` — hardware/software inventory (CPU, RAM, disk, OS).
- `custom-properties` — MSP metadata (backup vendor, contract tier, line-of-business app).
- `monitor-status` — current service-monitoring state per device.

Join enrichment to the base export on `deviceId`. Run separate `report_devices_bulk` calls per `dataType` if multiple column sets are requested.

## Step 4 — Assemble and hand back

Combine the base device CSV with any enrichment CSVs (keyed on `deviceId`). Deliver the CSV to the user. Then summarize counts by customer, OS, and device class so they can sanity-check coverage before opening the file.

## Output format

Deliver the raw CSV (base columns plus any enrichment columns), then a summary block:

**Fleet Export — [Scope] — [Date]**

Base columns (from the chosen list/report tool): `deviceId`, `longName`, `customerId`, `customerName`, `siteName`, `deviceClass`, `osName`, `lastLoggedInUser`, `isManaged`, `lastReboot` (exact set depends on the tool). Enrichment appends the requested `assets` / `custom-properties` / `monitor-status` columns, joined on `deviceId`.

Then three summary tables, each sorted by Count descending:

| Customer | Devices |
|---|---|

| OS | Devices |
|---|---|

| Device Class | Devices |
|---|---|

Close with a one-line total: `N devices exported across M customers — scope: [Whole fleet | OU <id> | SO <id>].`
