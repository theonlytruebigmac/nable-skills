# N-able / N-central MSP Skill Library

MSP skill library for Claude Code, covering **two** N-able/N-central APIs:

- **N-able MCP** (`nable-mcp`) — the official GraphQL preview endpoint (n-query based).
- **N-central MCP** (`N-central-mcp`) — the classic N-central REST API.

The two are complementary: the GraphQL API is strong on vulnerabilities, per-patch detail,
and metrics; the classic REST API is strong on live monitoring alerts, PSA/ticketing,
custom properties, maintenance windows, warranty/lifecycle, RBAC, and deployment. A set of
**hybrid** skills uses both at once.

📖 **[Which MCP should I use?](docs/which-mcp.md)** · 📖 **[N-central MCP tool reference](docs/ncentral-mcp-reference.md)**

---

## MCP config (`~/.claude/mcp.json`)

```json
{
  "mcpServers": {
    "nable-mcp": {
      "command": "npx",
      "args": [
        "mcp-remote@latest",
        "https://api.n-able.com/mcp-preview",
        "--header",
        "Authorization: Bearer YOUR_TOKEN"
      ]
    },
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

- N-able GraphQL token: `https://n-able.app/api-token-management`.
- N-central API token: **N-central → User Administration → your user → API Access**.

You can run either MCP on its own; the `hybrid-*` skills assume both are connected.

---

## MCP tools

### N-able MCP (GraphQL)

| Tool | Purpose |
|---|---|
| `introspect` | Discover fields on any schema type |
| `search` | Find queryable fields by keyword |
| `validate` | Check a query before running — always first |
| `execute` | Run a GraphQL query against your fleet |
| `get_asset_inventory` | Paginated list of managed devices |
| `get_single_asset_details_and_metrics` | Full detail + CPU/memory for one device |
| `list_script_tasks_for_asset` | Script execution history for a device |
| `assign_tags_to_assets` | Bulk-tag devices (write) |

**Rule:** never `execute` without passing `validate` first.

Confirmed GraphQL queries: `assetSearch`, `asset(id)`, `patchSearch`,
`patchInstallationSearch`, `patchInstallationAggregations`, `patchAggregations`,
`vulnerabilityDetectionSearch`, `vulnerabilityDetectionAggregations`,
`vulnerabilityDetectionByCustomerSearch`, `organizationSearch`, `taskSearch`,
`taskAggregations`, `scriptSearch`, `tagSearch`, `auditRecordSearch`.

### N-central MCP (classic REST)

~80 tools spanning devices, live monitoring (`list_active_issues`, `get_device_status`),
scheduled tasks, custom properties, maintenance windows, warranty/lifecycle, PSA/ticketing,
users/roles/access, device notes, deployment, reports, and org provisioning. See the full
catalog and call conventions in **[docs/ncentral-mcp-reference.md](docs/ncentral-mcp-reference.md)**.

---

## Org hierarchy

```
              N-able GraphQL                         N-central REST
Partner                                     Service Organization (soId)
  └── ServiceOrganization                     └── Customer (customerId)
        └── Customer                                └── Site (siteId)
              └── Site                                    └── Device (deviceId)
                    └── Asset
```

GraphQL `assetSearch` accepts `inOrganizations: [id]` at any level. N-central uses numeric
`orgUnitId`/`soId`/`customerId`/`siteId` and string `deviceId`.

