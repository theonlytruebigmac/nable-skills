# Which MCP? — capability map & decision guide

This library wraps **two** N-able/N-central APIs. They overlap a little and complement
each other a lot. Use this guide to pick the right one (or both).

| MCP | Server name | API | Strengths |
|---|---|---|---|
| **N-able MCP** | `nable-mcp` | GraphQL preview (`assetSearch`, `vulnerabilityDetectionSearch`, …) | CVE-level vulnerabilities, per-patch install records, script-execution metrics, CPU/memory metrics, tags, flexible cross-fleet search |
| **N-central MCP** | `N-central-mcp` | Classic N-central REST | Live monitoring alerts, PSA/ticketing, custom properties, maintenance windows, scheduled-task status, warranty/asset lifecycle, users/roles/access, device notes, agent deployment, license limits, org provisioning |

## Capability matrix

| I want… | N-able GraphQL | N-central REST | Skill(s) |
|---|:---:|:---:|---|
| Live monitoring **alerts** (service down/warning) | ✗ (no alert type) | ✅ `list_active_issues`, `get_device_status` | `ncentral-active-issues`, `hybrid-alert-triage` |
| **CVE** vulnerability detections | ✅ | ✗ | `alert-triage`, `patch-status` (GraphQL) |
| Per-patch **install/error** records | ✅ | partial (report) | `patch-status`, `hybrid-patch-reconciliation` |
| **Patch maintenance windows** | ✗ | ✅ | `ncentral-maintenance-windows` |
| **Scheduled task / job** status | partial (history) | ✅ | `ncentral-scheduled-tasks` |
| **Custom properties** (MSP metadata) | ✗ | ✅ | `ncentral-custom-property-audit` |
| **Warranty / asset lifecycle** | ✗ | ✅ | `ncentral-warranty-lifecycle` |
| Hardware/software **asset inventory** | partial | ✅ `get_device_assets` | `ncentral-asset-inventory` |
| **PSA / tickets / company mapping** | ✗ | ✅ | `ncentral-psa-ticketing` |
| **Users / roles / access groups** | ✗ | ✅ | `ncentral-access-review` |
| **Device notes** | ✗ | ✅ | `ncentral-device-notes` |
| **Agent deployment** (tokens/installers/keys) | ✗ | ✅ | `ncentral-deployment-kit` |
| **License / capacity** limits | ✗ | ✅ | `ncentral-license-capacity` |
| **Tags** | ✅ | ✗ | `tag-devices` (GraphQL) |
| CPU / memory **metrics** | ✅ | ✗ | `device-health-check` (GraphQL) |
| **Org hierarchy** export & ID lookup | partial (`organizationSearch`) | ✅ `report_org_hierarchy` | `ncentral-org-hierarchy` |
| **Org provisioning** (create SO/customer/site/device) | ✗ | ✅ | `ncentral-deployment-kit` |

## When to reach for a hybrid skill

Use a `hybrid-*` skill when the answer needs data **only available on different sides**:

- **One device, everything** → `hybrid-device-360` (metrics + CVEs from GraphQL; live
  service status, warranty, custom props, notes from N-central).
- **Triage that's actually prioritized** → `hybrid-alert-triage` (N-central's real alerts,
  ranked by GraphQL CVE/patch risk).
- **Whole account review** → `hybrid-customer-360`.
- **Is an unpatched device actually scheduled to patch?** → `hybrid-patch-reconciliation`
  (GraphQL patch state × N-central maintenance windows).
- **Incident write-up + open a ticket** → `hybrid-incident-bridge`.
- **Richer health / onboarding / QBR** → `hybrid-health-score-plus`, `hybrid-onboarding-plus`,
  `hybrid-qbr-plus` (the original GraphQL skills plus N-central business signals).

## The one gotcha that bites everyone

The two MCPs use **different ID spaces**. Never pass a GraphQL id to an N-central tool or
vice versa. Join across them by **customer name** and **device name/hostname**. See
[Cross-MCP correlation](ncentral-mcp-reference.md#cross-mcp-correlation).
