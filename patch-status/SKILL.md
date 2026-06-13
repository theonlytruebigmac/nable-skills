---
description: Report patch compliance and installation status for a customer or the full fleet. Triggers on "patch status", "patch compliance", "missing patches", "who needs patches", "patch report for [client]".
---

# Patch Status

Query N-central patch installation data using the N-able MCP GraphQL API.

## Key types

- `patchInstallationSearch` — per-asset records; statuses: `AVAILABLE`, `APPROVED`, `COMPLETED`, `OUTDATED`, `ERROR`, `SCHEDULED`, `DECLINED`
- `patchInstallationAggregations` — aggregate counts by status using `buckets`
- `AssetPatchManagement` — per-asset: `lastPatchScanTime`, `status` (`ACTIVE`/`INACTIVE`/`PENDING`/`UNSPECIFIED`)

## Severity values: `CRITICAL`, `IMPORTANT`, `MODERATE`, `LOW`, `UNKNOWN`

## Step 1 — Patch compliance summary for a customer

```graphql
query PatchStatusSummary($customerId: ID!) {
  patchInstallationAggregations(inOrganization: $customerId) {
    status {
      buckets(size: 7) {
        key
        count
      }
    }
    patch {
      severity {
        buckets(size: 5) {
          key
          count
        }
      }
    }
  }
}
```

## Step 2 — Devices with ERROR installations

```graphql
query PatchErrors($customerId: ID!) {
  patchInstallationSearch(
    first: 100
    where: { installationStatus: { equals: ERROR } }
    inOrganizations: [$customerId]
    orderBy: [{ field: FAILURE_COUNT, direction: DESC }]
  ) {
    totalCount
    nodes {
      asset { id name agentConnection { status } }
      customer { name }
      patch { name severity classification isRebootRequired publishedOn }
      failureCount
      attemptCount
      errorDetails { message occurredAt }
      lastUpdatedAt
    }
  }
}
```

## Step 3 — Devices with AVAILABLE (unpatched) critical patches

```graphql
query UnpatchedCritical($customerId: ID!) {
  patchInstallationSearch(
    first: 100
    where: {
      and: [
        { installationStatus: { equals: AVAILABLE } }
        { patch: { severity: { equals: CRITICAL } } }
      ]
    }
    inOrganizations: [$customerId]
  ) {
    totalCount
    nodes {
      asset { id name }
      customer { name }
      patch { name severity classification publishedOn }
      lastUpdatedAt
    }
  }
}
```

## Step 4 — Devices with inactive patch management

```graphql
query PatchMgmtInactive($customerId: ID!) {
  assetSearch(
    first: 100
    where: { patchManagement: { status: { notEquals: ACTIVE } } }
    inOrganizations: [$customerId]
  ) {
    nodes {
      id
      name
      customer { name }
      patchManagement { status lastPatchScanTime }
      agentConnection { status }
    }
  }
}
```

Validate each query before executing.

## Output format

**Patch Status Report — [Customer] — [Date]**

Summary (from aggregations):
| Status | Count |
|---|---|
| COMPLETED | X |
| AVAILABLE | X |
| ERROR | X |
| APPROVED | X |
| DECLINED | X |

Severity breakdown: CRITICAL X | IMPORTANT X | MODERATE X | LOW X

List ERROR devices sorted by failure count descending:
- Device | Customer | Patch | Severity | Failures | Last error message

List devices with unpatched CRITICAL patches.

Flag devices where `patchManagement.status != ACTIVE` — they cannot receive patches.
Flag disconnected agents — they cannot receive patches until reconnected.
