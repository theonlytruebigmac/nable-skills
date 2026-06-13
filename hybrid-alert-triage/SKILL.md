---
description: Triage REAL N-central monitoring alerts, then enrich each alerting device with GraphQL CVE and patch-failure context to produce a risk-prioritized action list. Triggers on "true alert triage", "what's alerting and why", "prioritized alerts", "alerts with risk context", "triage with vulns", "smart triage".
---

# True Alert Triage (Alerts + CVE/Patch Context)

Start from N-central's live monitoring alerts (the only source of real alert state), then layer GraphQL vulnerability and patch-failure detail on top. Uses **both** the N-able MCP (GraphQL) and the N-central MCP (classic REST).

This is the triage the GraphQL-only `alert-triage` skill cannot do: GraphQL has **no alert type**. N-central owns "what is firing"; GraphQL owns "how dangerous the box is."

## ID-SPACE WARNING (read first)
The two MCPs use **different ID spaces**. A GraphQL organization/asset id is NOT an N-central `orgUnitId`/`customerId`/`deviceId`. **Never pass an id from one MCP to the other.** Correlate ONLY on stable human attributes:
- **Customer** — match by exact customer NAME.
- **Device** — match by device NAME and/or `hostname`, OS as tiebreaker.
If a device appears in only one system, flag it (unmanaged on one side, or named differently) and confirm the match before merging.

## What's available
- N-central: `list_active_issues` (live alerts; customer/site org units only — iterate a SO's customers), `get_device_status` (failing service tasks), `list_customers` / `report_org_hierarchy` (resolve names+ids).
- GraphQL: `vulnerabilityDetectionSearch`, `patchInstallationSearch`, `assetSearch` (agent status). `validate` before every `execute`.
- Read-only skill — no mutations, no confirmation gate needed.

## Step 1 — Resolve IDs in both systems by name
Get the N-central org tree, then the GraphQL org ids. Hold a name->id map per system; do not cross them.
```json
{ "tool": "report_org_hierarchy", "args": { "format": "json" } }
```
```graphql
query ResolveOrgs { organizationSearch(first: 50, where: { name: { contains: "" } }) { nodes { id name } } }
```
Run `validate` on the GraphQL query, then `execute`. Match customers by exact name across the two result sets.

## Step 2 — Pull live alerts from N-central (source of truth)
For EACH customer org unit (iterate; `list_active_issues` rejects service-org ids), list active monitoring issues.
```json
{ "tool": "list_active_issues", "args": { "orgUnitId": 101, "format": "json" } }
```
The response is a bare array; each row has top-level `deviceId` + `serviceName` and an `_extra` object with `deviceName` (use to correlate to GraphQL) and `transitionTime` (since). `_extra.deviceClassValue/Label` are always null (known bug) — derive class elsewhere. These rows are the **only** real alerts; everything below is enrichment.

## Step 3 — Drill the worst devices' failing services (N-central)
For each device named in an active issue, pull its service-monitoring state to name the failing tasks.
```json
{ "tool": "get_device_status", "args": { "deviceId": "55012" } }
```
The result is `{ data: [...], totalItems }`; each task has `moduleName` (service) and `stateStatus`. Record each task where `stateStatus` != `Normal` (e.g. `Failed`/`Warning`/`Stale`). This is live state, not historical.

## Step 4 — Enrich each alerting device with open CVEs (GraphQL)
Match the alerting device by name, then pull unresolved vulnerabilities with exploit/ransomware flags and risk score.
```graphql
query AlertCVEs($name: String!) {
  vulnerabilityDetectionSearch(
    where: { status: { in: [UNRESOLVED] }, asset: { name: { equals: $name } } }
    orderBy: [{ field: RISK_SCORE, direction: DESC }]
  ) {
    nodes {
      asset { name } customer { name }
      vulnerability { cveIdentifier severity riskScore hasExploit hasRansomwareCampaignUse }
      firstDetectedAt patchableStatus
    }
  }
}
```
`validate` then `execute`. An `hasExploit` or `hasRansomwareCampaignUse` CVE on an actively-alerting box is a top-priority flag.

## Step 5 — Enrich with patch failures (GraphQL)
For the same device name, find ERROR and AVAILABLE patches — failed remediation compounds the live alert.
```graphql
query AlertPatches($name: String!) {
  patchInstallationSearch(
    where: {
      asset: { name: { equals: $name } }
      or: [
        { installationStatus: { equals: ERROR } }
        { installationStatus: { equals: AVAILABLE } }
      ]
    }
    orderBy: [{ field: FAILURE_COUNT, direction: DESC }]
  ) {
    nodes {
      asset { name } customer { name }
      patch { name severity classification }
      status failureCount errorDetails { message occurredAt } lastUpdatedAt
    }
  }
}
```
`installationStatus` only supports `equals`/`notEquals` (no `in`) — OR two `equals` to span ERROR + AVAILABLE.
`validate` then `execute`. Optionally confirm agent reachability with `assetSearch` -> `agentConnection { status }` so you can tell "agent down" from "service down."

## Step 6 — Compute composite priority
Rank each alerting device by a composite of: live service-down (N-central) + open exploitable/ransomware CVE (GraphQL) + failed patches (GraphQL). The top tier is **service-down AND exploitable CVE AND failed patch** on the same box. Sort descending by that composite.

## Output format
**True Alert Triage — [Scope] — 2026-06-13**

Sorted by composite risk, highest first:

| Device | Customer | Live alert (N-central) | Open CVEs / risk (GraphQL) | Patch failures (GraphQL) | Recommended action |
|--------|----------|------------------------|----------------------------|--------------------------|--------------------|
| host-01 | Acme | Disk svc DOWN | CVE-2024-1234 (9.8, exploit, ransomware) | 2 ERROR (critical) | Isolate + emergency patch |

Tag every fact with its source MCP (N-central = live alert; GraphQL = CVE/patch). Note any device that appeared in only one system as **UNCONFIRMED MATCH**. Close with a one-line count: "N alerting devices triaged; M top-priority (live alert + exploitable CVE + failed patch). Combines N-central monitoring — which GraphQL has no alert type for — with GraphQL CVE/patch detail."
