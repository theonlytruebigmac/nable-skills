---
description: Build a complete executive account rollup for one customer by merging N-able GraphQL ops data with N-central business/asset data, correlated by customer name. Triggers on "customer 360", "everything about [client]", "full account review [client]", "account health deep dive", "client overview".
---

# Customer 360 (Both MCPs)

A single executive customer card combining live ops (GraphQL) with business, asset, and ticketing data (N-central). Uses **both** the N-able MCP (GraphQL) and the N-central MCP (classic REST).

> **IDs don't cross MCPs** — never pass an id between them; join on customer NAME + device name/hostname (OS as tiebreaker), and flag any single-system match. See [cross-MCP correlation](../docs/ncentral-mcp-reference.md#cross-mcp-correlation).

Tag every fact in the report with its source MCP.

This skill is read-only. No mutations.

## Key tools/types
- GraphQL: `organizationSearch`, `assetSearch`, `patchInstallationAggregations`, `vulnerabilityDetectionAggregations` — via `validate` then `execute`.
- N-central: `list_customers` / `report_org_hierarchy`, `list_active_issues`, `list_devices_by_org_unit`, `get_device_lifecycle`, `get_org_unit_limits`, `report_devices_bulk`, `list_custom_psa_tickets` / `get_psa_customer_mapping`.

## Step 1 — Resolve IDs in both systems by name
Resolve the same customer on each side independently.

GraphQL — get the org id. Always `validate` before `execute`; name every operation.
```graphql
query Resolve360Org { organizationSearch(first: 5, where: { name: { contains: "Acme" } }) { nodes { id name } } }
```
N-central — get the numeric `customerId` and its `siteId`s.
```json
{ "soId": null, "all": true }
```
Call `list_customers` (or `report_org_hierarchy` to also pull site IDs and contacts). Match on the EXACT customer name. If only one system has the customer, STOP and confirm the name match with the operator — it may be unmanaged on one side or named differently.

## Step 2 — Fleet size, OS mix, agent status, tags (GraphQL)
```graphql
query Fleet360($orgId: ID!) {
  assetSearch(first: 500, inOrganizations: [$orgId]) {
    nodes { id name customer { name } operatingSystemInfo { name type }
      agentConnection { status } isManaged tags { nodes { name } } }
  }
}
```
`validate` then `execute`. Bucket by `operatingSystemInfo.name`, count `agentConnection.status` (ACTIVE vs disconnected), tally `isManaged=false`. Source: **N-able GraphQL**.

## Step 3 — Security posture and patch compliance (GraphQL)
```graphql
query Posture360($orgId: ID!) {
  vulnerabilityDetectionAggregations(inOrganization: $orgId) {
    status { buckets(size: 5) { key count } }
    vulnerability { severity { buckets(size: 5) { key count } } }
  }
  patchInstallationAggregations(inOrganization: $orgId) {
    status { buckets(size: 7) { key count } }
  }
}
```
`validate` then `execute`. Aggregations take a single `inOrganization` (not a list) and return `buckets { key count }`. Read the unresolved-CVE total off the vulnerability `status` buckets and break severity down via `vulnerability.severity`; report the patch `status` mix as installed vs pending/error. Source: **N-able GraphQL**.

## Step 4 — Live monitoring issues (N-central)
`list_active_issues` works ONLY on customer/site org units, not service-orgs. Query the `customerId` once (issues roll up from its sites) — don't also sum each `siteId`, or you double-count. Use per-site calls only if you need a per-site breakdown.
```json
{ "orgUnitId": 4002, "format": "json" }
```
Count active issues for the customer. Known bug: `_extra.deviceClassValue/Label` are always null — ignore. Source: **N-central**.

## Step 5 — Hardware refresh outlook (N-central)
Pull lifecycle per device. Get device IDs from `list_devices_by_org_unit` on the `customerId`, then `get_device_lifecycle` per device.
```json
{ "deviceId": "98231" }
```
Roll up `warrantyExpiryDate` and `expectedReplacementDate`: count devices expired / expiring within 12 months. Source: **N-central**.

## Step 6 — License utilization (N-central)
```json
{ "orgUnitId": 4002 }
```
`get_org_unit_limits` on the `customerId` returns rows of `{ limitName, value, maxValue }` where `value` = the configured/licensed limit and `maxValue` = the ceiling — NEITHER is current usage. Read the device-limit rows (`MaxDevicesProfessional` / `MaxDevicesEssential` / `MaxDevicesMobile`) for the licensed limits. Derive actual usage by counting devices from the `list_devices_by_org_unit` enumeration already pulled in Step 5 (per license class), then report `used / licensed` = device-count / `value`. Source: **N-central**.

## Step 7 — Metadata completeness (N-central)
```json
{ "orgUnitId": 4002, "dataType": "custom-properties", "format": "json" }
```
`report_devices_bulk` fans the custom-property call across every device. Count devices with blank/missing key properties (backup vendor, contract tier, LOB app). Source: **N-central**.

## Step 8 — Open PSA tickets (N-central)
```json
{ "customerId": 4002 }
```
`get_psa_customer_mapping` to confirm the PSA link, then `list_custom_psa_tickets` for open Custom-PSA tickets. If no mapping exists, report "PSA not mapped". Source: **N-central**.

## Output format
**Customer 360 — [Customer] — [today]**
GraphQL org id: `[id]` · N-central customerId: `[id]` (name-matched)

| Section | Metric | Source |
|---|---|---|
| Fleet & OS | total assets, top 3 OS, unmanaged count | N-able GraphQL |
| Agent status | ACTIVE vs disconnected | N-able GraphQL |
| Security posture | unresolved CVEs | N-able GraphQL |
| Patch compliance | installed vs pending/error | N-able GraphQL |
| Live issues | active issues (customer total) | N-central |
| Hardware refresh | warranty-expired, replace ≤12mo | N-central |
| License utilization | used (device count) / licensed (limit value) | N-central |
| Metadata completeness | devices missing key props | N-central |
| Open tickets | open Custom-PSA count | N-central |

Sort the table in the row order above. Flag any cross-system name mismatches under the table.
End with a one-line account-health takeaway: **"[Customer]: [GREEN/AMBER/RED] — [single biggest risk or 'no concerns']."** plus a one-line count: `N assets · M CVEs · K active issues · J open tickets`.
