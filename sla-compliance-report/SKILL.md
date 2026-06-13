---
description: Report on patch and vulnerability SLA compliance across customers. Triggers on "SLA report", "SLA compliance", "are we meeting SLA", "response time report", "resolution time for [client]".
---

# SLA Compliance Report

Evaluate SLA performance using patch installation and vulnerability detection data from the N-able MCP.

## Note on scope

The N-able GraphQL API does not expose ticket response times (those live in your PSA). What it does expose — and what drives technical SLA conversations — is:

- **Patch installation latency** — `PatchInstallationCompletionDetails.lag` (time from detection to install)
- **Vulnerability detection age** — `firstDetectedAt` on unresolved detections
- **Patch error persistence** — how long patches have been in ERROR state

## Query 1 — Long-unresolved vulnerabilities

```graphql
query SLAVulnAge($customerId: ID!) {
  vulnerabilityDetectionSearch(
    first: 100
    where: {
      status: { in: [UNRESOLVED] }
    }
    inOrganization: $customerId
    orderBy: [{ field: RISK_SCORE, direction: DESC }]
  ) {
    totalCount
    nodes {
      asset { name }
      customer { name }
      vulnerability { cveIdentifier severity riskScore hasExploit isCisaKev }
      firstDetectedAt
      patchableStatus
      remediation { status patchUnavailableReason }
    }
  }
}
```

## Query 2 — Persistent patch errors

```graphql
query SLAPatchErrors($customerId: ID!) {
  patchInstallationSearch(
    first: 50
    where: { installationStatus: { equals: ERROR } }
    inOrganizations: [$customerId]
    orderBy: [{ field: LAST_UPDATED_AT, direction: ASC }]
  ) {
    totalCount
    nodes {
      asset { name }
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

Validate before executing. To run across all customers, omit `$customerId` and remove the `inOrganization` (Query 1) / `inOrganizations` (Query 2) argument.

## SLA breach thresholds (customize to your contracts)

| Severity | Patch / Remediate SLA |
|---|---|
| Critical CVE | 7 days from detection |
| Important CVE | 30 days from detection |
| Medium CVE | 90 days from detection |
| Critical patch | 14 days in ERROR state |
| Important patch | 30 days in ERROR state |

## Analysis

For each unresolved vulnerability: calculate days since `firstDetectedAt`. Flag as **SLA BREACH** if over threshold for its severity.

For each patch error: calculate days since `errorDetails.occurredAt`. Flag as **OVERDUE** if error has persisted beyond the patch SLA.

## Output format

**SLA Compliance Report — [Customer or All] — [Date]**

**Vulnerability SLA**
| Severity | Total Unresolved | Breaching SLA | % Compliant |
|---|---|---|---|

Breach list: CVE | Asset | Customer | Days open | Patchable?

**Patch Error SLA**
| Severity | Persistent Errors | Days in Error (avg) |
|---|---|---|

Error list: Patch | Asset | Customer | Days in error | Last error message

**Critical flags** — any CISA KEV or exploit-known vulns unresolved past 7 days.
