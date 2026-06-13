---
description: Find devices that haven't connected recently — ghost agents, offline devices, license waste. Triggers on "stale devices", "offline devices", "agents not checking in", "ghost devices", "license cleanup".
---

# Stale Device Audit

Identify devices in N-central not actively checking in using the N-able MCP.

## Query

```graphql
query StaleDeviceAudit {
  assetSearch(
    first: 200
    where: { agentConnection: { status: { equals: DISCONNECTED } } }
    orderBy: [{ field: NAME, direction: ASC }]
  ) {
    totalCount
    nodes {
      id
      name
      customer { name }
      serviceOrganization { name }
      operatingSystemInfo { name type }
      chassis { types }
      agentConnection { status statusChangedAt }
      lastBootedAt
      tags { nodes { name } }
      isManaged
      systemInfo { hostname manufacturer model }
    }
  }
}
```

Validate before executing.

## Categorization

Use `agentConnection.statusChangedAt` to calculate days offline:

| Category | Days Since Disconnect | Recommended Action |
|---|---|---|
| Recently Offline | < 7 days | Monitor — may be transient |
| Investigate | 7–30 days | Confirm with customer if device still in use |
| Likely Decommissioned | 30–90 days | Schedule retirement from N-central |
| Remove | 90+ days | Remove agent, retire device record |

## Output format

Summary: total disconnected devices, count per category, estimated licenses to reclaim.

Then list devices grouped by category, sorted by customer:

For each device:
- Device name | Customer | Service Org
- OS | Chassis type | Tags
- Last status change (disconnect date)
- Days offline
- `isManaged` flag
- Recommended action

**Flag any disconnected device tagged as a server or production system** — these need investigation before removal.

Finish with: "Estimated license reclaim: X devices (Likely Decommissioned + Remove categories)."
