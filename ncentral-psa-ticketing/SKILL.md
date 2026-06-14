---
description: Create and inspect Custom PSA tickets and audit PSA customer mappings in N-central. Triggers on "create psa ticket", "psa tickets", "ticket for [device]", "psa mapping", "map customer to psa company", "list tickets", "psa companies".
---

# N-central PSA Ticketing & Mapping

Work with Custom PSA tickets and PSA customer mappings. Uses the **N-central MCP** (classic N-central REST API).

Ticket creation and mapping updates are WRITE ops — always preview and confirm before writing.

## What's available
- **Tickets are Custom PSA ONLY.** `list_custom_psa_tickets`, `get_custom_psa_ticket_detail`, `create_custom_psa_ticket` work only when the customer is on the Custom PSA integration. They do not touch ConnectWise/Autotask/etc.
- **Mappings are Standard PSA.** They link an N-central `customerId` to a PSA company: `get_psa_customer_mapping`, `list_psa_customer_mappings`, `update_psa_customer_mappings`. These are a SEPARATE integration from Custom PSA tickets above — do not assume a Custom PSA ticket customer has a Standard PSA mapping or vice versa.
- **PSA targets** (Standard PSA companies/contacts/sites to map to): `list_psa_companies`, `list_psa_company_contacts`, `list_psa_company_sites`. `psaCompanyId` is a STRING on the contacts/sites calls.
- `validate_psa_credential` ONLY supports TigerPaw 3.0 — if asked to validate any other PSA, say so and skip it.

## SECURITY
PSA credentials passed to `get_custom_psa_ticket_detail` (and any creds in `update_psa_customer_mappings` body) are SENSITIVE. They travel in the request body. NEVER echo, log, or print them back to the user. Ask the user to supply them at call time and discard after.

## Step 1 — List tickets
```json
{}
```
Call `list_custom_psa_tickets` with empty args. Returns all Custom PSA tickets visible to the user. Use this to find a `customPsaTicketId` for detail lookup.

## Step 2 — Ticket detail (creds required)
```json
{ "customPsaTicketId": "4412", "username": "<psa-user>", "password": "<psa-pass>" }
```
`get_custom_psa_ticket_detail` is a POST — creds go in the body. `customPsaTicketId` is a STRING. SENSITIVE; do not echo. If the user has not supplied creds, ask before calling.

## Step 3 — Mapping audit (read-only)
For each customer in scope, check whether a PSA mapping exists.
```json
{ "customerId": 101 }
```
Call `get_psa_customer_mapping` (single) or `list_psa_customer_mappings` (all mappings for that customer). A customer with NO mapping is a coverage gap — flag it. Resolve customer IDs with `list_customers` or `report_org_hierarchy` when you only have names.

## Step 4 — Discover PSA targets (for an unmapped customer)
```json
{ "customerId": 101 }
```
`list_psa_companies` returns mappable PSA companies. Then `list_psa_company_contacts { customerId, psaCompanyId }` and `list_psa_company_sites { customerId, psaCompanyId }` enumerate the contacts/sites under a chosen company so the user can pick a target.

> **Gotcha:** for a customer with **no Standard PSA integration configured**, `list_psa_companies` and the mapping calls return **HTTP 500**, not an empty list. Treat a 500 here as "no Standard PSA for this customer" — report it as a coverage gap and move on; don't abort the whole run.

## Step 5 — Draft a ticket, then PREVIEW + CONFIRM (WRITE)
Build the ticket body, then show the draft and STOP for confirmation. Do not call `create_custom_psa_ticket` yet.

Preview block to show the user:
```
TICKET PREVIEW — create_custom_psa_ticket
  Customer:  Contoso (customerId 101)
  Device:    SVR-DC01 (deviceId 88123)   ← if ticket is for a device
  Subject:   Disk space critical on SVR-DC01
  Body:      C: at 96%, monitoring alert fired <alert-timestamp>
  Priority:  High
```
Ask explicitly: "Create this Custom PSA ticket? (yes/no)". Only on an affirmative reply, call:
```json
{ "body": { "customerId": 101, "subject": "Disk space critical on SVR-DC01", "description": "C: at 96%, monitoring alert fired <alert-timestamp>", "priority": "High" } }
```

## Step 6 — Verify the create
After `create_custom_psa_ticket` returns, capture the new ticket id from the response and confirm it landed:
```json
{}
```
Re-run `list_custom_psa_tickets` and confirm the new id appears. Report the returned ticket id.

## Step 7 — Update a mapping — PREVIEW + CONFIRM (WRITE)
To map/remap a customer to a PSA company, show the before/after first.

Preview block:
```
MAPPING PREVIEW — update_psa_customer_mappings (customerId 101)
  Current:  <none>
  New:      PSA company "Contoso Ltd" (psaCompanyId 7), site "HQ" (siteId 3)
```
Ask "Apply this mapping? (yes/no)". On yes:
```json
{ "customerId": 101, "body": { "psaCompanyId": 7, "psaSiteId": 3 } }
```
Then verify with `get_psa_customer_mapping { customerId: 101 }` and show the now-current mapping.

## Output format

**Ticket list** —
**PSA Tickets — [Customer/Scope] — [Date]**
| Ticket ID | Customer | Subject | Status | Priority | Created |
|-----------|----------|---------|--------|----------|---------|

Sort by Created descending. End with a one-line count: `N tickets`.

**Ticket detail** — single block: ID, Customer, Subject, Status, Priority, Assignee, Created, Body.

**Mapping coverage** —
**PSA Mapping Coverage — [Date]**
| Customer | customerId | Mapped? | PSA Company | psaCompanyId |
|----------|-----------|---------|-------------|--------------|

Sort unmapped customers first. End with: `X of Y customers mapped — Z gaps`.

**On create** — show the TICKET PREVIEW block, the confirmation gate, and after creation a one-line result: `Created Custom PSA ticket #<id>`.

Security note (always append): PSA credentials are sensitive and are never echoed in any output.
