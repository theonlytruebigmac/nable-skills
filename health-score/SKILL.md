---
description: Compute a composite health score for a customer's managed environment. Triggers on "health score for [client]", "how healthy is [client]", "score [client]'s environment", "client health rating".
---

# Health Score

Compute a weighted composite health score (0–100) for a customer using the N-able MCP.

## Queries

```graphql
query HealthScoreAssets($customerId: ID!, $after: String) {
  assetSearch(first: 200, after: $after, inOrganizations: [$customerId]) {
    totalCount
    pageInfo { endCursor hasNextPage }
    nodes {
      id
      name
      agentConnection { status statusChangedAt }
      patchManagement { status }
      reboot { isRequired }
      operatingSystemInfo { name version featureRelease }
      isManaged
      tags { nodes { name } }
    }
  }
}
```

```graphql
query HealthScorePatches($customerId: ID!) {
  patchInstallationAggregations(inOrganization: $customerId) {
    status { buckets(size: 7) { key count } }
  }
}
```

```graphql
query HealthScoreVulns($customerId: ID!) {
  vulnerabilityDetectionAggregations(inOrganization: $customerId) {
    status { buckets(size: 5) { key count } }
    vulnerability {
      severity { buckets(size: 5) { key count } }
    }
  }
}
```

Validate each before executing. If `totalCount > 200`, paginate the assets query: re-run with `after: <pageInfo.endCursor>` and loop while `pageInfo.hasNextPage` is true, accumulating all `nodes` so the per-device dimensions (connectivity, EOL OS, managed coverage) are computed over the whole fleet rather than the first 200 assets.

## Scoring model

Score each dimension 0–100, then apply weights:

| Dimension | Weight | How to score |
|---|---|---|
| Agent connectivity | 25% | (connected devices / total) × 100 |
| Patch compliance | 25% | (COMPLETED / (COMPLETED + AVAILABLE + ERROR)) × 100 |
| Vulnerability posture | 20% | Start 100, subtract: CRITICAL=15pts, IMPORTANT=8pts, MODERATE=3pts (floor 0) |
| EOL OS exposure | 15% | (devices on supported OS / total) × 100 |
| Managed coverage | 15% | (isManaged=true / total) × 100 |

For **EOL OS exposure**, classify each device's `operatingSystemInfo.name`/`version` against known end-of-life versions (e.g. Windows 10, Server 2012/2012 R2, Server 2008, macOS older than 3 major versions; see the `eol-os-report` skill's EOL reference table for the full list). A device counts as "supported OS" if its OS is still vendor-patched.

**Overall = sum of (dimension score × weight)**

If a dimension has no data, redistribute its weight proportionally to scored dimensions and note the gap.

## Rating scale

| Score | Grade | Color |
|---|---|---|
| 90–100 | A | Green |
| 80–89 | B | Light green |
| 70–79 | C | Yellow |
| 60–69 | D | Orange |
| <60 | F | Red |

## Output format

**Health Score: [score]/100 — Grade [X]**

Score breakdown table (dimension | raw score | weight | contribution).

**Top 3 factors dragging the score down** — one sentence each explaining what's driving it.

**One-sentence summary** for an account manager to share with a customer contact.

**How to get to an A:** 2–3 specific actions with the highest score impact.
