---
description: Pre-on-site briefing for a tech visiting a customer location. Triggers on "dispatch brief for [client]", "pre-site brief", "on-site prep for [client]", "what should I know before going to [client]".
---

# Site Dispatch Brief

Generate a situational briefing before a tech goes on-site using the N-able MCP.

## Step 1 — Find customer ID if needed

```graphql
query { organizationSearch(first: 5, where: { name: { contains: "<name>" } }) { nodes { id name } } }
```

## Step 2 — Get all devices at the site

```graphql
query SiteDispatchBrief($customerId: ID!) {
  assetSearch(
    first: 100
    inOrganizations: [$customerId]
  ) {
    totalCount
    nodes {
      id
      name
      operatingSystemInfo { name type }
      chassis { types }
      agentConnection { status statusChangedAt }
      lastBootedAt
      systemInfo { hostname manufacturer model }
      patchManagement { status lastPatchScanTime }
      reboot { isRequired }
      vulnerabilityManagement { status }
      tags { nodes { name } }
      isManaged
    }
  }
}
```

## Step 3 — Script task history on key devices

For any server-class devices, using the N-able MCP call `list_script_tasks_for_asset` with the asset `id` from Step 2 (same GraphQL id space).

Validate queries before executing.

## Output format

**Dispatch Brief — [Customer] — [Date]**

**Fleet at this site** (quick table)
| Device | Type | OS | Agent |
|---|---|---|---|
| name | server/workstation | OS name | CONNECTED/DISCONNECTED |

**Open issues right now**
- Disconnected devices (list with last seen date)
- Reboot required (list)
- Patch management inactive (list)

**Devices to be aware of**
- Any device tagged as critical/production
- Servers offline — may be powered down, confirm before treating as hardware failure
- Devices that recently reconnected (potential instability)

**Recent script activity** (for servers)
- Last script run, status, any failures

**Quick reminders**
- Do not reboot servers without confirming with the customer
- Offline devices may be powered off — ask customer before escalating
- Check back with dispatcher when on-site work is complete

Keep this brief — readable in under 2 minutes before arriving.
