# N-central MCP Reference (classic N-central REST API)

Companion reference for the `ncentral-*` and `hybrid-*` skills. This documents the
**classic N-central REST API** exposed by the `N-central-mcp` server — a different
surface from the N-able GraphQL preview (`nable-mcp`) used by the original skills.

> Tool ids are `mcp__N-central-mcp__<name>`. Skills reference them by bare name
> (e.g. `list_active_issues`), the same way the GraphQL skills reference `validate`/`execute`.

## MCP config (`~/.claude/mcp.json`)

```json
{
  "mcpServers": {
    "N-central-mcp": {
      "command": "npx",
      "args": ["-y", "<your-n-central-mcp-package-or-path>"],
      "env": {
        "NCENTRAL_BASE_URL": "https://<your-ncentral-server>",
        "NCENTRAL_API_TOKEN": "YOUR_JWT_OR_API_TOKEN"
      }
    }
  }
}
```

Generate an API-access (JWT) token in **N-central → User Administration → your user →
API Access**. Both MCPs can be configured side by side; the hybrid skills assume both
are connected.

---

## Org hierarchy & IDs

```
Service Organization (soId)
  └── Customer (customerId)
        └── Site (siteId)
              └── Device (deviceId)
```

- A generic numeric **`orgUnitId`** addresses **any** level (SO, customer, or site).
- **`deviceId` is a string** in most read tools; create/bulk tools take numeric
  `deviceIDs` arrays.
- Need an ID? Run `report_org_hierarchy` (flat SO→Customer→Site with IDs) or the
  `org-hierarchy` skill.

