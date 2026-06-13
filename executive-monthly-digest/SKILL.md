---
description: Generate a monthly MSP operations digest across all customers for leadership. Triggers on "monthly digest", "executive summary for [month]", "MSP operations summary", "monthly report for leadership".
---

# Executive Monthly Digest

Produce a leadership-ready monthly operations summary across all managed customers using the N-able MCP.

## Queries

### Fleet overview

```graphql
query MonthlyDigestFleet($after: String) {
  assetSearch(first: 500, after: $after) {
    totalCount
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      customer { id name }
      serviceOrganization { name }
      operatingSystemInfo { name type }
      chassis { types }
      agentConnection { status }
      isManaged
      patchManagement { status }
    }
  }
}
```

### Patch aggregations (all customers)

```graphql
query MonthlyDigestPatches {
  patchInstallationAggregations {
    status { buckets(size: 7) { key count } }
  }
}
```

### Critical-unresolved vulnerabilities (per customer)

Run once per customer (use the `customer.id` values from the fleet query). Filtering on BOTH status and severity in one query gives a true critical-AND-unresolved count — do not derive it by summing independent severity and status buckets.

```graphql
query MonthlyDigestCriticalVulns($customerId: ID!) {
  vulnerabilityDetectionSearch(
    inOrganization: $customerId
    where: { status: { in: [UNRESOLVED] }, vulnerability: { severity: { in: [CRITICAL] } } }
  ) {
    totalCount
  }
}
```

Validate before executing. Paginate the fleet query when `totalCount > 500`: re-run with `after: <pageInfo.endCursor>` while `pageInfo.hasNextPage` is true, aggregating node-derived metrics across all pages.

## Metrics to compute from returned data

| Metric | Source |
|---|---|
| Total managed devices | `assetSearch.totalCount` |
| Fleet by OS type | Group `operatingSystemInfo.type` (WINDOWS/LINUX/DARWIN) |
| Agent connectivity rate | connected / total × 100 |
| Unmanaged devices | `isManaged = false` count |
| Patch compliance | COMPLETED / (COMPLETED + AVAILABLE + ERROR) × 100 |
| Patch errors | ERROR count from aggregations |
| Open critical vulns | Sum of `vulnerabilityDetectionSearch.totalCount` (critical + unresolved) across customers |
| Customers with critical vulns | Count of customers whose `vulnerabilityDetectionSearch.totalCount > 0` |

## Output format

**MSP Operations Digest — [Month Year]**
*For: Leadership / Executive Team*

---

**KPI Summary**

| Metric | Value | Notes |
|---|---|---|
| Total managed devices | X | |
| Agent connectivity | X% | X disconnected |
| Patch compliance | X% | X errors |
| Open critical vulns | X | X customers affected |
| Unmanaged devices | X | Coverage gap |

---

**Fleet Highlights** — 3–4 bullets on notable data points.

**Customers Requiring Attention** — Top 3–5 by risk signal count (link to renewal-risk-report for detail).

**Operational Risks** — 2–3 risks visible in the data leadership should be aware of.

**Growth Opportunities** — 1–2 upsell signals (link to upsell-opportunity-report for detail).

---

Keep to one page. This is leadership context — engineers have the individual skill reports.
