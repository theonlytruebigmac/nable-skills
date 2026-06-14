---
description: GraphQL-inferred triage of disconnected/vuln/patch-failure signals across N-central. Triggers on "triage alerts", "what's firing", "GraphQL alert triage", "disconnected/vuln/patch-failure triage", "morning standup", "what needs attention".
---

# Alert Triage

Pull and prioritize active issues across the N-central fleet using the N-able MCP.

## What's available

The N-able GraphQL API does not have a dedicated "alert" type. Active issues surface through four signals:
1. **Disconnected agents** — assets where `agentConnection.status = DISCONNECTED`
2. **Vulnerability detections** — via `vulnerabilityDetectionSearch` with `status: UNRESOLVED`
3. **Patch failures** — via `patchInstallationSearch` with `status: ERROR`
4. **Reboot required** — disconnected agents (from signal #1) where `reboot.isRequired = true`

## Steps

### 1. Disconnected agents

```graphql
query AlertTriageDisconnected($customer: ID) {
  assetSearch(
    first: 100
    where: { agentConnection: { status: { equals: DISCONNECTED } } }
    inOrganizations: [$customer]
  ) {
    totalCount
    nodes {
      id
      name
      customer { name }
      agentConnection { status statusChangedAt }
      operatingSystemInfo { name type }
      chassis { types }
      lastBootedAt
      reboot { isRequired }
      tags { nodes { name } }
    }
  }
}
```

Remove `inOrganizations` arg if running across all customers (omit the variable).

### 2. Patch failures

```graphql
query AlertTriagePatchErrors($customer: ID) {
  patchInstallationSearch(
    first: 50
    where: { installationStatus: { equals: ERROR } }
    inOrganizations: [$customer]
  ) {
    totalCount
    nodes {
      asset { id name }
      customer { name }
      patch { name severity classification }
      failureCount
      attemptCount
      errorDetails { message occurredAt }
      lastUpdatedAt
    }
  }
}
```

Remove `inOrganizations` arg if running across all customers (omit the variable).

### 3. Critical unresolved vulnerabilities

```graphql
query AlertTriageVulns($customer: ID) {
  vulnerabilityDetectionSearch(
    first: 50
    where: { status: { in: [UNRESOLVED] } }
    inOrganization: $customer
    orderBy: [{ field: RISK_SCORE, direction: DESC }]
  ) {
    totalCount
    nodes {
      asset { name }
      customer { name }
      vulnerability { cveIdentifier severity riskScore hasExploit hasRansomwareCampaignUse }
      firstDetectedAt
      patchableStatus
    }
  }
}
```

Remove `inOrganization` arg if running across all customers (omit the variable).

Validate each query with `validate` before executing with `execute`.

## Output format

**TRIAGE BRIEF — [Date/Time]**

**Disconnected Agents** (sorted by time offline, longest first)
- Device name | Customer | Last seen | OS

**Patch Failures** (sorted by failure count desc)
- Device | Customer | Patch | Failures | Last error

**Critical Vulnerabilities** (severity HIGH/CRITICAL or hasExploit=true)
- Device | Customer | CVE | Risk score | Patchable?

**Reboot Required** (disconnected agents from Step 1 where `reboot.isRequired = true`)
- Device | Customer | Last booted

End with a one-line count: X disconnected, X patch failures, X critical vulns.