> ⚠ **The two ID spaces are different.** Never pass a GraphQL id to N-central or vice versa —
> join by **customer name** and **device name/hostname**. See
> [Cross-MCP correlation](docs/ncentral-mcp-reference.md#cross-mcp-correlation).

---

## Skills

### N-able MCP — GraphQL skills

#### Engineer
| Skill | What it does | Cadence |
|---|---|---|
| `alert-triage` | Disconnected agents + patch errors + critical vulns | Daily / on-call |
| `patch-status` | Patch compliance by status across customer or fleet | Weekly |
| `patch-approval-queue` | AVAILABLE and APPROVED patches awaiting action | Daily / patch Tuesday |
| `device-health-check` | Full device snapshot with metrics | Before any touch |
| `onboarding-checklist` | Gap analysis vs. MSP baseline | After agent deploy |
| `incident-summary` | Timeline from activity logs + task history | Post-incident |
| `stale-device-audit` | Disconnected devices + license cleanup | Monthly |
| `automation-policy-review` | Script task failures and coverage | Monthly |
| `eol-os-report` | EOL OS inventory with upgrade priority | Quarterly |
| `network-discovery-review` | Unmanaged and disconnected device gaps | Monthly |
| `site-dispatch-brief` | Pre-on-site briefing for field techs | Before every visit |
| `tag-devices` | Assign tags in bulk (write) | On demand |
| `snapshot` | Save point-in-time fleet state for trends | Weekly / before QBR |

#### Executive / Sales
| Skill | What it does | Cadence |
|---|---|---|
| `qbr-brief` | Fleet + patch + vuln summary for QBR | 1 week before QBR |
| `health-score` | Weighted 0–100 environment health score | Monthly |
| `renewal-risk-report` | Customers with deteriorating signals | Weekly |
| `upsell-opportunity-report` | Coverage gaps mapped to service offerings | Monthly |
| `executive-monthly-digest` | Full-fleet KPI memo for leadership | 1st of month |
| `new-prospect-baseline` | Pre-sales scoping assessment | Before SOW |
| `sla-compliance-report` | Patch/vuln age vs. SLA thresholds | Monthly |
| `fleet-growth-report` | Device count and composition by customer | Monthly |

### N-central MCP — classic REST skills (`ncentral-*`)

#### Monitoring & ops
| Skill | What it does | Cadence |
|---|---|---|
| `ncentral-active-issues` | Live monitoring-alert triage (the real alerts) | Daily / on-call |
| `ncentral-scheduled-tasks` | Scheduled-task & job-status monitor | Daily / weekly |
| `ncentral-maintenance-windows` | Review/create/update patch windows (write) | Monthly / onboarding |

#### Asset & lifecycle
| Skill | What it does | Cadence |
|---|---|---|
| `ncentral-warranty-lifecycle` | Warranty + hardware-refresh forecast & budget | Quarterly |
| `ncentral-custom-property-audit` | MSP metadata completeness + bulk fill (write) | Monthly |
| `ncentral-device-notes` | Read/add/bulk device runbook notes (write) | On demand |
| `ncentral-asset-inventory` | Hardware/software asset inventory | Quarterly |

#### Admin & onboarding
| Skill | What it does | Cadence |
|---|---|---|
| `ncentral-psa-ticketing` | PSA tickets + customer↔company mapping (write) | On demand |
| `ncentral-deployment-kit` | Tokens/installers/keys to deploy the agent | On onboarding |
| `ncentral-license-capacity` | License limits vs. usage / headroom | Monthly |
| `ncentral-access-review` | Users, roles, access groups audit | Quarterly |
| `ncentral-org-hierarchy` | SO→Customer→Site export + ID lookup | On demand |
| `ncentral-patch-comparison` | N-central native patch comparison report | Weekly |
| `ncentral-fleet-export` | Full spreadsheet-ready device export | On demand |
| `ncentral-server-health` | N-central server version/uptime/clock drift | Weekly |

### Hybrid skills — both MCPs (`hybrid-*`)

| Skill | What it does | Cadence |
|---|---|---|
| `hybrid-device-360` | One device's complete picture across both APIs | Before any touch |
| `hybrid-alert-triage` | N-central live alerts ranked by GraphQL CVE/patch risk | Daily / on-call |
| `hybrid-customer-360` | Full account rollup: ops + business signals | Before QBR / escalation |
| `hybrid-health-score-plus` | `health-score` + monitoring, warranty, metadata, license | Monthly |
| `hybrid-onboarding-plus` | `onboarding-checklist` + N-central config completeness | After onboarding |
| `hybrid-incident-bridge` | Incident timeline from both, then draft a PSA ticket | Post-incident |
| `hybrid-qbr-plus` | `qbr-brief` + hardware refresh + tickets + license | 1 week before QBR |
| `hybrid-patch-reconciliation` | Unpatched devices × maintenance windows = root cause | Weekly / patch Tuesday |

---

## Gaps & guidance

- **Live alerts** live in N-central (`list_active_issues`), not GraphQL — use the
  `ncentral-active-issues` / `hybrid-alert-triage` skills for real alert state.
- **CVE vulnerabilities, tags, and CPU/memory metrics** live in GraphQL, not REST.
- **Month-over-month deltas**: both APIs are current-state — use `snapshot` to build trends.
- **Ticket response times** live in your PSA; N-central can create/read Custom PSA tickets
  and mappings via `ncentral-psa-ticketing`.
- Writes (tagging, maintenance windows, custom properties, notes, lifecycle, tickets,
  provisioning) always **preview and confirm** before executing.
