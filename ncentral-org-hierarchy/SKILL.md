---
description: Export the full Service Org → Customer → Site hierarchy and resolve the numeric IDs every other N-central skill needs. Triggers on "org hierarchy", "export customers and sites", "list all customers", "SO customer site tree", "org structure", "customer site summary", "find customer id".
---

# N-central Org Hierarchy Export

Export the Service Org → Customer → Site tree with IDs, contacts, and device counts. This is the go-to lookup for resolving the `soId` / `customerId` / `siteId` / `orgUnitId` values the other N-central skills consume.

Uses the **N-central MCP** (classic N-central REST API). Read-only — no mutations.

## Org model
- Three levels: **Service Org** (`soId`) → **Customer** (`customerId`) → **Site** (`siteId`).
- A generic numeric `orgUnitId` addresses ANY level — use it when a tool wants an org unit regardless of tier.
- IDs resolved here feed `list_devices_by_org_unit`, `list_active_issues`, `report_devices_by_so`, `list_org_custom_properties`, and most other skills.

## What's available
- `report_org_hierarchy` — flat SO→Customer→Site rows with IDs, names, contacts, addresses. Best single source for resolving IDs.
- `report_customer_site_summary` — customers + sites + device counts.
- Drill tools: `list_service_orgs`, `list_customers`, `list_sites`, `list_org_units`, `list_org_unit_children`.
- Detail tools: `get_service_org`, `get_customer`, `get_site`, `get_org_unit`.

## Step 1 — Pull the flat hierarchy
Get every SO→Customer→Site row with IDs and contacts in one call.
```json
{ "tool": "report_org_hierarchy", "args": { "format": "csv" } }
```
> **Size warning:** on a large partner this report is HUGE (~1.3M characters for ~49 SOs / ~157 customers) and will be persisted to a file rather than returned inline. Prefer `format: "csv"` (far more compact than JSON), and for a big partner scope to one branch via Step 3 instead of pulling the whole tree. Each row carries `orgUnitId`, `orgUnitName`, `orgUnitType`, `parentId`, contact, and address.

## Step 2 — Add device counts
Layer customer/site device counts onto the tree.
```json
{ "tool": "report_customer_site_summary", "args": { "format": "json" } }
```
Join on `customerId` / `siteId` to attach counts to each node from Step 1.

## Step 3 — Drill a single branch (optional)
When the user names one SO or customer, scope the pulls instead of the whole fleet.
```json
{ "tool": "list_customers", "args": { "soId": 50, "all": true } }
```
```json
{ "tool": "list_sites", "args": { "customerId": 1234, "all": true } }
```
For an arbitrary org unit whose tier is unknown, walk children directly:
```json
{ "tool": "list_org_unit_children", "args": { "orgUnitId": 1234 } }
```

## Step 4 — Resolve a specific ID (lookup mode)
When the ask is "find the customer id for [name]", pull the flat report and match by name.
```json
{ "tool": "report_org_hierarchy", "args": { "format": "json" } }
```
Filter rows where the customer/site name matches; return the numeric ID(s). If the user already has one ID and wants its parent/details, use `get_customer` / `get_site` / `get_service_org` / `get_org_unit`.

## Notes
- Always pass `all: true` on `list_*` drill tools to auto-paginate; page size caps at 200.
- Names can collide across SOs — when reporting an ID, always show its parent SO/customer for disambiguation.
- `select` (FIQL/RSQL, e.g. `soId==50`) filters rows, not fields; use it to narrow `list_customers` / `list_sites` server-side.

## Output format
**Org Hierarchy — [Service Org or "All"] — 2026-06-13**

Indented tree, sorted alphabetically by name within each tier:
```
[SO] Acme Service Org  (soId 50)
  [Customer] Northwind Traders  (customerId 1234) — 87 devices
    [Site] HQ Seattle  (siteId 9001) — 60 devices
    [Site] Branch Tacoma  (siteId 9002) — 27 devices
  [Customer] Globex  (customerId 1235) — 12 devices
    [Site] Main  (siteId 9010) — 12 devices
```
Then an ID lookup table for the IDs other skills need:

| Level | Name | ID | Parent | Devices |
|---|---|---|---|---|
| Customer | Northwind Traders | 1234 | Acme Service Org | 87 |
| Site | HQ Seattle | 9001 | Northwind Traders | 60 |

Sort the table by Level (SO, Customer, Site), then name.

Close with a one-line count: "N service orgs, N customers, N sites, N total devices." Offer the CSV export (`report_org_hierarchy` with `format: "csv"`) for spreadsheet use.
