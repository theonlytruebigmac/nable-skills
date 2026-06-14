---
description: Find service coverage gaps across the fleet that map to MSP upsell opportunities. Triggers on "upsell opportunities", "expansion opportunities", "coverage gaps", "where can we grow", "what services are customers missing".
---

# Upsell Opportunity Report

Scan N-central data for coverage gaps that map to MSP service offerings using the N-able MCP.

## Step 1 — Fleet and coverage data

```graphql
query UpsellFleet {
  assetSearch(
    first: 500
    orderBy: [{ field: NAME, direction: ASC }]
  ) {
    totalCount
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      name
      customer { id name }
      serviceOrganization { name }
      operatingSystemInfo { name version type featureRelease }
      chassis { types }
      agentConnection { status }
      isManaged
      patchManagement { status }
      vulnerabilityManagement { status }
      tags { nodes { name } }
    }
  }
}
```

If `totalCount > 500`, paginate: re-run with `after: <pageInfo.endCursor>` while `pageInfo.hasNextPage` is true, accumulating nodes before running opportunity detection.

## Step 2 — Vulnerability exposure by customer

```graphql
query UpsellVulnExposure($customerId: ID!) {
  vulnerabilityDetectionAggregations(
    inOrganization: $customerId
    where: { status: { in: [UNRESOLVED] } }
  ) {
    status { buckets(size: 5) { key count } }
    vulnerability {
      severity { buckets(size: 5) { key count } }
    }
  }
}
```

Validate before executing. Collect the distinct `customer { id }` values from Step 1's assetSearch nodes and run this query once per customer id, then aggregate the per-customer results into the report.

## Opportunity detection

Group assets by customer, then check:

| Opportunity | Signal | Service to pitch |
|---|---|---|
| Vulnerability management gap | `vulnerabilityManagement.status` not ACTIVE or null on any device | VM / Security scanning add-on |
| Patch management gap | `patchManagement.status` not ACTIVE or null on any device | Managed patching tier upgrade |
| Unmanaged devices | `isManaged = false` devices present | Full managed services expansion |
| EOL OS exposure | Devices on known end-of-life OS (e.g. Windows 10, Server 2012/2016) | OS upgrade project / vCIO advisory |
| High vuln count | CRITICAL or IMPORTANT unresolved detections >5 | MDR / security stack |
| No tags = no organization | `tags.nodes` empty on most devices | Structured onboarding / assessment |

For EOL OS exposure, classify each device from `operatingSystemInfo.name`/`version` against known end-of-life OS versions (cross-reference the `eol-os-report` skill) rather than relying on a lifecycle field the query does not return.

## Output format

**Upsell Opportunity Report — [Date]**

Summary table:
| Opportunity | Customers Affected | Devices Affected |
|---|---|---|

Then one section per opportunity type:
- Customer name | Service Org | Devices affected | Count
- **Talking point** (1 sentence for account manager)

Example: "You have 6 devices with vulnerability management disabled — a single unpatched critical CVE could mean ransomware exposure across those endpoints."

Sort customers by device count affected descending (biggest opportunity first).

Close with total estimated expansion pipeline for the sales team.
