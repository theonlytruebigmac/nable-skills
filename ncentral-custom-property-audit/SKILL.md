---
description: Audit MSP metadata completeness in N-central custom properties and optionally bulk-fill missing values with preview+confirm. Triggers on "custom property audit", "metadata audit", "missing custom properties", "which devices missing [property]", "backup vendor field", "contract tier", "bulk set property".
---

# N-central Custom Property Audit

Audit custom-property completeness (backup vendor, contract tier, primary app, etc.) across devices and org units, then optionally bulk-fill the blanks. Uses the **N-central MCP** (classic N-central REST API).

Writes (filling values) are gated behind a preview + explicit confirmation and verified afterward.

## What's available
- `list_org_custom_properties` — property DEFINITIONS for an org unit (device-prop defaults live here too). Gives propertyId, propertyName, propertyType.
- `report_devices_bulk` with `dataType: "custom-properties"` — the workhorse: fans out a per-device custom-property pull across every device under an org unit.
- `list_device_custom_properties` / `get_device_custom_property` — single-device props.
- `get_org_custom_property_default` — the org-level default value for a property at an SO/customer/site.
- `update_device_custom_property` — WRITE; PUT requires `propertyName` + `propertyType`. `propertyType` ∈ HTML_LINK | TEXT | DATE | ENUMERATED | PASSWORD and MUST match the definition.
- `update_org_custom_property_default` — WRITE; set an org default with `propagationType` controlling inheritance.

Resolve `orgUnitId` with `report_org_hierarchy` if the operator gives a name.

## Step 1 — Discover property definitions
```json
{ "orgUnitId": 1001, "all": true, "format": "json" }
```
`list_org_custom_properties` returns each definition's `propertyId`, `propertyName`, and `propertyType`. Confirm with the operator which properties are REQUIRED (e.g. Backup Vendor, Contract Tier, Primary App) — only those count toward completeness.

## Step 2 — Bulk pull device values
```json
{ "orgUnitId": 1001, "dataType": "custom-properties", "format": "json" }
```
`report_devices_bulk` (json) returns a **bare array** of per-device rows of custom-property values (an empty `[]` when no devices have properties — not a `{data}` wrapper). This is the audit dataset. Increase `concurrency` for large orgs; expect probes/agentless to be sparse.

## Step 3 — Find blanks
For each REQUIRED property, flag every device whose value is empty, null, or whitespace. Build:
- a completeness count per property (populated vs blank), and
- a per-property list of the deviceIds + device names missing it.

Spot-check a single device with `list_device_custom_properties { "deviceId": "12345" }` if a row looks ambiguous.

## Step 4 — Org-level defaults (optional)
For SO/customer/site metadata, read the current default before proposing a change:
```json
{ "orgUnitId": 1001, "propertyId": 42 }
```
`get_org_custom_property_default` shows what new/inheriting devices currently get.

## Step 5 — Preview the fill (REQUIRED before any write)
If the operator wants to bulk-fill, assemble the full change set and STOP. Present a table and ask for explicit confirmation:

| deviceId | device | property | propertyType | current | new value |
|----------|--------|----------|--------------|---------|-----------|
| 12345 | DC01 | Backup Vendor | TEXT | (blank) | Veeam |

State: "About to write N values across M devices. Reply 'confirm' to proceed." Do NOT call any update tool until the operator confirms. Verify `propertyType` for each property matches the Step 1 definition exactly — a mismatch errors.

For any ENUMERATED property in the change set, fetch its allowed values first — `list_org_custom_properties` does NOT return them. Call `get_device_default_custom_property` (orgUnitId + propertyId) or `get_device_custom_property` on a sample device; both return the enumerated value list. Confirm each proposed value is a member of that list before writing.

## Step 6 — Execute the fill (only after confirmation)
Per device, per property:
```json
{ "deviceId": "12345", "propertyId": 42, "propertyName": "Backup Vendor", "propertyType": "TEXT", "value": "Veeam" }
```
`update_device_custom_property` is a PUT — send `propertyName` + `propertyType` every call. For ENUMERATED props pass a `value` from the allowed list fetched in Step 5 (via `get_device_default_custom_property` / `get_device_custom_property`), and also pass the optional `enumeratedValueList` argument (array of allowed values) alongside it. To set an org default instead:
```json
{ "orgUnitId": 1001, "propertyId": 42, "propertyName": "Backup Vendor", "defaultValue": "Veeam", "propagate": true, "propagationType": "SERVICE_ORGANIZATION_AND_CUSTOMER_AND_SITE" }
```
`update_org_custom_property_default` — `propagationType` MUST be one of: NO_PROPAGATION, SERVICE_ORGANIZATION_ONLY, SERVICE_ORGANIZATION_AND_CUSTOMER, SERVICE_ORGANIZATION_AND_SITE, SERVICE_ORGANIZATION_AND_CUSTOMER_AND_SITE, CUSTOMER_AND_SITE, CUSTOMER_ONLY, SITE_ONLY, ORGANIZATION_ONLY, DEVICE_ONLY (and the DEVICE-inclusive variants). Pick the one matching which levels should inherit. Inspect any per-device failures returned.

## Step 7 — Verify
Re-run Step 2 (`report_devices_bulk`, `dataType: "custom-properties"`) or `get_device_custom_property` per touched device and confirm each new value persisted. Report any that did not take.

## Output format
**Custom Property Audit — [Org Unit / Customer] — [Date]**

Completeness matrix, one row per REQUIRED property, sorted by % complete ascending (worst first):

| Property | # Populated | # Blank | % Complete |
|----------|-------------|---------|------------|
| Backup Vendor | 42 | 8 | 84% |
| Contract Tier | 30 | 20 | 60% |

Then, per property with blanks, a list of devices missing it (deviceId — device name), sorted by device name.

**Remediation plan** — bulleted: which properties to fill, proposed values/source, and whether device-level or org-default propagation.

If a fill was executed, append a **Changes applied** line: N values written across M devices, V verified, F failed.

Close with one line: overall metadata-completeness % across all required properties (total populated / total required cells) and device/property counts audited.
