---
description: Audit scheduled and automation tasks and their run status across the N-central environment. Triggers on "scheduled tasks", "job status", "did [task] run", "failed jobs", "automation task status", "what tasks are scheduled".
---

# N-central Scheduled Task Monitor

Inventory scheduled tasks and report their last-run status, success/fail counts, and the devices that failed.

Uses the **N-central MCP** (classic N-central REST API). Read-only — no writes.

## What's available
- `list_scheduled_tasks` — every scheduled task in the environment. Supports `select` (FIQL row filter, e.g. `customerId==123`, joined with `;`) and `all` for full pagination.
- `get_scheduled_task` — one task's `parentId`, `name`, `type`, `customerId`, `deviceIds`, `enabled`.
- `get_scheduled_task_status` — aggregated last-run status. `detailed:true` gives a per-device pass/fail breakdown but ONLY for SYSTEM/CUSTOMER-level task ids — NOT device-level ids. Navigate to the parent via `parentId`.
- `list_device_tasks` — scheduled tasks on a single device (`taskId`, `taskName`, `status`).
- `get_appliance_task` — appliance-level task info.

Levels: a task is **device-level**, **customer-level**, or **system-level**. Detailed per-device status only works at customer/system level — always resolve the parent first.

## Step 1 — Pull the task inventory
Full environment:
```json
{ "all": true }
```
Scope to a customer (rerun if a name needs resolving first):
```json
{ "all": true, "select": "customerId==123" }
```
Only enabled tasks:
```json
{ "all": true, "select": "enabled==true" }
```
This gives you the task ids to enrich. **Note:** on some tenants the global `list_scheduled_tasks` returns only `{ "_links": {...} }` with no `data` array (no system/customer-level tasks exposed) — when that happens, drive the report per device instead. `list_device_tasks` with a `deviceId` returns `{ data: [ { taskId, taskName, status } ], totalItems }` (status values like `Completed`, `Scheduled (Last Job Completed)`). Enrich any task of interest via Step 2/3, then go to the Output format.

## Step 2 — Resolve each task's metadata and parent
For each task id from Step 1:
```json
{ "taskId": "98765" }
```
`get_scheduled_task` returns `{ data: { taskId, parentId, name, taskName, type, isEnabled, customerId, deviceIds:[...] } }` — the enabled flag is **`isEnabled`** (not `enabled`). Record `type` (e.g. `Script`) and the level. If the task id is device-level, hold its `parentId` for the detailed status call in Step 4.

## Step 3 — Aggregated status per task
```json
{ "taskId": "98765" }
```
`get_scheduled_task_status` returns `{ data: { taskName, statusCounts: { <Status>: count } } }` (e.g. `{ "Completed": 1 }`). Aggregated status works on any id, including device-level. Use this for the summary row of every task. Do NOT pass `detailed:true` here unless the id is already customer/system-level.

## Step 4 — Per-device drill-down on failures
For any task showing failures, call the SYSTEM/CUSTOMER-level id (the `parentId` you held from Step 2) with detail on:
```json
{ "taskId": "12345", "detailed": true }
```
This returns the per-device pass/fail breakdown. Collect the failed `deviceId`s and their error/status for the drill-down list.

For appliance tasks, use `get_appliance_task`:
```json
{ "taskId": "55501" }
```

## Notes
- Do not call `detailed:true` on a device-level id — it returns nothing useful. Always navigate via `parentId` to the customer/system parent first.
- Disabled tasks still appear in the inventory; flag them as `Enabled: No` rather than counting them as failures.
- If `get_scheduled_task` 404s for an id, skip it and note it as unresolved.

## Output format

**Scheduled Task Status — [Customer or "All"] — [Date]**

| Task | Type | Level | Enabled | Last status | Success/Fail |
|------|------|-------|---------|-------------|--------------|

- One row per task. Sort failing tasks first, then by `Last status`, then task name.
- `Success/Fail` is the count from the aggregated status (e.g. `42/3`); use `—` when no run history.

**Failed devices**
- One bullet per failed device, grouped under its task name:
  - `[Task name] → [device name / deviceId] — [error or status]`
- Omit this section entirely if nothing failed.

Close with a one-line count: **X failing tasks / Y total (Z disabled).**
