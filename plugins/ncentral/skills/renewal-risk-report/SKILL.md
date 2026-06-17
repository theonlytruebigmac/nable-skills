---
description: Identify customers showing environment signals that correlate with dissatisfaction or churn risk. Triggers on "renewal risk", "at-risk customers", "churn risk", "which clients need attention", "account health".
---

# Renewal Risk Report

Surface customers with deteriorating environment signals using the N-able MCP.

## Step 1 — Full fleet snapshot

```graphql
query RenewalRiskFleet {
  assetSearch(
    first: 500
    orderBy: [{ field: NAME, direction: ASC }]
  ) {
    totalCount
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      name
      customer { id name parentOrganization { name } }
      agentConnection { status statusChangedAt }
      patchManagement { status }
      operatingSystemInfo { name }
      isManaged
      reboot { isRequired }
      tags { nodes { name } }
    }
  }
}
```

Validate before executing. If `totalCount > 500`, re-run with `after: <endCursor>` while `hasNextPage` is true, accumulating `nodes` — the percentage signals below must cover the full fleet, not just the first 500.

## Step 2 — Unresolved CRITICAL vulnerability count per customer

```graphql
query RenewalRiskCriticalVulns($customerId: ID!) {
  vulnerabilityDetectionSearch(
    inOrganization: $customerId
    where: {
      vulnerability: { severity: { in: [CRITICAL] } }
      status: { in: [UNRESOLVED] }
    }
  ) {
    totalCount
  }
}
```

Run this per customer found in Step 1. `totalCount` is the number of detections
that are both CRITICAL and UNRESOLVED — the count the scoring signal needs.

## Step 3 — Patch error counts per customer

```graphql
query RenewalRiskPatchErrors($customerId: ID!) {
  patchInstallationAggregations(inOrganization: $customerId) {
    status { buckets(size: 7) { key count } }
  }
}
```

Validate before executing.

## Risk signal scoring

For each customer, accumulate risk points:

| Signal | Points |
|---|---|
| >25% of devices agent DISCONNECTED | +3 |
| Any unresolved CRITICAL vulnerability | +2 each (cap at +6) |
| Patch ERROR count >5 | +2 |
| Patch management INACTIVE on any device | +1 |
| 10–25% devices disconnected | +1 |
| Any device on EOL OS | +1 |
| Unmanaged devices present | +1 |

**Risk tier:** 6+ = HIGH | 3–5 = MEDIUM | 0–2 = LOW

## Output format

**Renewal Risk Report — [Date]**

Summary: X customers HIGH, X MEDIUM, X LOW.

List HIGH and MEDIUM customers sorted by score:
- Customer name | Service Org | Risk tier | Score
- Bullet list of signals that contributed
- One-paragraph account summary for the account manager
- Recommended action: Immediate outreach / Schedule check-in / Monitor

LOW customers: summary table only (name, score).
