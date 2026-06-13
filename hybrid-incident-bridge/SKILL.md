---
description: Reconstruct an incident from N-central live signals and N-able GraphQL forensic history, build a merged timeline, infer root cause, and optionally open a PSA ticket. Triggers on "incident bridge", "incident report and ticket", "write up outage and open ticket", "incident with psa ticket", "log incident to psa".
---

# Incident Bridge (Report + PSA Ticket)

Reconstruct an outage from BOTH MCPs, correlate by name, write a report, and optionally draft + open a PSA ticket. Uses **both** the N-able MCP (GraphQL) and the N-central MCP (classic REST).

> **ID-SPACE WARNING:** the two MCPs use DIFFERENT id spaces. A GraphQL organization/asset id is NOT an N-central numeric `customerId`/`deviceId`. NEVER pass an id from one MCP to the other. Correlate ONLY on stable human attributes: exact customer NAME, and device NAME/hostname (OS as tiebreaker). If a device appears in only one system, flag it — it may be unmanaged on one side or named differently — and have the operator confirm the match before merging.

Ask for the **customer name** and the **time window** (start/end, UTC) before doing anything.

## Step 1 — Resolve IDs in both systems by name
GraphQL side — find the organization and its assets:
```graphql
query ResolveOrg($name: String!) {
  organizationSearch(where: { name: { contains: $name } }) {
    nodes { id name }
  }
}
```
N-central side — resolve the numeric customer + devices:
```json
{ "tool": "report_org_hierarchy", "format": "json" }
```
```json
{ "tool": "list_devices_by_org_unit", "orgUnitId": 1234, "all": true }
```
`validate` then `execute` the GraphQL query (never `execute` an unvalidated query). Build a name->{graphqlAssetId, ncentralDeviceId} map. Note any unmatched devices for the operator.

## Step 2 — N-central live signals (what alerted, what failed)
```json
{ "tool": "list_active_issues", "orgUnitId": 1234, "format": "json" }
```
`list_active_issues` works on customer/site org units only (not service-orgs). For each affected device pull live service-monitor state:
```json
{ "tool": "get_device_status", "deviceId": "5678" }
```
Then job/task failures inside the window:
```json
{ "tool": "list_job_statuses", "orgUnitId": 1234, "format": "json" }
```
```json
{ "tool": "get_scheduled_task_status", "taskId": "9012", "detailed": true }
```
`taskId` is a string. `detailed: true` returns a per-device breakdown but is rejected for DEVICE-level task ids — pass a SYSTEM/CUSTOMER-level (parent) task id, navigating up via `parentId` if needed. Tag every fact pulled here as source **[N-central]**.

## Step 3 — GraphQL forensic timeline (agent + patch history)
Agent connection changes and patch install failures during the window. `assetSearch` cannot filter by id (no `id` field on `AssetWhereInput`); fetch the mapped GraphQL asset ids directly with the `assets(ids:)` root query, which returns `[Asset!]!` (no `nodes` wrapper):
```graphql
query IncidentForensics($assetIds: [ID!]!, $orgId: ID!, $from: DateTime!, $to: DateTime!) {
  assets(ids: $assetIds) {
    name systemInfo { hostname }
    agentConnection { status statusChangedAt }
    lastBootedAt reboot { isRequired }
  }
  patchInstallationSearch(inOrganizations: [$orgId], where: {
    installationStatus: { equals: ERROR }
    lastUpdatedAt: { gte: $from, lte: $to }
  }) {
    nodes {
      asset { name } patch { name severity }
      failureCount errorDetails { message occurredAt } lastUpdatedAt
    }
  }
}
```
`validate` first, then `execute`. Tag every fact pulled here as source **[N-able]**.

## Step 4 — Merge into one chronological timeline + infer root cause
Sort all events ascending by timestamp into a single table. Each row keeps its source tag. Correlate the first alert ([N-central]) against agent disconnect / patch error / reboot ([N-able]) to infer the root cause. State confidence and call out any single-source events the operator should confirm.

## Step 5 — Draft the PSA ticket (preview — do NOT create yet)
`create_custom_psa_ticket` is **Custom PSA only** and keys the ticket to the N-central numeric `customerId` resolved in Step 1 — do NOT source ticket fields from `list_psa_companies` (that is the separate Standard PSA subsystem and its company id is not valid here). Compose the ticket body (summary + merged timeline) and SHOW the full draft to the operator:

```json
{
  "tool": "create_custom_psa_ticket",
  "body": {
    "customerId": 1234,
    "subject": "[Incident] <customer> — <short cause> — <window>",
    "description": "<summary + merged timeline with [source] tags + root cause + impact>",
    "priority": "High"
  }
}
```
**STOP. Show the operator the exact ticket above and ask: "Create this PSA ticket? (yes/no)". Do NOT call `create_custom_psa_ticket` without an explicit yes.** If no, deliver the report only.

## Step 6 — Create and verify (only after confirmation)
On a yes, call `create_custom_psa_ticket` with the confirmed body, then verify it landed:
```json
{ "tool": "list_custom_psa_tickets" }
```
Confirm the new ticket appears and return its ticket id. If creation returned an error, report it and do not retry silently.

## Output format

**Incident Report — [Customer] — [Window UTC]**

- **Summary** — 2-3 sentences: what broke, blast radius, current state.
- **Timeline** — merged, ascending by time:

  | Time (UTC) | Source | Event | Device |
  |---|---|---|---|

  Source is `[N-central]` or `[N-able]`. Single-source rows flagged with `(confirm)`.
- **Affected systems** — device name, hostname, role; mark any unmatched/single-system devices.
- **Root cause** — inferred cause + confidence (high/medium/low).
- **Impact** — services/users affected and duration.
- **Resolution** — what restored service (or "ongoing").
- **Next steps** — preventive actions.

Then the **Drafted PSA Ticket** block and the confirmation gate. After a confirmed create, append: `Ticket created: #<id>`.

End with a one-line count: `N events merged (X N-central / Y N-able) across Z devices; PSA ticket: <#id | not created>.`
