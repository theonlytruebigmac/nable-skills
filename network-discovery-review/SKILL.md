---
description: Find devices that are unmanaged or have disconnected agents across the fleet. Triggers on "network discovery", "unmanaged devices", "discovered devices", "shadow IT", "coverage gaps".
---

# Network Discovery Review

Surface devices that are discovered but not under full management using the N-able MCP.

## Query — unmanaged devices

```graphql
query UnmanagedDevices {
  assetSearch(
    first: 200
    where: { isManaged: { equals: false } }
  ) {
    totalCount
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      name
      customer { name }
      serviceOrganization { name }
      operatingSystemInfo { name type }
      chassis { types }
      agentConnection { status statusChangedAt }
      systemInfo { hostname manufacturer model }
      tags { nodes { name } }
      lastBootedAt
      isManaged
    }
  }
}
```

## Query — recently disconnected (potential new coverage gaps)

```graphql
query RecentlyDisconnected {
  assetSearch(
    first: 100
    where: { agentConnection: { status: { equals: DISCONNECTED } } }
    orderBy: [{ field: NAME, direction: ASC }]
  ) {
    totalCount
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      name
      customer { name }
      agentConnection { status statusChangedAt }
      operatingSystemInfo { name type }
      chassis { types }
      isManaged
      tags { nodes { name } }
    }
  }
}
```

Validate before executing. If `totalCount` exceeds the returned node count, page the rest (re-run with `after: <pageInfo.endCursor>`, re-validating, while `pageInfo.hasNextPage` is true) so the count-by-OS and count-by-customer summaries cover the full fleet.

## Classification

| Category | Criteria | Priority |
|---|---|---|
| Unmanaged server | `isManaged=false`, chassis=server type, or OS contains "Server" | HIGH |
| Unmanaged workstation | `isManaged=false`, workstation/laptop chassis | MEDIUM |
| Managed but offline | `isManaged=true`, agent DISCONNECTED | MEDIUM |
| Recently disconnected | Agent disconnected within last 7 days | MONITOR |

## Output format

Summary: total unmanaged devices, count by OS type (Windows/Linux/macOS), count by customer.

List unmanaged devices grouped by customer:
- Device name | Customer | OS | Chassis | Last seen | Tags

Recommended actions per chassis type:
- **Server** → Escalate immediately — unmanaged servers are a security blind spot
- **Workstation/laptop** → Schedule agent deployment with user notification
- **Unknown chassis** → Investigate with customer before deploying
