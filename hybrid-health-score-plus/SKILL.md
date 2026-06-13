---
description: Compute a composite health score for a customer enriched with N-central monitoring, lifecycle, maintenance, metadata, and license signals on top of the GraphQL dimensions. Triggers on "health score plus", "full health score [client]", "enriched health score", "health score with n-central", "complete health rating".
---

# Health Score+ (Both MCPs)

Superset of the GraphQL `health-score` skill: keep every GraphQL dimension, then layer in N-central operational signals for a fuller rating. Read-only — no mutations.

Uses **both** the N-able MCP (GraphQL) and the N-central MCP (classic REST).

## ID-SPACE WARNING
The two MCPs use DIFFERENT ID SPACES. A GraphQL organization/asset id is NOT an N-central numeric `customerId`/`orgUnitId`/`deviceId`. NEVER pass an id from one MCP to the other. Correlate by exact customer NAME (and device name/hostname for per-device merges). If a customer or device appears in only one system, flag it — it may be unmanaged on one side or named differently — and have the operator confirm the match before merging.

## Dimensions and weights (combined set)
GraphQL: Agent connectivity (15), Patch compliance (15), Vuln posture (15), EOL OS (10), Managed coverage (10).
N-central: Monitoring-service health (12), Hardware lifecycle (8), Maintenance coverage (5), Metadata completeness (5), License headroom (5).
Total = 100. If a dimension has no data, drop it and redistribute its weight proportionally across the remaining dimensions; note the gap in output.

## Step 1 — Resolve IDs in both systems by name
GraphQL side — find the organization id. Always `validate` before `execute`; name the op.
```graphql
query ResolveOrg { organizationSearch(where:{name:{contains:"Acme"}}) { nodes { id name } } }
```
N-central side — resolve the matching customer id. Pull the full list and match by name (the `select` FIQL filter only supports queryable fields; matching client-side avoids "Field not found").
```json
{ "tool": "list_customers", "args": { "all": true, "format": "json" } }
```
Find the row whose `customerName` equals "Acme"; its `customerId` is also the org unit id for N-central calls. Confirm the two records are the same customer by name before proceeding. Capture the GraphQL org id (GraphQL calls only) and the numeric `customerId`/`orgUnitId` (N-central calls only).

## Step 2 — GraphQL dimensions (unchanged)
Pull the asset-level facts. `validate` then `execute`.
```graphql
query HealthAssets($org: ID!) {
  assetSearch(first: 500, inOrganizations: [$org]) {
    nodes {
      name systemInfo { hostname }
      agentConnection { status }
      patchManagement { status lastPatchScanTime }
      operatingSystemInfo { name type featureRelease }
      reboot { isRequired } isManaged
    }
  }
}
```
Vuln posture (separate query):
```graphql
query HealthVulns($org: ID!) {
  vulnerabilityDetectionSearch(inOrganization: $org, where:{ status:{ in:[UNRESOLVED] } }) {
    nodes { asset { name } customer { name } vulnerability { severity riskScore hasExploit } }
  }
}
```
Compute: % agents CONNECTED, % patch COMPLETED/compliant, vuln score (penalize high riskScore/hasExploit), % non-EOL OS, % isManaged. [GraphQL]

## Step 3 — N-central monitoring-service health
List active monitoring issues for the customer org unit (customer/site only — iterate sites if needed). For a sample/all devices, confirm service-task state.
```json
{ "tool": "list_active_issues", "args": { "orgUnitId": 50, "format": "json" } }
```
```json
{ "tool": "get_device_status", "args": { "deviceId": "12345" } }
```
Raw = % devices with NO failing service-monitoring tasks. [N-central]

## Step 4 — Hardware lifecycle
Pull warranty/replacement data per device.
```json
{ "tool": "get_device_lifecycle", "args": { "deviceId": "12345" } }
```
Raw = % devices in-warranty (warrantyExpiryDate in future) AND not past `expectedReplacementDate` (use today 2026-06-13). Devices missing lifecycle data count as unknown — exclude from the denominator and note coverage. [N-central]

## Step 5 — Maintenance coverage
```json
{ "tool": "get_maintenance_windows", "args": { "deviceId": "12345" } }
```
Raw = % devices with at least one maintenance window defined. [N-central]

## Step 6 — Metadata completeness
Fan out custom properties across all devices in the org unit.
```json
{ "tool": "report_devices_bulk", "args": { "orgUnitId": 50, "dataType": "custom-properties", "format": "json" } }
```
Raw = % of required MSP props (e.g. backup vendor, contract tier, line-of-business app) populated across devices. Define the required set up front; report which props drag the score. [N-central]

## Step 7 — License headroom
```json
{ "tool": "get_org_unit_limits", "args": { "orgUnitId": 50 } }
```
`get_org_unit_limits` returns `data:[{limitName, value, maxValue}]` where `value` = the configured limit and `maxValue` = the ceiling — NEITHER is current usage. Get actual usage by counting devices for the org unit:
```json
{ "tool": "list_devices_by_org_unit", "args": { "orgUnitId": 50, "format": "json" } }
```
Per device class, count devices against the matching limit row (`MaxDevicesProfessional`/`MaxDevicesEssential`/`MaxDevicesMobile`).
Raw = headroom = (limit − device_count) / limit, where limit = the row's `value` (lower headroom = lower raw). [N-central]

## Step 8 — Score and grade
For each dimension: contribution = raw% × weight (after any redistribution from Step's "no data" rule). Overall = sum of contributions, 0–100. Grade: A ≥90, B 80–89, C 70–79, D 60–69, F <60. Rank dimensions by lost points (weight − contribution) to find drag factors.

## Output format
**Health Score+ — [Customer] — 2026-06-13**

Overall: **NN / 100 — Grade X**

| Dimension | Source | Raw | Weight | Contribution |
|---|---|---|---|---|
| Agent connectivity | GraphQL | 96% | 15 | 14.4 |
| Patch compliance | GraphQL | ... | 15 | ... |
| Vuln posture | GraphQL | ... | 15 | ... |
| EOL OS | GraphQL | ... | 10 | ... |
| Managed coverage | GraphQL | ... | 10 | ... |
| Monitoring-service health | N-central | ... | 12 | ... |
| Hardware lifecycle | N-central | ... | 8 | ... |
| Maintenance coverage | N-central | ... | 5 | ... |
| Metadata completeness | N-central | ... | 5 | ... |
| License headroom | N-central | ... | 5 | ... |

Sort the table by Contribution ascending (worst dragging dimension first).

**Top drag factors** — top 3 dimensions by lost points, one line each (what's missing, how many devices).

**How to get to an A** — bullet list of concrete actions tied to the drag factors (e.g. "approve N pending patches", "add maintenance windows to N devices", "populate backup-vendor prop on N devices").

**Data gaps** — list any dimension dropped for missing data and the weight redistributed.

Close with: "Scored across N devices — M GraphQL dimensions, K N-central dimensions — name-correlated on customer '[Customer]'."
