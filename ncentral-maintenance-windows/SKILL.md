---
description: Review and manage patch maintenance windows for devices and customers, flagging devices that have no window and safely creating or updating schedules. Triggers on "maintenance windows", "patch schedule for [device/client]", "set maintenance window", "when does [client] patch", "review patch windows".
---

# N-central Maintenance Windows

Review and manage device patch maintenance windows. Uses the **N-central MCP** (classic N-central REST API).

Maintenance windows control WHEN a device patches unattended. A device with NO window will not auto-patch in a controlled time slot — flag it. Create/update are WRITE operations: preview, confirm, then verify.

## What's available
- `list_devices_by_org_unit` — resolve all devices under a customer/site.
- `get_maintenance_windows {deviceId}` — read existing windows for one device.
- `create_maintenance_windows {deviceIDs:[number], maintenanceWindows:[obj]}` — WRITE; returns per-device partial-failure statuses, inspect them.
- `update_maintenance_windows {maintenanceWindows:[{scheduleId,...}]}` — WRITE; edits an existing window by `scheduleId`.

Note: `deviceIDs` in create/update are NUMERIC ids; `deviceId` from list/get is a string. Convert before writing.

## Step 1 — Resolve devices
Pull every device under the org unit (customer or site). Use `all:true` for a complete list.

```json
{ "orgUnitId": 1234, "all": true }
```

Capture each `deviceId` (string) and numeric device id for later writes.

## Step 2 — Review existing windows
For each device, read its current windows.

```json
{ "deviceId": "987654" }
```

Returns `{ data: [...], totalItems }`. Each window has (verified): `scheduleID`, `name`, `type`, `cron` (the recurrence, e.g. `0 0 0,16 * * ? *`), `duration` (minutes), `enabled`, `applicableAction` (e.g. Patch `detect`/`download`/`install`/`reboot`), `ruleName`, and reboot settings. Build a per-device row: window name, `cron`, `duration`, `enabled`, action. Devices that return NO windows go on the "missing" list — they won't auto-patch in a controlled slot.

> **Casing gotcha:** the GET returns `scheduleID` (capital ID), but `update_maintenance_windows` expects `scheduleId` (lowercase d). Map the value across when editing.

## Step 3 (WRITE) — Preview the create, then confirm
Before any `create_maintenance_windows` call, render a PREVIEW block to the user:
- The exact `maintenanceWindows` spec (name, type, `cron`, `duration`, `applicableAction`, enabled).
- The full list of target devices (name + numeric id).
- A one-line summary: "Creating N maintenance window(s) on M device(s)."

Then STOP and ask: **"Apply this maintenance window to the M device(s) above? (yes/no)"**

Do NOT call the write tool until the user explicitly replies yes. If anything is ambiguous (timezone, recurrence, target set), ask before proceeding.

## Step 4 (WRITE) — Create the windows
Only after explicit confirmation. Pass numeric `deviceIDs`.

```json
{
  "deviceIDs": [987654, 987655],
  "maintenanceWindows": [
    {
      "name": "Weekly Patch Window",
      "type": "Patch",
      "cron": "0 0 2 ? * SUN *",
      "duration": 120,
      "enabled": true,
      "applicableAction": "install",
      "ruleName": "Weekly Patch Window"
    }
  ]
}
```

The response returns a PER-DEVICE status. Inspect carefully — a partial failure means some devices succeeded and others did not. Report each device's result.

## Step 5 (WRITE) — Update an existing window
To edit a window in place, supply its `scheduleId` (from `get_maintenance_windows`). Same preview + confirm gate as Step 3 applies.

```json
{
  "maintenanceWindows": [
    {
      "scheduleId": 55501,
      "name": "Weekly Patch Window",
      "type": "Patch",
      "cron": "0 0 3 ? * SUN *",
      "duration": 90,
      "enabled": true,
      "applicableAction": "install",
      "ruleName": "Weekly Patch Window"
    }
  ]
}
```

## Step 6 (WRITE) — Verify
After any create/update, re-read each affected device to confirm the change landed.

```json
{ "deviceId": "987654" }
```

Diff the result against the intended spec. Report any device where the window is absent or does not match.

## Output format
**Maintenance Window Review — [Customer/Org] — [Date]**

| Device | Window Name | Cron | Duration | Action | Enabled |
|--------|-------------|------|----------|--------|---------|
| ... | ... | `0 0 2 ? * SUN *` | 120m | install | yes |

Sort by device name. Then:

**Devices missing a maintenance window**
- DEVICE-NAME (id 987660)
- ...

For WRITE operations, also include:

**Preview** — the exact `maintenanceWindows` spec + target device list, followed by the explicit confirmation gate.

**Write result** — per-device success/failure table from the create/update response:

| Device | Action | Result |
|--------|--------|--------|
| DEVICE-NAME | create | success |
| DEVICE-NAME | create | failed: <reason> |

**Verification** — post-write `get_maintenance_windows` confirmation per device (match / mismatch).

Close with a one-line count: "X devices reviewed, Y missing a window, Z windows written (W succeeded, V failed)."
