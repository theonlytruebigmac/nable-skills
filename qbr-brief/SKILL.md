---
description: Generate a Quarterly Business Review data brief for a customer. Triggers on "QBR for [client]", "quarterly business review [client]", "QBR data for [client]", "prepare QBR for [client]".
---

# QBR Brief

Pull a structured operational summary for a customer's Quarterly Business Review using the N-able MCP.

## Step 1 — Fleet overview

```graphql
query QBRFleet($customerId: ID!) {
  assetSearch(
    first: 200
    inOrganizations: [$customerId]
  ) {
    totalCount
    nodes {
      id
      name
      operatingSystemInfo { name version type architecture featureRelease }
      chassis { types }
      agentConnection { status statusChangedAt }
      lastBootedAt
      patchManagement { status lastPatchScanTime }
      vulnerabilityManagement { status lastUpdatedAt }
      reboot { isRequired }
      tags { nodes { name } }
      isManaged
      systemInfo { hostname manufacturer model }
    }
  }
}
```

## Step 2 — Patch installation status summary

```graphql
query QBRPatchSummary($customerId: ID!) {
  patchInstallationAggregations(inOrganization: $customerId) {
    status {
      buckets(size: 7) { key count }
    }
    patch {
      severity {
        buckets(size: 5) { key count }
      }
    }
  }
}
```

## Step 3 — Vulnerability summary

```graphql
query QBRVulnSummary($customerId: ID!) {
  vulnerabilityDetectionAggregations(inOrganization: $customerId) {
    status {
      buckets(size: 5) { key count }
    }
    vulnerability {
      severity {
        buckets(size: 5) { key count }
      }
    }
  }
}
```

Validate all queries before executing.

## Output format — slide-ready sections

**[Customer] QBR — [Quarter Year]**

---

**Fleet Overview**
- Total managed devices: X (servers: X, workstations: X, other: X)
- Agent connectivity: X connected (X%), X disconnected
- OS distribution: [top OS names and counts]
- Unmanaged devices: X

**Patch Posture**
- Completed: X | Available (unpatched): X | Error: X | Declined: X
- Devices with patch management inactive: X
- Key recommendation if patch compliance is low

**Vulnerability Posture**
- Unresolved detections: X (Critical: X, Important: X, Moderate: X, Low: X)
- Resolved this quarter: X

**Environment Risks**
- Devices on EOL OS: [list]
- Devices with reboot pending: X
- Longest-offline device: [name, days offline]

**Recommendations for Next Quarter**
2–3 specific, data-driven action items from what the data shows.

---

Note: QBR data reflects current state. For trend comparisons (month-over-month), save this report quarterly to build a baseline over time.
