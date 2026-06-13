---
description: Report on fleet size and composition across customers. Triggers on "fleet growth", "device count trends", "which customers are growing", "fleet size report", "device inventory".
---

# Fleet Growth Report

Analyze fleet composition across all customers using the N-able MCP.

## Query

```graphql
query FleetGrowthReport($after: String) {
  assetSearch(
    first: 500
    after: $after
    orderBy: [{ field: NAME, direction: ASC }]
  ) {
    totalCount
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      name
      customer { id name }
      serviceOrganization { id name }
      operatingSystemInfo { name type }
      chassis { types }
      agentConnection { status }
      isManaged
      tags { nodes { name } }
      lastBootedAt
    }
  }
}
```

Also get asset aggregations:

```graphql
query FleetAggregations {
  assetAggregations {
    agentConnection {
      status {
        buckets(size: 3) { key count }
      }
    }
  }
}
```

Validate before executing. If `totalCount > 500`, paginate: pass `pageInfo.endCursor` as the `$after` argument and re-query while `pageInfo.hasNextPage` is true, aggregating nodes across all pages before computing the breakdowns below.

## Aggregations to compute

From the returned data, group and count:

- **By customer** — total devices, servers vs. workstations vs. other
- **By OS type** — WINDOWS / LINUX / DARWIN
- **By agent status** — CONNECTED vs. DISCONNECTED
- **By managed status** — `isManaged = true/false`

Note: The GraphQL schema is a current-state system. Month-over-month delta requires saving a baseline snapshot from a prior run and comparing. If no prior baseline exists, report current state and recommend saving this output for future comparison.

## Output format

**Fleet Report — [Date]**

**Estate Overview**
- Total managed devices: X (nodes where `isManaged = true`)
- By OS: Windows X | Linux X | macOS X
- Agent connectivity: X connected (X%) | X disconnected
- Unmanaged devices: X across X customers

**Per-Customer Fleet Table**
| Customer | Service Org | Servers | Workstations | Other | Total | Connected |
|---|---|---|---|---|---|---|
Sort by total descending.

**Notable signals for sales team**
- Largest customers by device count: prioritize for QBR investment
- Customers with high unmanaged device count: coverage gap / upsell signal
- Customers with >25% disconnected: potential churn risk or network issue

**Growth signals** (if comparing to prior snapshot)
- Customers that added devices: [list with delta]
- Customers with shrinking fleet: [list — potential churn signal]
