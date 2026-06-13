---
description: Review script task executions across devices for failures and coverage gaps. Triggers on "automation policy review", "script task failures", "automation failures for [client]", "what automations are running", "script results".
---

# Automation Policy Review

Audit script task executions across N-central using the N-able MCP.

## Tools available

- `list_script_tasks_for_asset` — script execution history for a specific device (use assetId)
- `taskSearch` — search across all task executions with filtering and pagination
- `taskAggregations` — aggregate task counts by status

## 1. Fleet-wide task failure summary

```graphql
query TaskFailureSummary($customerId: ID) {
  taskAggregations(inOrganization: $customerId) {
    status {
      buckets(size: 8) { key count }
    }
  }
}
```

Omit `inOrganization` to run across all customers.

## 2. Recent failed task executions

```graphql
query RecentTaskFailures($customerId: ID) {
  taskSearch(
    first: 50
    where: { status: { equals: FAILED } }
    inOrganization: $customerId
    orderBy: [{ field: UPDATED_AT, direction: DESC }]
  ) {
    totalCount
    nodes {
      id
      name
      status
      failureReason
      retryCount
      createdAt
      latestExecution {
        startedAt
        durationMilliseconds
        errorMessage
        exitCode
        status
      }
    }
  }
}
```

## 3. Script history for a specific device

Use the dedicated tool:
- Tool: `list_script_tasks_for_asset`
- Required: `assetId`
- Optional: `first` (default 25), `where: { task: { type: { equals: "TASK_SUBTYPE_RUN_SCRIPT" } } }`

Returns: `id`, `status` (SUCCEEDED/FAILED/IN_PROGRESS), `errorMessage`, `exitCode`, `task.name`, `startedAt`, `durationMilliseconds`

Validate GraphQL queries before executing.

## Analysis flags

| Flag | Condition |
|---|---|
| High failure rate | >20% of recent runs failed |
| Long-running | Duration >10 minutes for a routine script |
| Repeated failures | Same task failing 3+ times |
| Recently failed | Failed within last 24h |

## Output format

Summary: total tasks reviewed, count succeeded / failed / in-progress.

Then list failed tasks sorted by customer:
- Task name | Customer | Device | Status | Exit code | Error message | Started at

Recommended actions:
- Exit code non-zero → check script logic and target device state
- Error message mentions permissions → verify script credentials
- Task stuck IN_PROGRESS → check if agent is still connected

Note: task execution is read-only in this skill. To trigger new script runs, use `assetsScriptRun` (write-mode, requires explicit permission grant).
