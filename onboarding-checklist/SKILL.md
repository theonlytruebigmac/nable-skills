---
description: Check a new customer's onboarding completeness against MSP baseline. Triggers on "onboard [client]", "new customer checklist", "onboarding gaps for [client]", "set up monitoring for [client]".
---

# Onboarding Checklist

Audit a new customer's environment against an MSP onboarding baseline using the N-able MCP.

## Find the customer ID first

```graphql
query FindCustomer($name: String!) {
  organizationSearch(
    first: 10
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

## Pull all devices for the customer

```graphql
query OnboardingAudit($customerId: ID!, $after: String) {
  assetSearch(
    first: 200
    after: $after
    inOrganizations: [$customerId]
  ) {
    totalCount
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      name
      agentConnection { status statusChangedAt }
      isManaged
      operatingSystemInfo { name type architecture role }
      chassis { types }
      systemInfo { hostname manufacturer model }
      patchManagement { status lastPatchScanTime isVolumeOwnershipRequired }
      vulnerabilityManagement { status }
      tags { nodes { name } }
      reboot { isRequired }
      lastBootedAt
    }
  }
}
```

Validate before executing. The `first: 200` cap covers most customers in one page. If `totalCount` exceeds 200, confirm scope or page through with `after: $endCursor` (re-validating each time) while `pageInfo.hasNextPage` is true so the counts reflect the full fleet.

## Baseline checks per device

| Check | Field to inspect | Pass condition |
|---|---|---|
| Agent installed and online | `agentConnection.status` | `CONNECTED` |
| Device is managed | `isManaged` | `true` |
| Device tagged | `tags.nodes` | At least one tag present |
| Patch management active | `patchManagement.status` | `ACTIVE` |
| Patch scan recent | `patchManagement.lastPatchScanTime` | Within last 7 days |
| Vulnerability management enabled | `vulnerabilityManagement.status` | Not `DISABLED` |
| OS identified | `operatingSystemInfo.name` | Not null |
| No reboot pending | `reboot.isRequired` | `false` |

## Output format

Start with: X devices found, X fully onboarded, X with gaps.

List each failing check grouped by gap type for batch remediation:

**Missing tags:** [device list]
**Patch management inactive:** [device list]
**Agent disconnected:** [device list]
**Unmanaged devices:** [device list]

End with a prioritized remediation list:
1. Connect disconnected agents (blind spots)
2. Activate patch management (compliance risk)
3. Apply tags (organization and reporting)
4. Confirm vulnerability management enabled
