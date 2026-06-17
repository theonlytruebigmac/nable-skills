---
description: Run N-central's native async patch comparison report and summarize installs, approvals, and categories since a start date. Triggers on "patch comparison report", "n-central patch report", "patch install history", "patch approvals report", "what patched since [date]".
---

# N-central Patch Comparison Report

Run N-central's built-in patch comparison report (catalog-level, asynchronous) and summarize patch activity over a window. Uses the **N-central MCP** (classic N-central REST API).

This is read-only: it submits a report job and polls for the result. No mutations, no confirmation gate required.

## What's available
- `generate_patch_comparison_report` — submits the report job; returns a `reportId`. ASYNC.
- `get_report` — fetches a finished report by `reportId`. May return "not ready"/empty until generation completes — poll a few times.
- `get_server_time` — sanity-check the clock before picking `startDate`.

Filter args on `generate_patch_comparison_report`:
- `startDate` — REQUIRED, ISO-8601 (`YYYY-MM-DD` or full timestamp). Window is start-date to now.
- `installStatuses?` — array, e.g. `["INSTALLED","FAILED","PENDING","NOT_APPROVED"]`. Omit for all.
- `patchApprovals?` — array, e.g. `["APPROVED","DECLINED","NOT_APPROVED"]`. Omit for all.
- `patchCategories?` — array, e.g. `["SECURITY","CRITICAL","UPDATE_ROLLUP"]`. Omit for all.

## Relationship to other skills
- This is N-central's **native catalog-level** report — what the platform shipped/approved across its patch catalog.
- The GraphQL `patch-status` skill gives **live per-asset** install records (N-able MCP). Different source, different granularity.
- For a combined view that reconciles native vs. per-asset data, cross-link the `hybrid-patch-reconciliation` skill.

## Step 1 — Confirm the window
Resolve `startDate` from the user's phrasing ("since last month", "since 2026-05-01"). If ambiguous, check server time first so the window is anchored to N-central's clock, not yours.

```json
{}
```
Call `get_server_time` (no args) to confirm current date before computing a relative start.

## Step 2 — Submit the report (async)
Submit with the resolved filters. Capture the returned `reportId`.

```json
{
  "startDate": "2026-05-01",
  "installStatuses": ["INSTALLED", "FAILED", "PENDING"],
  "patchApprovals": ["APPROVED", "DECLINED"],
  "patchCategories": ["SECURITY", "CRITICAL"]
}
```
Pass `generate_patch_comparison_report` these args. Omit any filter array to include all values for that dimension. The response is a `reportId`, NOT the data — generation runs asynchronously on the server.

## Step 3 — Poll for the result
Fetch the report by id. If it returns empty / "not ready" / a still-generating status, wait briefly and re-fetch. Poll a few times (e.g. up to 3-5 attempts with a short pause) before giving up.

```json
{
  "reportId": "<reportId from Step 2>"
}
```
Pass `get_report` the id. Once it returns rows, stop polling. If still not ready after several attempts, report that the job is still generating and surface the `reportId` so the user can re-fetch later.

## Step 4 — Summarize the window
Aggregate the returned rows into counts across three dimensions: install status, approval, and category. Total the distinct patches and note the window boundaries (`startDate` to now).

## Output format
**Patch Comparison Report — [startDate] to [today] — [Date generated]**

Report ID: `<reportId>`

By install status (sort by count desc):
| Install Status | Count |
|---|---|
| INSTALLED | n |
| FAILED | n |
| PENDING | n |
| ... | ... |

By approval (sort by count desc):
| Approval | Count |
|---|---|
| APPROVED | n |
| DECLINED | n |
| NOT_APPROVED | n |

By category (sort by count desc):
| Category | Count |
|---|---|
| SECURITY | n |
| CRITICAL | n |
| ... | ... |

End with a one-line summary: total distinct patches in window, count installed vs. failed, and a pointer to run `patch-status` (live per-asset) or `hybrid-patch-reconciliation` for deeper reconciliation. If the job never finished, state that and echo the `reportId`.
