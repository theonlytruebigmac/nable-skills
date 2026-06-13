---
description: Assess a prospect or new customer's environment for scoping and pre-sales purposes. Triggers on "prospect assessment", "pre-sales audit", "scope [prospect]", "what would we be taking on with [prospect]", "new customer baseline".
---

# New Prospect Baseline

Analyze a prospect's N-central environment to scope an engagement accurately before contract signing.

## Prerequisite

The prospect must have N-central agents deployed (trial, POC, or temporary access). Find their customer ID via:

```graphql
query FindProspect($name: String!) {
  organizationSearch(
    first: 5
    where: { name: { contains: $name } }
  ) {
    nodes {
      id
      name
      ... on Customer {
        parentOrganization { name }
      }
    }
  }
}
```

## Fleet assessment query

```graphql
query ProspectBaseline($customerId: ID!) {
  assetSearch(
    first: 200
    inOrganizations: [$customerId]
  ) {
    totalCount
    nodes {
      id
      name
      operatingSystemInfo { name version type architecture featureRelease buildNumber }
      chassis { types }
      agentConnection { status statusChangedAt }
      lastBootedAt
      isManaged
      patchManagement { status lastPatchScanTime }
      vulnerabilityManagement { status }
      tags { nodes { name } }
      systemInfo { hostname manufacturer model serialNumber }
      reboot { isRequired }
    }
  }
}
```

## Patch and vulnerability baseline

```graphql
query ProspectPatchBaseline($customerId: ID!) {
  patchInstallationAggregations(inOrganization: $customerId) {
    status { buckets(size: 7) { key count } }
  }
}
```

```graphql
query ProspectVulnBaseline($customerId: ID!) {
  vulnerabilityDetectionAggregations(inOrganization: $customerId, where: { status: { in: [UNRESOLVED] } }) {
    vulnerability {
      severity { buckets(size: 5) { key count } }
    }
    status { buckets(size: 5) { key count } }
  }
}
```

Validate all queries before executing.

## Output format

**Prospect Environment Assessment — [Company] — [Date]**

**Fleet Summary**
- Total discovered devices: X (servers: X, workstations: X, other: X)
- OS distribution table (OS name → count)
- % on EOL or approaching-EOL OS

**Agent & Connectivity**
- Connected: X | Disconnected: X
- Unmanaged: X (flag as scope risk — unknown state)

**Patch Posture**
- Compliance state breakdown (COMPLETED / AVAILABLE / ERROR / DECLINED)
- Estimated effort: Low (<20% non-compliant) / Medium (20–50%) / High (>50%)

**Vulnerability Exposure**
- Unresolved by severity: Critical X | Important X | Moderate X | Low X

**Technical Debt Flags**
- EOL servers: [name + OS] — HIGH RISK
- Large disconnected device count — UNKNOWN SCOPE
- Patch management inactive on any device

**Onboarding Effort Estimate**

| Factor | Assessment |
|---|---|
| Fleet size | Small (<50) / Medium (50–200) / Large (200+) |
| OS currency | Good / Fair / Poor |
| Patch backlog | Low / Medium / High |
| Vuln exposure | Low / Medium / High |
| **Overall effort** | **Low / Medium / High** |

**Recommended Service Tier** — with 1-sentence rationale.

**SOW Risk Flags** — anything requiring a caveat or exclusion in the contract scope.
