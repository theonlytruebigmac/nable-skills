---
description: Generate a structured incident report from N-central data. Triggers on "incident summary for [client]", "write up the outage", "incident report", "what happened with [client] on [date]".
---

# Incident Summary

Reconstruct an incident timeline from N-central data using the N-able MCP.

## Steps

Ask for the customer name and time window if not provided (e.g., "June 9 between 2pm–6pm").

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
query AssetActivityLog($assetId: ID!, $since: DateTime!) {
  asset(id: $assetId) {
    name
    activitySearch(
      first: 50
      where: { occurredAt: { gte: $since } }
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
query IncidentTaskHistory($assetId: ID!, $since: DateTime!) {
  asset(id: $assetId) {
    taskExecutionSearch(
      first: 25
      where: { startedAt: { gte: $since } }
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
query IncidentPatchFailures($customerId: ID!) {
  patchInstallationSearch(
    first: 25
    where: { installationStatus: { equals: ERROR } }
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

**Root Cause** — Best inference from available data. Note if causation cannot be determined from N-central data alone.

**Impact** — Duration, devices affected.

**Resolution** — What resolved it, timestamp.

**Next Steps** — 2–3 action items.
