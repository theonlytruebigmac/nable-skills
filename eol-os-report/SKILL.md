---
description: Find devices running end-of-life or unsupported operating systems. Triggers on "EOL OS", "end of life OS", "unsupported OS", "OS currency report", "outdated operating systems".
---

# EOL OS Report

Identify devices running end-of-life operating systems using the N-able MCP.

## Query

```graphql
query EOLOsReport($after: String) {
  assetSearch(
    first: 500
    after: $after
    orderBy: [{ field: NAME, direction: ASC }]
  ) {
    totalCount
    pageInfo { endCursor hasNextPage }
    nodes {
      id
      name
      customer { name }
      serviceOrganization { name }
      operatingSystemInfo { name version type architecture role buildNumber featureRelease }
      chassis { types }
      agentConnection { status }
      tags { nodes { name } }
      systemInfo { hostname }
    }
  }
}
```

Validate with `validate` before `execute`. If `totalCount > 500`, paginate: re-run with `after: <pageInfo.endCursor>` and loop while `pageInfo.hasNextPage` is true, accumulating all `nodes` so the totals and percentages cover the whole fleet.

## EOL reference

Dates are vendor end-of-support milestones; classify each device by comparing to today.

| OS | Status |
|---|---|
| Windows 10 (all editions) | **EOL** — End of support Oct 2025 |
| Windows 7 / Server 2008 / 2008 R2 | **EOL** — Long past end of life |
| Windows Server 2012 / 2012 R2 | **EOL** — Extended support ended Oct 2023 |
| Windows Server 2016 | Mainstream ended Jan 2022; Extended until Jan 2027 |
| Windows Server 2019 | Mainstream ended Jan 2024; Extended until Jan 2029 |
| Windows Server 2022 | Mainstream until Oct 2026; Extended until Oct 2031 |
| Windows 11 | Supported |
| Windows Server 2025 | Supported |
| macOS older than 3 major versions | Vendor no longer patches |

## Classification

Parse `operatingSystemInfo.name`, `version` (and `buildNumber`), and `featureRelease` against the table above. Use `version` to distinguish server releases (Server 2016/2019/2022) and Windows 10 vs 11, since `name` alone does not differentiate them and `featureRelease` only applies to Windows 10/11:

| Category | Meaning |
|---|---|
| **EOL — No Support** | Past all support; no patches available |
| **EOL — Extended Only** | Past mainstream; security patches via ESU/Extended |
| **Approaching EOL** | Within 12 months of end of support (relative to today) |
| **Supported** | Actively patched by vendor |

## Output format

Open with risk summary:
- Total devices scanned
- Count per EOL category
- % of fleet on unsupported OS

List EOL and Approaching EOL devices grouped by customer:
- Device name | Customer | OS name + version | featureRelease | Chassis type | Agent status | EOL category

**Flag servers on fully-EOL OS as HIGH RISK** — no patches = active attack surface.

Close with a one-paragraph upgrade priority narrative for account manager use.
