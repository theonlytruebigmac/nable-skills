---
description: Generate a structured incident report (report only, no PSA ticket) from N-able GraphQL data. Triggers on "incident summary for [client]", "write up the outage", "incident report (no ticket)", "what happened with [client] on [date]".
---

# Incident Summary

Reconstruct an incident timeline from the N-able MCP (GraphQL).

## Steps

Ask for the customer name and time window if not provided (e.g., "June 9 between 2pm–6pm"). Capture both endpoints of the window as `$since` (start) and `$until` (end) so queries can be bounded on both sides.

### 1. Find affected devices and their current state

```graphql
query IncidentDeviceState($customerId: ID!) {
  assetSearch(
    first: 100
    inOrganizations: [$customerId]
  ) {
    nodes {
      id
      name
      agentConnection { status statusChangedAt }
      lastBootedAt
      reboot { isRequired }
      patchManagement { status lastPatchScanTime }
      operatingSystemInfo { name type }
      chassis { types }
      systemInfo { hostname }
      tags { nodes { name } }
    }
  }
}
```

### 2. Activity history on affected devices

For each device of interest, query activity log:

```graphql
query AssetActivityLog($assetId: ID!, $since: DateTime!, $until: DateTime!) {
  asset(id: $assetId) {
    name
    activitySearch(
      first: 50
      where: { occurredAt: { gte: $since, lte: $until } }
      orderBy: [{ field: OCCURRED_AT, direction: ASC }]
    ) {
      nodes {
        type
        activity
        occurredAt
        additionalData
      }
    }
  }
}
```

### 3. Script task executions during the window

```graphql
query IncidentTaskHistory($assetId: ID!, $since: DateTime!, $until: DateTime!) {
  asset(id: $assetId) {
    taskExecutionSearch(
      first: 25
      where: { startedAt: { gte: $since, lte: $until } }
      orderBy: [{ field: STARTED_AT, direction: ASC }]
    ) {
      nodes {
        task { name }
        status
        startedAt
        durationMilliseconds
        errorMessage
        exitCode
      }
    }
  }
}
```

### 4. Patch installation failures in the window

```graphql
query IncidentPatchFailures($customerId: ID!, $since: DateTime!, $until: DateTime!) {
  patchInstallationSearch(
    first: 25
    where: {
      installationStatus: { equals: ERROR }
      errorDetails: { occurredAt: { gte: $since, lte: $until } }
    }
    inOrganizations: [$customerId]
  ) {
    nodes {
      asset { name }
      patch { name severity }
      errorDetails { message occurredAt }
      failureCount
    }
  }
}
```

Validate all queries before executing.

## Output format

**Incident Report — [Customer] — [Date]**

**Summary** — One paragraph: what happened, which systems, when it started and ended.

**Timeline** — Chronological events with timestamps from activity logs, task executions, and agent status changes.

**Affected Systems** — Table: device name, role, impact, resolution status.

**Root Cause** — Best inference from available data. Note if causation cannot be determined from the available data alone.

**Impact** — Duration, devices affected.

**Resolution** — What resolved it, timestamp.

**Next Steps** — 2–3 action items.
