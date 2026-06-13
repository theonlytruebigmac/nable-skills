# N-able MCP Skills for N-central
MSP skill library for Claude Code + Official N-able MCP (GraphQL preview endpoint)

## MCP Config (~/.claude/mcp.json)

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
    }
  }
}
```

Get your API token at `https://n-able.app/api-token-management`.

---

## MCP Tools used by these skills

| Tool | Purpose |
|---|---|
| `introspect` | Discover available fields on any schema type |
| `search` | Find queryable fields by keyword |
| `validate` | Check a query before running — always use this first |
| `execute` | Run a GraphQL query against your fleet |
| `get_asset_inventory` | Paginated list of all managed devices |
| `get_single_asset_details_and_metrics` | Full detail + CPU/memory metrics for one device |
| `list_script_tasks_for_asset` | Script execution history for a device |

**Rule:** Never `execute` without passing `validate` first.

---

## Real GraphQL queries available (confirmed from live schema)

| Query | What it returns |
|---|---|
| `assetSearch` | All devices with OS, agent status, patch/vuln management, tags, chassis, metrics |
| `asset(id)` | Single asset by ID |
| `patchSearch` | Patch catalog with eligible/installed/failed counts |
| `patchInstallationSearch` | Per-asset patch installation records with status, errors, completion |
| `patchInstallationAggregations` | Count of patches by status per org |
| `patchAggregations` | Count of patches by classification/severity |
| `vulnerabilityDetectionSearch` | CVE detections per asset with severity, risk score, exploit flags |
| `vulnerabilityDetectionAggregations` | Count of detections by severity/status |
| `vulnerabilityDetectionByCustomerSearch` | Vuln roll-up by customer |
| `organizationSearch` | Find Partners, Service Orgs, Customers, Sites by name |
| `taskSearch` | Script/task execution history with status, errors, duration |
| `taskAggregations` | Count of tasks by status |
| `scriptSearch` | Script repository |
| `tagSearch` | Tag definitions |
| `auditRecordSearch` | Audit trail of changes |

---

## Skills

### Engineer Skills

| Skill | What it does | Cadence |
|---|---|---|
| `alert-triage.md` | Disconnected agents + patch errors + critical vulns | Daily / on-call |
| `patch-status.md` | Patch compliance by status across customer or fleet | Weekly |
| `device-health-check.md` | Full device snapshot with metrics | Before any touch |
| `onboarding-checklist.md` | Gap analysis vs. MSP baseline | After agent deploy |
| `incident-summary.md` | Timeline from activity logs + task history | Post-incident |
| `stale-device-audit.md` | Disconnected devices + license cleanup | Monthly |
| `automation-policy-review.md` | Script task failures and coverage | Monthly |
| `eol-os-report.md` | EOL OS inventory with upgrade priority | Quarterly |
| `network-discovery-review.md` | Unmanaged and disconnected device gaps | Monthly |
| `site-dispatch-brief.md` | Pre-on-site briefing for field techs | Before every visit |
| `patch-approval-queue.md` | AVAILABLE and APPROVED patches awaiting action | Daily / patch Tuesday |
| `tag-devices.md` | Assign tags to devices in bulk (write operation) | On demand |
| `snapshot.md` | Save point-in-time fleet state for trend comparison | Weekly / before QBR |

### Executive / Sales Skills

| Skill | What it does | Cadence |
|---|---|---|
| `qbr-brief.md` | Fleet + patch + vuln summary for QBR | 1 week before QBR |
| `health-score.md` | Weighted 0–100 environment health score | Monthly |
| `renewal-risk-report.md` | Customers with deteriorating signals | Weekly |
| `upsell-opportunity-report.md` | Coverage gaps mapped to service offerings | Monthly |
| `executive-monthly-digest.md` | Full-fleet KPI memo for leadership | 1st of month |
| `new-prospect-baseline.md` | Pre-sales scoping assessment | Before SOW |
| `sla-compliance-report.md` | Patch/vuln age vs. SLA thresholds | Monthly |
| `fleet-growth-report.md` | Device count and composition by customer | Monthly |

---

## Org hierarchy

```
Partner
  └── ServiceOrganization
        └── Customer
              └── Site
                    └── Asset
```

`assetSearch` accepts `inOrganizations: [id]` — pass any level of the hierarchy. Pass a Customer ID to scope to one customer, a ServiceOrg ID for a whole service org.

## What's NOT in the GraphQL schema

- Alert acknowledgment / N-central monitoring alert history (use N-central UI reports)
- Backup job status (not yet exposed — infer from tags/custom properties)
- Ticket response times (live in your PSA)
- Month-over-month delta (current-state system — save snapshots to build trends)
