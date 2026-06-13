---
description: Assemble everything needed to deploy the N-central agent to a customer or site — registration token, installer download link, and optional device pre-creation with activation key. Triggers on "deploy agent", "install agent for [client]", "registration token", "download link", "agent installer", "activation key", "onboard new device".
---

# N-central Agent Deployment Kit

Build a complete "Deployment Packet" so a tech can install the N-central agent at a customer/site in one pass. Uses the **N-central MCP** (classic N-central REST API).

SECURITY: registration tokens, download links, and activation keys are credentials. They let any host enroll into the customer's tenant. Warn the operator NOT to paste them into shared chat, tickets, email, or unsecured channels — hand them off via a secure/ephemeral method.

## What's available
- `report_org_hierarchy` / `list_customers` / `list_sites` — resolve customer/site names to numeric IDs.
- `get_registration_token` — the token a device registers with; scoped per entity.
- `get_software_installers` — available installers + their `softwareId` for a customer.
- `generate_software_download_link` — the actual time-limited download URL for a `softwareId`.
- `create_device` (WRITE) — pre-create a device record before the agent checks in.
- `get_device_activation_key` — per-device activation key (only after the device exists).

## Step 1 — Resolve the customer/site IDs
Org hierarchy is Service Org -> Customer (`customerId`) -> Site (`siteId`). Start from a name.
```json
{ "tool": "report_org_hierarchy", "args": { "format": "json" } }
```
Match the customer/site name to its numeric ID. If only a customer was named, list its sites with `list_sites {customerId}` and confirm which site the device belongs to before pulling a token.

## Step 2 — Get the registration token
Pick the narrowest scope the operator asked for: a site enrolls into that site, a customer into the customer's default. `entityType` is `"site"`, `"orgUnit"`, or `"customer"`; `id` is that entity's numeric ID.
```json
{ "tool": "get_registration_token", "args": { "entityType": "site", "id": 12345 } }
```
This token is sensitive (see SECURITY above). Record it for the packet; do not echo it into anything shared.

## Step 3 — List installers and get the download link
List the customer's installers, pick the right one for the target OS/architecture from the returned rows, then turn its `softwareId` into a URL. `softwareType` filters by software (e.g. `"agent"`); `installerType` filters by package format (e.g. `"msi"`, `"exe"`) — neither is an OS filter, so read the OS/arch off each installer row rather than filtering by it.
```json
{ "tool": "get_software_installers", "args": { "customerId": 678, "softwareType": "agent", "installerType": "exe" } }
```
`softwareId` is a string — pass it through verbatim from the installer row, do not coerce it to a number.
```json
{ "tool": "generate_software_download_link", "args": { "customerId": 678, "softwareId": "901" } }
```
The download URL is time-limited and credential-bearing — treat it like the token.

## Step 4 (OPTIONAL, WRITE) — Pre-create the device + activation key
Only when the operator wants the device record staged ahead of install. This is a mutation. First show a preview and get explicit confirmation.

Preview and confirm — print exactly what will be created and STOP for a yes:
> About to CREATE device under customer 678:
> - longName: `WS-FRONTDESK-01`
> - networkAddress: `10.0.4.21`
> - supportedOs: `Windows 11`
> - deviceClass: `Workstation - Windows`
>
> Confirm creation? (yes/no)

On explicit `yes`, create it:
```json
{ "tool": "create_device", "args": { "body": {
  "customerId": 678,
  "longName": "WS-FRONTDESK-01",
  "networkAddress": "10.0.4.21",
  "supportedOs": "Windows 11",
  "deviceClass": "Workstation - Windows",
  "description": "Pre-staged for agent deploy 2026-06-13"
} } }
```
Then pull the per-device activation key (needs the new `deviceId`):
```json
{ "tool": "get_device_activation_key", "args": { "deviceId": "555123" } }
```

## Step 5 (verify, only if Step 4 ran) — Confirm the device exists
After a create, read the record back so the packet reflects reality, not the request.
```json
{ "tool": "get_device", "args": { "deviceId": "555123" } }
```
Confirm `longName`, `customerId`, and `networkAddress` match what was submitted. If `create_device` returned a partial/error, surface it — do not hand off a packet for a device that didn't land.

## Output format

**Deployment Packet — [Customer] / [Site] — 2026-06-13**

Target
| Field | Value |
|---|---|
| Customer (ID) | [name] (678) |
| Site (ID) | [name] (12345) |
| Supported OS | [os] |

Credentials (HANDLE SECURELY — do not paste into shared channels)
| Item | Value |
|---|---|
| Registration token | [token] |
| Download URL | [url] |
| Activation key | [key — or "n/a, device not pre-created"] |

Install notes
- Run the installer on the target host; supply the registration token when prompted.
- Match the installer to the host OS/architecture (Step 3 selection).
- If the device was pre-created (Step 4), the activation key binds the agent to record `[deviceId]`.

Close with a one-line summary: customer/site, installer OS, and whether a device was pre-created (yes/no + deviceId).
