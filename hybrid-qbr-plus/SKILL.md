---
description: Enrich the GraphQL QBR brief with N-central hardware refresh forecast, ticket themes, and license utilization, correlated by customer name across both MCPs. Triggers on "qbr plus", "full QBR [client]", "QBR with hardware and tickets", "business review deep", "enriched quarterly review".
---

# QBR+ (Both MCPs)

Build a superset QBR data brief: the standard GraphQL ops view plus N-central business data (hardware budget, support activity, license headroom). Read-only — do not modify anything.

Uses **both** the N-able MCP (GraphQL) and the N-central MCP (classic REST).

## ID-space warning

The two MCPs use DIFFERENT ID SPACES. A GraphQL organization/asset id is NOT an N-central numeric `customerId`/`deviceId`. NEVER pass an id from one MCP into the other. Correlate ONLY on stable human attributes: exact customer NAME, and device NAME/hostname (OS as tiebreaker). If a customer or device appears in only one system, FLAG it and ask the operator to confirm the match before merging — it may be unmanaged on one side or named differently.

## What's available

- GraphQL (N-able): `assetSearch`, `patchInstallationAggregations`, `vulnerabilityDetectionAggregations`, `organizationSearch`. Source tag: **[GQL]**.
- N-central (REST): `list_customers`, `report_org_hierarchy`, `list_devices_by_org_unit`, `get_device_lifecycle`, `list_custom_psa_tickets`, `get_org_unit_limits`, `report_devices_bulk`. Source tag: **[NC]**.

## Step 1 — Resolve IDs in both systems by name

Resolve the customer independently on each side. Do NOT reuse ids across MCPs.

GraphQL side — validate before execute (never `execute` an un-`validate`d query):
```graphql
query QbrPlusOrg($name: String!) {
  organizationSearch(first: 5, where: { name: { contains: $name } }) {
    nodes { id name }
  }
}
```

N-central side:
```json
{ "all": true }
```
Call `list_customers` (or `report_org_hierarchy` to also pull contacts/addresses), match the same customer NAME, and capture the numeric `customerId` / `orgUnitId`. State both resolved ids and confirm the name match before proceeding.

## Step 2 — Fleet summary [GQL]

```graphql
query QbrPlusFleet($org: ID!) {
  assetSearch(first: 200, inOrganizations: [$org]) {
    nodes {
      name isManaged
      operatingSystemInfo { name type }
      agentConnection { status }
      patchManagement { status lastPatchScanTime }
      chassis { types }
    }
  }
}
```
Count managed vs unmanaged, online vs disconnected, server vs workstation. Validate, then execute.

## Step 3 — Patch compliance & vulnerability posture [GQL]

```graphql
query QbrPlusPosture($org: ID!) {
  patchInstallationAggregations(inOrganization: $org) {
    status { buckets(size: 7) { key count } }
  }
  vulnerabilityDetectionAggregations(inOrganization: $org) {
    status { buckets(size: 5) { key count } }
    vulnerability { severity { buckets(size: 5) { key count } } }
  }
}
```
Aggregations take a single `inOrganization` (not a list) and return `buckets { key count }`. Roll up the patch `status` mix and read unresolved counts off the vulnerability `status` buckets, broken down by `vulnerability.severity`. If a prior snapshot exists, note the trend vs last snapshot; otherwise mark trend N/A.

## Step 4 — Hardware refresh & budget [NC]

Pull devices for the customer (use the N-central `customerId`/`orgUnitId` from Step 1, never the GraphQL org id), then lifecycle per device.
```json
{ "orgUnitId": 4002, "all": true }
```
`list_devices_by_org_unit` to enumerate (`all:true` auto-paginates), then `get_device_lifecycle` per `deviceId`:
```json
{ "deviceId": "98231" }
```
Bucket each device by the earlier of `warrantyExpiryDate` / `expectedReplacementDate` relative to today (2026-06-13):
- Q1 (now–2026-09-13), Q2 (–2026-12-13), Q3 (–2027-03-13), Q4 (–2027-06-13), Beyond.
Sum `cost` per bucket for estimated spend. Devices with no lifecycle data go to an "Unknown" bucket — flag them.

## Step 5 — Support activity [NC]

```json
{}
```
`list_custom_psa_tickets` — count tickets, group by status and by recurring theme (subject keywords). Report total volume and top 3 themes. If Custom PSA is not configured, mark the section "no PSA data".

## Step 6 — License utilization & metadata coverage [NC]

```json
{ "orgUnitId": 4002 }
```
`get_org_unit_limits` (customer/SO level) returns rows of `{ limitName, value, maxValue }` where `value` = the configured (licensed) limit and `maxValue` = the ceiling — NEITHER is current usage. Read `value` as the limit per license class (device rows are `MaxDevicesProfessional` / `MaxDevicesEssential` / `MaxDevicesMobile`). Derive actual usage by counting the managed devices enumerated in Step 4 (`list_devices_by_org_unit`), correlated to license class, then compute Headroom = `value` (limit) − counted usage. If an org unit returns no limit, mark it `n/a`. Then audit asset/property coverage:
```json
{ "orgUnitId": 4002, "dataType": "custom-properties", "format": "json" }
```
`report_devices_bulk` fans out per-device; compute the % of devices with required custom properties populated.

## Step 7 — Merge & flag mismatches

Join GQL fleet to NC lifecycle/tickets per the Step 1 name match. List any device present in only one system under a "Name-match review" note. Do not guess a match — leave unmatched rows explicit.

## Output format

**QBR+ Brief — [Customer] — 2026-06-13**

**Fleet Summary [GQL]**
| Metric | Count |
| --- | --- |
Managed / Unmanaged, Online / Disconnected, Servers / Workstations.

**Patch & Vulnerability Posture [GQL]**
| Patch Status | Count |  — and —  | Vuln Severity | Count |
Note trend vs last snapshot (or N/A).

**Hardware Refresh & Budget [NC]**
| Quarter | Devices Due | Est. Spend |
| --- | --- | --- |
Rows Q1→Q4→Beyond→Unknown, sorted by quarter; closing line = total devices due in next 4 quarters and total estimated spend.

**Support Activity [NC]**
| Theme | Tickets |
Top 3 themes, sorted by count desc; include total ticket volume.

**License Utilization [NC]**
| License | Used | Limit | Headroom |
Used = managed device count per license class (from Step 4); Limit = `value` from `get_org_unit_limits`; Headroom = Limit − Used. Plus metadata coverage % of devices with required custom properties.

**Name-match review**
List customers/devices found in only one MCP that need operator confirmation.

Close with one line: total assets [GQL], devices due for refresh + estimated spend [NC], open tickets [NC], license headroom [NC].
