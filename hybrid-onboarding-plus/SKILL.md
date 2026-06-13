---
description: Audit full onboarding completeness for a customer across technical (N-able GraphQL) and business/config (N-central REST) baselines, correlating by name. Triggers on "onboarding plus", "full onboarding check [client]", "is [client] fully onboarded", "onboarding across both systems", "complete onboarding audit".
---

# Onboarding+ (Both MCPs)

Combined onboarding scorecard that supersets the GraphQL `onboarding-checklist` skill with N-central business/config checks. Uses **both** the N-able MCP (GraphQL) and the N-central MCP (classic REST). Do NOT modify `onboarding-checklist`; this is the deeper, cross-system version.

## ID-space warning (read first)
The two MCPs use **different ID spaces**. A GraphQL organization/asset id is NOT an N-central `customerId`/`orgUnitId`/`deviceId`. NEVER pass an id from one MCP into the other. Correlate only on stable human attributes: customer **name** and device **name/hostname** (OS as tiebreaker). If a device appears on only one side, flag it — it may be unmanaged on one side or named differently. Confirm the name match with the operator before merging.

## Step 1 — Resolve IDs in both systems by name
GraphQL side — find the customer org id. Always `validate` before `execute`; name every operation.
```graphql
query ResolveOrg { organizationSearch(where: { name: { contains: "Acme" } }) { nodes { id name } } }
```
N-central side — find the numeric `customerId` independently.
```json
{ "all": true, "format": "json" }
```
The `{ all, format }` block above is for `list_customers`. To resolve IDs flat instead, use `report_org_hierarchy` with only `{ "format": "json" }` (it has no `all` param). Match the customer **name** exactly. For a customer, its `customerId` IS its org-unit id, so the same number feeds both `list_devices_by_org_unit` (as `orgUnitId`) and `get_psa_customer_mapping` (as `customerId`) downstream. Record both ids side by side; everything downstream forks per MCP.

## Step 2 — Technical baseline (N-able GraphQL)
Pull the asset roster for the customer org. `validate` then `execute`.
```graphql
query OnboardTech($org: ID!) {
  assetSearch(first: 200, inOrganizations: [$org]) {
    nodes {
      id name systemInfo { hostname }
      isManaged
      agentConnection { status statusChangedAt }
      patchManagement { status lastPatchScanTime }
      tags { nodes { name } }
      operatingSystemInfo { name type }
    }
  }
}
```
Score four technical items across the roster: **Agents deployed/connected** (`agentConnection.status` CONNECTED), **Patch management ACTIVE** (`patchManagement.status`), **Tags applied** (non-empty `tags`), **Managed coverage** (`isManaged` true). PASS if the item holds for every (or your baseline %) asset; else GAP with the offending device names.

## Step 3 — Business/config baseline (N-central REST)
Enumerate the customer's devices, then audit config. `orgUnitId` here is the customer's org-unit id from Step 1 (== its `customerId`). Build the device list first:
```json
{ "orgUnitId": 1234, "all": true }
```
`list_devices_by_org_unit` returns `deviceId` strings. Name-match these against the Step 2 roster. Then run the config checks:

- **Custom properties populated** — fan out across all devices:
```json
{ "orgUnitId": 1234, "dataType": "custom-properties", "format": "json" }
```
`report_devices_bulk` — confirm required MSP props (backup vendor, contract tier, LOB app) are non-blank. Get the required set from `list_org_custom_properties`.
- **Maintenance windows set** — `get_maintenance_windows` `{ "deviceId": "..." }` per server; empty = GAP.
- **PSA mapping exists** — `get_psa_customer_mapping` `{ "customerId": 1234 }`; no mapping = GAP.
- **Lifecycle/warranty entered** — `get_device_lifecycle` `{ "deviceId": "..." }`; blank `warrantyExpiryDate`/`assetTag` = GAP.
- **Monitoring services configured** — `get_device_status` `{ "deviceId": "..." }`; empty `data` array (no service modules) = GAP (no monitoring). Service name is `moduleName`, live state is `stateStatus` — there is no "task list" on this endpoint.
- **Runbook notes present** — `list_device_notes` `{ "deviceId": "..." }`; key servers with zero notes = GAP.

Tag every fact with its source MCP in the report (N-able vs N-central) so the operator knows which system to fix in.

## Step 4 — (Optional) remediate gaps — preview + confirm gate
This skill is read-only by default. If the operator asks you to **fix** a gap (e.g. backfill a custom property, set a maintenance window, write a runbook note), you are mutating N-central — STOP and gate it:
1. Show the exact change as a table: device name, `deviceId`, field, current value -> new value, and the tool that will run (`update_device_custom_property`, `create_maintenance_windows`, `patch_device_lifecycle`, `add_device_note`/`add_notes_bulk`).
2. Ask the operator to confirm explicitly. Do not write until they reply yes.
3. After writing, **verify**: re-read with the matching getter (`get_device_custom_property`, `get_maintenance_windows`, `get_device_lifecycle`, `list_device_notes`) and confirm the new value landed. `create_maintenance_windows` returns per-device partial failures — inspect each.

## Output format
**Onboarding+ Report — [Customer] — 2026-06-13**

Resolved IDs: GraphQL org `<id>` / N-central customer `<customerId>` — confirm name match.

### Technical (N-able)
| Item | Status | Detail / offending devices |
|---|---|---|
| Agents connected | PASS/GAP | ... |
| Patch mgmt ACTIVE | PASS/GAP | ... |
| Tags applied | PASS/GAP | ... |
| Managed coverage | PASS/GAP | ... |

### Business/Config (N-central)
| Item | Status | Detail / offending devices |
|---|---|---|
| Custom properties | PASS/GAP | ... |
| Maintenance windows | PASS/GAP | ... |
| PSA mapping | PASS/GAP | ... |
| Lifecycle/warranty | PASS/GAP | ... |
| Monitoring services | PASS/GAP | ... |
| Runbook notes | PASS/GAP | ... |

### Remediation (prioritized)
Ordered list, highest-impact GAPs first (monitoring/agent gaps before notes), each naming the device(s), the fix, and which MCP owns it.

Close with one line: **Onboarding completeness: X% (N of M items PASS)** plus any name-match items appearing in only one system.
