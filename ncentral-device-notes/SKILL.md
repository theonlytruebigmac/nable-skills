---
description: Read and write device runbook/handoff notes in N-central, single or bulk, with preview-and-confirm on every write. Triggers on "device notes", "add note to [device]", "runbook note", "note all [client] servers", "read notes for [device]", "annotate devices".
---

# N-central Device Notes Manager

Read and write device notes (runbook/handoff annotations) on N-central devices. Writes always preview and confirm first.

Uses the **N-central MCP** (classic N-central REST API).

## What's available
- `list_device_notes {deviceId}` — read all notes on one device (timestamp, author, text).
- `add_device_note {deviceId, text}` — add one note; it attaches to the authenticated user as author.
- `update_device_note {deviceId, noteId, text}` — edit an existing note by its `noteId`. Both `deviceId` and `noteId` are STRINGS.
- `add_notes_bulk {deviceIDs:[number], text}` — add the SAME note to many devices in one call. `deviceIDs` are numeric.
- `list_devices_by_org_unit {orgUnitId, all?}` / `list_devices {all?}` — resolve the device set for a customer/site before bulk writing.

Notes:
- `deviceId` is a STRING for read/single ops; `deviceIDs` for bulk is an array of NUMBERS.
- There is no delete-note tool. To retract a note, `update_device_note` it to a tombstone (e.g. "[voided <date>]").
- Resolve org unit / customer / device IDs first with `report_org_hierarchy` or `list_devices` if the user gives names.

## Step 1 — Read notes for a device
```json
{ "deviceId": "123456" }
```
Call `list_device_notes` with the resolved `deviceId`. Render every note with its timestamp, author, and full text, newest first.

## Step 2 (write) — Add a single note: preview + confirm
Before calling `add_device_note`, show the user the target and exact text and wait for explicit confirmation.

Preview to display:
```
ADD NOTE — preview
Device:  WEB-PROD-01 (deviceId 123456)
Author:  <authenticated user>
Text:    "Rebooted after patch window 2026-06-13; verify IIS app pool on next check."
```
Ask: "Add this note? (yes/no)". Only on explicit `yes`, send:
```json
{ "deviceId": "123456", "text": "Rebooted after patch window 2026-06-13; verify IIS app pool on next check." }
```

## Step 3 (write) — Update an existing note: preview + confirm
First `list_device_notes` to get the `noteId`. Show the old vs. new text, then confirm.

Preview to display:
```
UPDATE NOTE — preview
Device:  WEB-PROD-01 (deviceId 123456)  noteId "9981"
Old:     "Temp DNS fix in place."
New:     "DNS permanently corrected 2026-06-13; temp fix removed."
```
On explicit `yes`, send:
```json
{ "deviceId": "123456", "noteId": "9981", "text": "DNS permanently corrected 2026-06-13; temp fix removed." }
```

## Step 4 (write) — Bulk-note all devices for a client: resolve, preview, confirm
1. Resolve the device list — pull every device under the org unit:
```json
{ "orgUnitId": 50, "all": true }
```
   Call `list_devices_by_org_unit`. Collect numeric device ids and longNames.
2. Preview the FULL list and the exact shared text, then confirm. Do NOT send if the list looks wrong or oversized — re-scope first.

Preview to display:
```
BULK ADD NOTE — preview
Org unit:  Contoso (orgUnitId 50)
Devices:   12
  - WEB-PROD-01 (101)
  - DB-PROD-01  (102)
  - ... (full list, no truncation)
Author:    <authenticated user>
Text:      "Q2 handoff: primary contact now Jane Doe, after-hours via on-call line."
```
Ask: "Send this note to all 12 devices? (yes/no)". On explicit `yes`, send:
```json
{ "deviceIDs": [101, 102, 103], "text": "Q2 handoff: primary contact now Jane Doe, after-hours via on-call line." }
```
`add_notes_bulk` may return partial failures — inspect the per-device result and report any that did not take.

## Step 5 (write) — Verify
After any write, re-read to confirm it landed:
- Single/update: `list_device_notes {deviceId}` on the affected device; confirm the new/edited text is present.
- Bulk: spot-check 2-3 devices from the set with `list_device_notes` and confirm the note appears with the right author/timestamp.

## Output format
**Device Notes — [Device or Customer] — [Date]**

For a read (per device):
| Timestamp | Author | Note |
|---|---|---|
| ... | ... | ... |

Sort notes newest-first. Repeat the table per device when reporting multiple.

For a write, after confirmation and verification:
| Device | deviceId | Action | Result |
|---|---|---|---|
| WEB-PROD-01 | 101 | add | OK (verified) |

List any bulk partial failures in a separate **Failures** subsection (device + reason).

End with a one-line summary: total notes read, or `Notes written: N devices — M verified, K failed`.
