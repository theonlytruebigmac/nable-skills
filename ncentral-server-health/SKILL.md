---
description: Quick health and identity check of the N-central server itself — version, uptime, clock drift, and authenticated session. Triggers on "n-central server health", "is n-central up", "server version", "clock drift", "appliance version", "api version", "who am i".
---

# N-central Server Health Check

Read-only status/identity check of the N-central server itself. Uses the **N-central MCP** (classic N-central REST API).

## What's available
- `get_server_info` — server metadata at three depths via `level`. **Verified shapes:**
  - omit `level` (or `"basic"`) → `{ version, ncentral, startTime, jvmVersion }` — API/build version, N-central product version, server **start time**, and JVM. **Uptime is derived from `startTime`** (this level, not `health`).
  - `"health"` → `{ currentTime }` only — a bare liveness ping (NOT uptime).
  - `"extra"` → `{ data: { _extra: { … } } }` — a flat map of component versions & config (e.g. `Installation: UI Product Version`, `License Key Expiry`, `NcentralRegion`, `SSLEnabled`).
- `get_server_time` — `{ serverTime, timezone, utcOffset }`; compare `serverTime` to operator/local clock to detect drift.
- `get_current_user` — `{ status, message, data: { userId, username, … } }`; confirms which account the session is authenticated as.
- `get_server_info_authenticated` — third-party/component versions, but requires raw `username`/`password` in the body (sensitive — only on explicit request).

This skill is read-only. No mutations, no confirmation gate.

## Step 1 — Version and uptime (basic)
```json
{ "level": "basic" }
```
Call `get_server_info` (omit `level` or pass `"basic"`). Read `version` (build), `ncentral` (product version, e.g. `2026.2.0.17`), and `startTime`. **Compute uptime = now − `startTime`.**

## Step 2 — Liveness ping (health)
```json
{ "level": "health" }
```
`level: "health"` returns only `{ currentTime }` — use it as a fast "is the API answering" check. It does NOT return uptime; get start time from Step 1.

## Step 3 — Component / system versions (extra)
```json
{ "level": "extra" }
```
`level: "extra"` returns a `data._extra` map. Pull the useful keys: `Installation: UI Product Version`, `Installation: Deployment Product Version`, `License Key Expiry`, `NcentralRegion`, `SSLEnabled`. Don't dump the whole map.

## Step 4 — Server time and clock drift
```json
{}
```
Call `get_server_time`, then compare the returned server time to the current operator/local clock. Compute the delta in seconds. **Flag drift > 2 minutes** — clock skew breaks task scheduling and token/auth validity.

## Step 5 — Confirm authenticated identity
```json
{}
```
Call `get_current_user` to confirm who the session is authenticated as (the "who am I" check). Report the username/account.

## Step 6 — Third-party component versions (optional, sensitive)
```json
{ "username": "<operator>", "password": "<secret>" }
```
ONLY when the user explicitly asks for third-party/authenticated component versions. `get_server_info_authenticated` takes raw credentials in the body — never log them, never store them, and ask the user to supply them at call time. Skip this step otherwise.

## Output format
**N-central Server Health — [Date]**

| Field | Value |
|---|---|
| Status | UP / DEGRADED |
| API/build version | `version` from `basic` |
| Product version | `ncentral` from `basic` (e.g. 2026.2.0.17) |
| Started | `startTime` from `basic` |
| Uptime | now − `startTime` (derived) |
| Liveness | `currentTime` from `health` |
| Server time | `serverTime` from `get_server_time` |
| Local time | operator clock |
| Clock drift | ±Ns — **FLAG if > 2 min** |
| Authenticated as | from `get_current_user` |
| Component versions | from `extra` (and authenticated, if requested) |

Sort rows in the order above. If clock drift exceeds 2 minutes, add a bold warning line: **CLOCK DRIFT — scheduling/auth at risk.**

End with a one-line summary: server status, version, and drift verdict (e.g. "N-central UP — vYYYY.x — clock in sync.").
