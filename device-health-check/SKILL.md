---
description: Full health snapshot for a specific device or all devices under a customer. Triggers on "health check for [device/client]", "is [device] healthy", "check [server name]", "device status".
---

# Device Health Check

Pull a comprehensive health snapshot using the N-able MCP.

## For a single known device — use `get_single_asset_details_and_metrics`

If you have the asset ID, call the dedicated tool directly:
- Tool: `get_single_asset_details_and_metrics`
- Required: `id` (assetId), `currentTimestamp` (ISO-8601, set to now)

This returns CPU/memory metrics, full OS info, agent status, logical drives, and more.

If you only have the device name, resolve the ID first:

```graphql
query FindAssetByName($name: String!) {
  assetSearch(
    first: 5
    where: { name: { contains: $name } }
  ) {
    nodes {
      id
      name
      customer { name }
    }
  }
}
```

## For all devices under a customer

```graphql
query CustomerHealthCheck($customerId: ID!) {
  assetSearch(
    first: 100
    inOrganizations: [$customerId]
  ) {
    totalCount
    nodes {
      id
      name
      customer { name }
      operatingSystemInfo { name version type architecture role }
      chassis { types }
      agentConnection { status statusChangedAt }
      lastBootedAt
      systemInfo { hostname manufacturer model serialNumber memoryTotalSizeBytes }
      logicalDrives { driveId totalSizeBytes fileSystem }
      patchManagement { status lastPatchScanTime }
      vulnerabilityManagement { status lastUpdatedAt }
      reboot { isRequired }
      tags { nodes { name } }
      isManaged
    }
  }
}
```

Validate before executing.

## Health classification

Per device, classify as:

| Status | Criteria |
|---|---|
| **HEALTHY** | Agent CONNECTED, no reboot required, patch management ACTIVE |
| **DEGRADED** | Agent connected but reboot required, or patch management INACTIVE/PENDING |
| **OFFLINE** | Agent DISCONNECTED |
| **UNMANAGED** | `isManaged = false` |

## Output format

Per device:
- Name | Customer | OS | Chassis type | Agent status | Last seen
- Patch management status | Last patch scan
- Vulnerability status | Last updated
- Reboot required: Yes/No
- Logical drives: drive letter, size, filesystem
- Tags

Group: OFFLINE, DEGRADED, and UNMANAGED first, then HEALTHY.

End with: X healthy, X degraded, X offline, X unmanaged across N devices.
