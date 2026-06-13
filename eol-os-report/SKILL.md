---
description: Find devices running end-of-life or unsupported operating systems. Triggers on "EOL OS", "end of life OS", "unsupported OS", "OS currency report", "outdated operating systems".
---

# EOL OS Report

Identify devices running end-of-life operating systems using the N-able MCP.

## Query

```graphql
query EOLOsReport {
  assetSearch(
    first: 500
    orderBy: [{ field: NAME, direction: ASC }]
  ) {
    totalCount
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

Validate before executing. Paginate using `after` cursor if `totalCount > 500`.

## EOL reference (as of June 2026)

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

Parse `operatingSystemInfo.name` and `featureRelease` against the table above:

| Category | Meaning |
|---|---|
| **EOL — No Support** | Past all support; no patches available |
| **EOL — Extended Only** | Past mainstream; security patches via ESU/Extended |
| **Approaching EOL** | Within 12 months of end of support |
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
