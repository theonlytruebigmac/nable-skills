---
description: Build a hardware and software inventory from N-central asset data for a customer or org unit. Triggers on "asset inventory", "hardware inventory", "software inventory", "what's installed on [device]", "CPU RAM disk report", "asset details", "serial numbers".
---

# N-central Hardware & Software Inventory

Compile per-device hardware (CPU/RAM/disk/OS/serial) and installed-software inventory from N-central asset records. Read-only.

Uses the **N-central MCP** (classic N-central REST API).

## What's available
- `report_devices_bulk` — fans out a per-device asset pull across every device in an org unit (`dataType:"assets"`). This is the bulk workhorse. Probes return 404 and are skipped automatically.
- `get_device_assets` — full asset record for one device: CPU, memory, disks, OS, BIOS, serial, installed software (`application.list`).
- `list_devices_by_org_unit` — device names/roles/IDs to cross-reference against asset records.
- `report_org_hierarchy` — resolve SO/customer/site names and IDs when the user names a client.

Org hierarchy: Service Org (`soId`) -> Customer (`customerId`) -> Site (`siteId`). A numeric `orgUnitId` addresses any level. `deviceId` is a string in these tools.

## Step 1 — Resolve the org unit
If the user named a customer or site, resolve its `orgUnitId`.

```json
{ "format": "csv" }
```
Call `report_org_hierarchy` and pick the matching customer/site row. Skip if the user already gave a numeric org unit.

## Step 2 — Pull device names for cross-reference
```json
{ "orgUnitId": 388, "all": true, "format": "json" }
```
Call `list_devices_by_org_unit` (`all:true` here because the inventory must cover every device in the org unit) and keep `deviceId -> longName / deviceClass / customerName / siteName`. You'll join this to the asset rows.

## Step 3 — Bulk asset pull across the org unit
```json
{ "orgUnitId": 388, "dataType": "assets", "format": "csv" }
```
Call `report_devices_bulk`. This invokes the asset endpoint per device and returns one combined result — in `json` form a **bare array** of per-device rows (empty `[]` if none, no `{data}` wrapper). Probes (and devices with no asset agent) return 404 and are dropped — track which `deviceId`s came back empty so you can report them. Tune `concurrency` only on very large org units.

## Step 4 — Per-device deep dive (optional)
When the user asks "what's installed on [device]" or wants full detail on one box:

```json
{ "deviceId": "987654" }
```
Call `get_device_assets`. Everything is nested under **`data._extra`** as keyed sub-objects/lists: `os` (`{ supportedos, installdate, serialnumber, licensekey, lastbootuptime }`), `memory.list[]` (`{ capacity, type, manufacturer, location }`), `processor`, `mediaaccessdevice.list[]` and `physicaldrive.list[]` (disks), `logicaldevice.list[]` (volumes + free space), and `osfeatures` (PowerShell/.NET/WindowsUpdate versions). Installed software is in `application.list[]`. Read fields off these sub-objects — there are no flat top-level asset fields.

## Step 5 — Build the software rollup
From the installed-software lists across all devices, count installs per application name (normalize obvious version-suffix noise) and rank descending. This surfaces standard apps, license-bearing software, and one-off installs worth flagging.

## Output format
**Asset Inventory — [Customer/Org Unit] — [today YYYY-MM-DD]**

Hardware table, sorted by Customer/Site then Device:

| Device | Customer/Site | CPU | RAM | Disk | OS | Serial |
|--------|---------------|-----|-----|------|----|--------|

- RAM as total GB; Disk as total capacity (note low-free disks if relevant).
- Use the device name from Step 2; fall back to `deviceId` if unnamed.

**Installed Software — Top by Install Count**

| Application | Version(s) | Installs |
|-------------|-----------|----------|

Sorted by install count descending; show the top 15-20 plus a one-line "X other apps" tail.

Notes:
- Probes are excluded (no asset data).
- List any devices that returned no asset data (404/empty), by name and `deviceId`.

Close with: **N devices inventoried, M skipped (probes/no-data) out of T total.**
