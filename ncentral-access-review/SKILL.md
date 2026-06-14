---
description: Security and compliance review of N-central users, roles, and access groups to surface unprotected accounts, over-broad access, and risky group settings. Triggers on "access review", "who has access", "user audit", "roles and permissions", "access groups", "RBAC review", "offboarding check".
---

# N-central Access & RBAC Review

Audit who can log in, what they can do, and what they can see across the org tree. Read-only by default.

Uses the **N-central MCP** (classic N-central REST API).

## What's available
- `report_all_users_by_so` {soId} — deduped user roster across a Service Org and all its customers (preferred). Fall back to `list_all_users` {all:true} for the whole environment, or `list_users` {orgUnitId} for a single scope. **Each user object carries (verified):** `userId`, `userName`, `fullName`, `isEnabled`, `isLocked`, `twoFactorEnabled`, `apiOnlyUser`, `loggedInUser` (currently logged in — NOT a history), `currentSsoProvider`, `customerTree`, and the assigned `roleIds[]` and `accessGroupIds[]` directly on the user.
- `list_user_roles` {orgUnitId}; `get_user_role` {orgUnitId, userRoleId} — role names and the permissions each grants. (The roster's per-user `roleIds[]` is the fastest way to map users↔roles.)
- `list_access_groups` {orgUnitId} → `{ data:[ { groupId, groupName, groupType:"DEVICE"|"ORG_UNIT", _extra:{ readonly, usernames:[...], cloneable } } ] }`. **`totalItems` is null for this endpoint — paginate with `all:true` / follow `nextPage`.** Drill with `get_access_group` (pass the `groupId` as `accessGroupId`) → `{ data:{ userIds:[...], orgUnitIds:[...], deviceIds:[...], autoIncludeNewOrgUnits } }`.
- `get_current_user` {} — context for who is running the review.
- `report_org_hierarchy` {} — resolve soId / customerId / siteId and names.

> **No last-login timestamp is exposed by the API** — you CANNOT detect "dormant / hasn't logged in for N days". Flag accounts on what IS available: `isEnabled:false`, `isLocked:true`, `twoFactorEnabled:false`, and `apiOnlyUser`. `loggedInUser` only tells you who is logged in right now.

Org tree: Service Org (`soId`) -> Customer (`customerId`) -> Site (`siteId`). A numeric `orgUnitId` addresses any level.

## Step 1 — Establish context and resolve scope
```json
{}
```
Run `get_current_user` first (no args). If the user named a Service Org or customer, resolve IDs with `report_org_hierarchy`; otherwise list targets with `list_service_orgs`.

## Step 2 — Pull the user roster
Prefer the deduped SO report. It collapses users that appear at both SO and customer level.
```json
{ "soId": 50, "format": "json" }
```
Call: `report_all_users_by_so`. For a full-environment sweep instead use `list_all_users` {all:true}, or scope a single org unit with `list_users` {orgUnitId}.

Capture per user: `fullName`, `userName`, `isEnabled`, `isLocked`, `twoFactorEnabled`, `apiOnlyUser`, `roleIds[]`, `accessGroupIds[]`, and `customerTree` (their scope). There is no last-login field — do not invent one.

## Step 3 — Enumerate roles and their permissions
```json
{ "orgUnitId": 50, "all": true }
```
Call `list_user_roles` for each scope. Then expand each role to see granted permissions:
```json
{ "orgUnitId": 50, "userRoleId": 1234 }
```
Call `get_user_role`. Note roles granting all/administrative permissions. The role object does not carry a user count — derive orphaned roles (zero assigned users) by cross-referencing the role names against the per-user role assignments captured in the Step 2 roster.

## Step 4 — Enumerate access groups and scope
```json
{ "orgUnitId": 50, "all": true }
```
Call `list_access_groups` (use `all:true` — `totalItems` is null so you must follow pages). The list already gives member display names in `_extra.usernames` and the `groupType`. Expand a group for exact scope by passing its `groupId` as `accessGroupId`:
```json
{ "accessGroupId": "789" }
```
Call `get_access_group`. Record member `userIds`, the `orgUnitIds`/`deviceIds` in scope, and whether `autoIncludeNewOrgUnits` is on.

## Step 5 — Flag findings
Apply these rules across the collected data:
- **No 2FA** — `twoFactorEnabled: false` on an enabled, non-API user (the top security gap, since last-login isn't available).
- **Locked / disabled-but-present** — `isLocked: true`, or `isEnabled: false` accounts still sitting in access groups (offboarding residue).
- **API-only accounts** — `apiOnlyUser: true`; inventory them and confirm each is an expected integration, not a stray token.
- **Over-broad access** — users whose `accessGroupIds` include all-customer / SO-wide groups (e.g. an "All Customers" or "All SO" group), or assigned an all-permission role.
- **Auto-include groups** — access groups with `autoIncludeNewOrgUnits: true` (silently grant access to future org units).
- **Orphaned roles** — a `userRoleId` that no roster user references in its `roleIds[]`.

This skill is read-only. If the review leads to a change (disabling a user, editing a role or access group, e.g. `create_user_role` / `create_access_group`), STOP: present a preview of exactly what will change (target ID, current value -> new value) and ask for explicit confirmation before any write, then re-run the relevant `get_*`/`list_*` call afterward to verify the change landed.

## Output format
**N-central Access & RBAC Review — [SO/Customer] — [today]**

Reviewed by: [current user]

### User × Role × Scope
| User | Username | Enabled | Locked | 2FA | API-only | Role(s) | Scope | Flag |
|------|----------|---------|--------|-----|----------|---------|-------|------|

Sort flagged users first, then no-2FA users next.

### Access Groups
| Group | Description | Members | Org Units in Scope | Auto-Include New | Flag |
|-------|-------------|---------|--------------------|-----------------|------|

Sort flagged groups first.

### Roles
| Role | # Users | Permission Breadth | Flag |
|------|---------|--------------------|------|

Mark all-permission and zero-user (orphaned) roles.

### Findings
- No-2FA accounts: [list]
- Locked / disabled-but-present: [list]
- API-only accounts: [list]
- Over-broad access: [list]
- Auto-include access groups: [list]
- Orphaned roles: [list]

Close with: **N users reviewed, M flagged (T no-2FA, L locked/disabled, B over-broad, A auto-include groups, O orphaned roles).**