> **These IDs are NOT the same values as the GraphQL API's org/asset IDs.** See
> [Cross-MCP correlation](#cross-mcp-correlation) before mixing the two.

---

## Global call conventions

Most `list_*` and `report_*` tools accept:

| Param | Meaning |
|---|---|
| `all: true` | Auto-paginate — fetch every page and return the combined list. Omit for a single (cheaper) page. |
| `pageNumber` / `pageSize` | Manual paging. `pageSize` max **200**. |
| `format` | `"json"` or `"csv"`. `list_*` default **json**, `report_*` default **csv**. |
| `sortBy` / `sortOrder` | Sort field + direction (`asc`/`desc`/…). |
| `select` | A **FIQL/RSQL row filter** like `soId==50`, joined with `;` for AND. **Despite the name it filters rows, it does NOT pick fields.** Unsupported fields error with `Field not found: X`. |

---

## Tool catalog

### Devices & live monitoring
| Tool | Purpose / notes |
|---|---|
| `list_devices` | All devices for the user. `filterId` applies a saved filter. |
| `list_devices_by_org_unit` | Devices under an `orgUnitId`. |
| `get_device` | One device. ⚠ `lastLoggedInUser`/`stillLoggedIn` may be null — get those from `list_devices`. |
| `get_device_assets` | Hardware/software asset inventory. ⚠ Probes return 404. |
| `get_device_status` | Status of the device's **service-monitoring** tasks — live monitoring state. |
| `list_active_issues` | Active monitoring issues for an org unit. ⚠ **Customer/site org units only** (not SOs — iterate a SO's customers). Known bug: `_extra.deviceClassValue/Label` always null. |
| `list_job_statuses` | Job statuses for an org unit. |
| `list_device_filters` | Saved device filters (feed `filterId`). |
| `list_device_tasks` | Scheduled tasks on a device. |

### Scheduled tasks
| Tool | Purpose / notes |
|---|---|
| `list_scheduled_tasks` | All scheduled tasks in the environment. |
| `get_scheduled_task` | parentId, name, type, customerId, deviceIds, enabled. |
| `get_scheduled_task_status` | Aggregated status; `detailed:true` gives per-device breakdown but **only for SYSTEM/CUSTOMER-level task ids** (navigate via `parentId`). |
| `get_appliance_task` | Appliance-task info. |

### Custom properties (MSP metadata)
| Tool | Purpose / notes |
|---|---|
| `list_org_custom_properties` | Property **definitions** for an org unit. |
| `list_device_custom_properties` | All custom props on one device. |
| `get_device_custom_property` | One device property. |
| `update_device_custom_property` | ⚠ PUT requires `propertyName` + `propertyType`. `propertyType` ∈ `HTML_LINK\|TEXT\|DATE\|ENUMERATED\|PASSWORD`, must match the definition. For `ENUMERATED`, also pass the optional `enumeratedValueList`. |
| `get_org_custom_property_default` / `get_device_default_custom_property` | Default values. `get_device_default_custom_property` (orgUnitId + propertyId) also returns the `ENUMERATED` allowed-value list. |
| `update_org_unit_custom_property` | Set a property on an SO/customer/site. |
| `update_org_custom_property_default` | Update a default + `propagationType` to push down the hierarchy. |
| `report_devices_bulk` | **Fan-out workhorse** — runs a per-device call across an org unit. `dataType` ∈ `custom-properties\|assets\|monitor-status`. |

**`propagationType`** (for `update_org_custom_property_default`) ∈ `NO_PROPAGATION`, `SERVICE_ORGANIZATION_ONLY`, `SERVICE_ORGANIZATION_AND_CUSTOMER`, `SERVICE_ORGANIZATION_AND_SITE`, `SERVICE_ORGANIZATION_AND_CUSTOMER_AND_SITE`, `CUSTOMER_ONLY`, `CUSTOMER_AND_SITE`, `SITE_ONLY`, `ORGANIZATION_ONLY`, `DEVICE_ONLY` — pick the one matching which levels should inherit (list illustrative; an unusual level may need a different value). For an ENUMERATED property, fetch allowed values first via `get_device_default_custom_property` (orgUnitId + propertyId) or `get_device_custom_property`.

### Maintenance windows (write)
| Tool | Purpose / notes |
|---|---|
| `get_maintenance_windows` | Windows for a device. |
| `create_maintenance_windows` | `{deviceIDs:[number], maintenanceWindows:[…]}`. ⚠ Partial failures returned per-device — inspect. |
| `update_maintenance_windows` | `{maintenanceWindows:[{scheduleId,…}]}`. |

### Asset lifecycle / warranty (write)
| Tool | Purpose / notes |
|---|---|
| `get_device_lifecycle` | warranty/lease/purchase/cost/location/assetTag/expectedReplacementDate. |
| `update_device_lifecycle` | PUT — **all** fields required (replace). |
| `patch_device_lifecycle` | PATCH — any subset (partial). Dates ISO-8601/`YYYY-MM-DD`. |

### Device notes (write)
`list_device_notes` · `add_device_note` · `update_device_note` · `add_notes_bulk {deviceIDs:[number], text}`

### PSA / ticketing
| Tool | Purpose / notes |
|---|---|
| `list_psa_companies` / `list_psa_company_contacts` / `list_psa_company_sites` | PSA company directory for a customer. |
| `get_psa_customer_mapping` / `list_psa_customer_mappings` | N-central customer ↔ PSA company mapping. |
| `update_psa_customer_mappings` | Update mapping (write). |
| `create_custom_psa_ticket` / `list_custom_psa_tickets` / `get_custom_psa_ticket_detail` | ⚠ **Custom PSA only.** Ticket detail POSTs PSA creds in the body — **sensitive**. |
| `validate_psa_credential` | ⚠ Only works with **TigerPaw 3.0**. |

### Users / roles / access (RBAC)
`list_all_users` · `list_users {orgUnitId}` · `report_all_users_by_so {soId}` (dedupes) ·
`list_user_roles` · `get_user_role` · `create_user_role` ·
`list_access_groups` · `get_access_group` · `create_access_group` · `create_device_access_group` ·
`get_current_user` (who am I)

### Deployment / provisioning (write for `create_*`)
| Tool | Purpose / notes |
|---|---|
| `get_registration_token` | `{entityType:"site"\|"orgUnit"\|"customer", id}`. ⚠ Sensitive. |
| `get_software_installers` | Installers + `softwareId` for a customer. |
| `generate_software_download_link` | Actual download URL. ⚠ Sensitive. |
| `get_device_activation_key` | Activation key for a device. ⚠ Sensitive. |
| `create_device` | `{body:{customerId, networkAddress, longName, supportedOs, deviceClass, …}}`. |
| `create_customer` / `create_site` / `create_service_org` | Org provisioning. |

### Org structure & capacity
`list_service_orgs` · `get_service_org` · `list_customers {soId?}` · `get_customer` ·
`list_sites {customerId?}` · `get_site` · `list_org_units` · `get_org_unit` ·
`list_org_unit_children` · `get_org_unit_property` ·
`get_org_unit_limits` (licensing/usage) · `update_org_unit_limits`

### Reports
| Tool | Purpose / notes |
|---|---|
| `report_org_hierarchy` | Flat SO→Customer→Site with IDs, contacts, addresses. |
| `report_customer_site_summary` | Customers + sites + device counts. |
| `report_devices_by_so` | Full device report for a SO. |
| `generate_patch_comparison_report` | `{startDate, installStatuses?, patchApprovals?, patchCategories?}` → returns a `reportId` (**async**). |
| `get_report` | Fetch a report by id (poll until ready). |

### Server
`get_server_info {level:"basic"\|"health"\|"extra"}` · `get_server_time` (clock-drift check) ·
`get_server_info_authenticated`

---

## Cross-MCP correlation

The N-able GraphQL API (`nable-mcp`) and the N-central REST API (`N-central-mcp`) use
**different ID spaces**. A GraphQL organization/asset id is **not** the same value as an
N-central numeric `orgUnitId`/`customerId`/`deviceId`. **Never pass an id from one MCP to
the other.**

To join data across the two, match on stable human attributes:

| Entity | GraphQL side | N-central side | Join key |
|---|---|---|---|
| Customer | `organizationSearch` → org id | `list_customers` / `report_org_hierarchy` → `customerId` | exact **customer name** |
| Device | `assetSearch` name / `systemInfo.hostname` | `list_devices` / `list_devices_by_org_unit` → `deviceId` | device **name** + hostname (OS as tiebreaker) |

Every hybrid skill performs this "resolve IDs in both systems by name" step first, and
flags any device/customer that appears in only one system (it may be unmanaged on one
side or named differently).

---

## Known quirks (quick list)

- `list_active_issues` — customer/site org units only; `deviceClass*` fields null.
- `get_org_unit_limits` — rows `{limitName, value, maxValue}`: `value` = configured/licensed limit, `maxValue` = the raise-to ceiling; **neither is current usage** (derive usage from device counts). Device rows: `MaxDevicesProfessional`/`MaxDevicesEssential`/`MaxDevicesMobile`.
- **Bare-array responses** — `list_active_issues` and `report_devices_bulk` (json, any `dataType`) return a **bare JSON array** with no `{data}` wrapper (`[]` when empty); contrast `get_org_unit_limits`/`get_device_status`/`get_maintenance_windows`, which wrap rows in `{data:[…]}`. `list_active_issues` rows also carry an `_extra` object (deviceName/transitionTime/customerTree).
- `get_scheduled_task_status` `detailed:true` — SYSTEM/CUSTOMER task ids only.
- `get_device_assets` — probes 404.
- `get_device` — `lastLoggedInUser`/`stillLoggedIn` can be null.
- `update_device_lifecycle` — PUT, all fields required; use `patch_device_lifecycle` for partial.
- `update_device_custom_property` — needs `propertyName` + matching `propertyType`.
- Custom PSA tickets only; `validate_psa_credential` is TigerPaw-3.0-only.
- Registration tokens, download links, activation keys, PSA credentials are **sensitive** — don't echo them into shared channels.
