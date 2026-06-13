---
description: Save a point-in-time snapshot of the full fleet to a local file for trend comparison. Triggers on "save snapshot", "baseline snapshot", "record fleet state", "save current state", "snapshot [client]".
---

# Fleet Snapshot

Save the current state of the N-central fleet to a local timestamped file so future runs of `/health-score`, `/fleet-growth-report`, and `/renewal-risk-report` can show trends over time.

## Step 1 — Fleet overview

```graphql
query SnapshotFleet {
  assetSearch(first: 500) {
    totalCount
    nodes {
      id
      customer { id name }
      serviceOrganization { name }
      operatingSystemInfo { name type }
      chassis { types }
      agentConnection { status }
      isManaged
      patchManagement { status }
      vulnerabilityManagement { status }
    }
  }
}
```

If `totalCount > 500`, paginate using `after` cursor until all assets are collected.

## Step 2 — Patch aggregations (full fleet)

```graphql
query SnapshotPatchAggs {
  patchInstallationAggregations {
    status { buckets(size: 7) { key count } }
  }
}
```

## Step 3 — Vulnerability aggregations (full fleet)

```graphql
query SnapshotVulnAggs {
  status: vulnerabilityDetectionAggregations {
    status { buckets(size: 5) { key count } }
  }
  unresolved: vulnerabilityDetectionAggregations(where: { status: { in: [UNRESOLVED] } }) {
    vulnerability {
      severity { buckets(size: 5) { key count } }
    }
  }
}
```

The `status` alias returns the overall resolved/unresolved counts; the `unresolved` alias scopes the severity breakdown to unresolved detections so the "Vulnerability Severity (unresolved)" table is accurate.

Validate each query before executing.

## Step 4 — Write snapshot to disk

After executing all three queries, compute per-customer summaries from the fleet nodes, then write the snapshot to:

```
~/Documents/nable-snapshots/YYYY-MM-DD.md
```

Create the directory if it does not exist.

## Snapshot file format

```markdown
# N-able Fleet Snapshot — YYYY-MM-DD

## Fleet Totals
- Total assets: X
- Connected: X (X%)
- Disconnected: X
- Managed: X | Unmanaged: X
- Patch mgmt active: X | Inactive/Pending: X
- Vuln mgmt active: X

## Patch Status (fleet-wide)
| Status | Count |
|---|---|
| COMPLETED | X |
| AVAILABLE | X |
| APPROVED | X |
| ERROR | X |
| DECLINED | X |
| SCHEDULED | X |
| OUTDATED | X |

## Vulnerability Status
| Status | Count |
|---|---|
| UNRESOLVED | X |
| RESOLVED | X |

## Vulnerability Severity (unresolved)
| Severity | Count |
|---|---|
| CRITICAL | X |
| IMPORTANT | X |
| MODERATE | X |
| LOW | X |

## Per-Customer Summary
| Customer | Total | Connected | Disconnected | Patch Active | Managed |
|---|---|---|---|---|---|
| Acme Corp | X | X | X | X | X |
...
```

## Comparing snapshots

To show trends, read a prior snapshot file and compare against the new one:
- Delta in total assets (fleet growth)
- Delta in AVAILABLE patch count (patch compliance trend)
- Delta in CRITICAL unresolved vulns
- Customers with worsening disconnected device count

Previous snapshots live in `~/Documents/nable-snapshots/`. List the files and read the most recent one before generating the new snapshot if a trend comparison is requested.
