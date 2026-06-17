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

## Install

This repo is a Claude Code plugin marketplace. Install all skills with:

```
/plugin marketplace add theonlytruebigmac/nable-skills
/plugin install ncentral@nable-skills
```

Skills are then available namespaced as `/ncentral:<skill-name>` (e.g. `/ncentral:alert-triage`). Pull updates later with `/plugin marketplace update nable-skills`.

The skills depend on the N-able and/or N-central MCP servers below — configure those before use.

### Repository layout

This repo is a **marketplace** (catalog) with **one plugin per N-able product**. Today that's `ncentral`; future products (e.g. N-sight, Cove) become sibling plugins under `plugins/`, each installable on its own.

```
nable-skills/                     ← marketplace (catalog)
├── .claude-plugin/marketplace.json
├── docs/                         ← shared reference
└── plugins/
    └── ncentral/                 ← the N-central product plugin
        ├── .claude-plugin/plugin.json
        └── skills/               ← all 44 N-central skills
```

## MCP servers

The `ncentral` plugin **bundles its MCP servers** ([`plugins/ncentral/.mcp.json`](plugins/ncentral/.mcp.json)), so installing the plugin declares both `nable-mcp` and `N-central-mcp` automatically. You only supply secrets — set these environment variables before launching Claude Code:

| Env var | Used by | Where to get it |
|---|---|---|
| `NABLE_API_TOKEN` | `nable-mcp` (GraphQL) | `https://n-able.app/api-token-management` |
| `NCENTRAL_MCP_PACKAGE` | `N-central-mcp` (REST) | npm package name or local path to your N-central MCP server |
| `NCENTRAL_BASE_URL` | `N-central-mcp` (REST) | `https://<your-ncentral-server>` |
| `NCENTRAL_API_TOKEN` | `N-central-mcp` (REST) | **N-central → User Administration → your user → API Access** |

```bash
export NABLE_API_TOKEN=...
export NCENTRAL_MCP_PACKAGE=@yourco/n-central-mcp   # or /path/to/server
export NCENTRAL_BASE_URL=https://your-ncentral-server
export NCENTRAL_API_TOKEN=...
claude
```

Run `/mcp` to check connection status. An **unset variable makes only that server fail** — the skills still load, and you can run either MCP on its own (the `hybrid-*` skills assume both). `nable-mcp` is reached directly through `mcp-remote`; if your N-able endpoint speaks streamable HTTP natively you can swap it for a `{"type":"http","url":...,"headers":{...}}` entry.

<details>
<summary>Prefer manual config instead of the bundled servers?</summary>

Define them yourself in `~/.claude/mcp.json` (and don't *also* rely on the bundle — declaring the same server name in both places is ambiguous):

```json
{
  "mcpServers": {
    "nable-mcp": {
      "command": "npx",
      "args": ["mcp-remote@latest", "https://api.n-able.com/mcp-preview", "--header", "Authorization: Bearer YOUR_TOKEN"]
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
</details>

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

Confirmed GraphQL queries (used by current skills): `assetSearch`, `asset(id)`,
`assetAggregations`, `patchInstallationSearch`, `patchInstallationAggregations`,
`vulnerabilityDetectionSearch`, `vulnerabilityDetectionAggregations`,
`organizationSearch`, `taskSearch`, `taskAggregations`, `tagSearch`. The `asset(id)`
node also exposes nested `activitySearch` and `taskExecutionSearch` (used by
`incident-summary`).

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

### N-central MCP — classic REST skills

#### Monitoring & ops
| Skill | What it does | Cadence |
|---|---|---|
| `active-issues` | Live monitoring-alert triage (the real alerts) | Daily / on-call |
| `scheduled-tasks` | Scheduled-task & job-status monitor | Daily / weekly |
| `maintenance-windows` | Review/create/update patch windows (write) | Monthly / onboarding |

#### Asset & lifecycle
| Skill | What it does | Cadence |
|---|---|---|
| `warranty-lifecycle` | Warranty + hardware-refresh forecast & budget | Quarterly |
| `custom-property-audit` | MSP metadata completeness + bulk fill (write) | Monthly |
| `device-notes` | Read/add/bulk device runbook notes (write) | On demand |
| `asset-inventory` | Hardware/software asset inventory | Quarterly |

#### Admin & onboarding
| Skill | What it does | Cadence |
|---|---|---|
| `psa-ticketing` | PSA tickets + customer↔company mapping (write) | On demand |
| `deployment-kit` | Tokens/installers/keys to deploy the agent | On onboarding |
| `license-capacity` | License limits vs. usage / headroom | Monthly |
| `access-review` | Users, roles, access groups audit | Quarterly |
| `org-hierarchy` | SO→Customer→Site export + ID lookup | On demand |
| `patch-comparison` | N-central native patch comparison report | Weekly |
| `fleet-export` | Full spreadsheet-ready device export | On demand |
| `server-health` | N-central server version/uptime/clock drift | Weekly |

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

### Meta / authoring

| Skill | What it does | Cadence |
|---|---|---|
| `create-skill` | Token-minimal template, token-cost model, and review checklist for authoring a new skill in this library | On demand |

---

## Gaps & guidance

- **Live alerts** live in N-central (`list_active_issues`), not GraphQL — use the
  `active-issues` / `hybrid-alert-triage` skills for real alert state.
- **CVE vulnerabilities, tags, and CPU/memory metrics** live in GraphQL, not REST.
- **Month-over-month deltas**: both APIs are current-state — use `snapshot` to build trends.
- **Ticket response times** live in your PSA; N-central can create/read Custom PSA tickets
  and mappings via `psa-ticketing`.
- Writes (tagging, maintenance windows, custom properties, notes, lifecycle, tickets,
  provisioning) always **preview and confirm** before executing.
