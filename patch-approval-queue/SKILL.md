---
description: Show pending patch approvals across a customer or the full fleet. Triggers on "patch approval queue", "what patches need approval", "patches to approve for [client]", "patch review", "pending patches".
---

# Patch Approval Queue

Surface patches awaiting approval and those approved but not yet installed using the N-able MCP.

## Step 1 — AVAILABLE patches (need review/approval)

```graphql
query PatchApprovalQueue($customerId: ID!) {
  patchInstallationSearch(
    first: 200
    where: { installationStatus: { equals: AVAILABLE } }
    inOrganizations: [$customerId]
    orderBy: [{ field: PATCH_SEVERITY, direction: ASC }]
  ) {
    totalCount
    nodes {
      asset { id name agentConnection { status } }
      customer { name }
      patch {
        name
        severity
        classification
        isRebootRequired
        publishedOn
      }
      lastUpdatedAt
    }
  }
}
```

Omit `inOrganizations` to run across all customers.

## Step 2 — APPROVED patches (scheduled, not yet installed)

```graphql
query PatchApprovedPending($customerId: ID!) {
  patchInstallationSearch(
    first: 100
    where: { installationStatus: { equals: APPROVED } }
    inOrganizations: [$customerId]
    orderBy: [{ field: PATCH_SEVERITY, direction: ASC }]
  ) {
    totalCount
    nodes {
      asset { id name }
      customer { name }
      patch { name severity classification isRebootRequired }
      installOptions {
        nextInstallAt
        installationSchedule { startOn runAt }
      }
      lastUpdatedAt
    }
  }
}
```

## Step 3 — DECLINED patches (visibility)

```graphql
query PatchDeclined($customerId: ID!) {
  patchInstallationSearch(
    first: 50
    where: { installationStatus: { equals: DECLINED } }
    inOrganizations: [$customerId]
    orderBy: [{ field: PATCH_SEVERITY, direction: ASC }]
  ) {
    totalCount
    nodes {
      asset { name }
      customer { name }
      patch { name severity classification publishedOn }
      lastUpdatedAt
    }
  }
}
```

Validate all queries before executing.

## Approval priority order

Process AVAILABLE patches in this order:

| Priority | Severity | Classification | Action |
|---|---|---|---|
| 1 | CRITICAL | Security | Approve immediately |
| 2 | IMPORTANT | Security | Approve within SLA window |
| 3 | CRITICAL/IMPORTANT | Non-security | Review with customer |
| 4 | MODERATE/LOW | Any | Batch for next maintenance window |

Flag patches where `isRebootRequired: true` — coordinate maintenance window with customer before approving.

Flag patches published more than 30 days ago that are still AVAILABLE — these represent approval backlog risk.

## Output format

**Patch Approval Queue — [Customer or All] — [Date]**

**Needs Approval (AVAILABLE): X patches across X devices**

Priority queue (approve these first):
| Severity | Patch Name | Devices Affected | Published | Reboot? |
|---|---|---|---|---|
| CRITICAL | ... | X | date | Yes/No |
| IMPORTANT | ... | X | date | Yes/No |

Backlog (>30 days old, still AVAILABLE):
- List patches with publish date and affected device count

**Approved / Pending Install: X patches**
- Next scheduled install: [nextInstallAt or startOn+runAt]
- List by customer, flag any with no schedule set

**Declined: X patches**
- List CRITICAL/IMPORTANT declined patches for re-review

**Reboot impact summary**
- X patches require reboot across X devices
- Customers with pending reboot-required approvals: [list]

Note: Patch approval itself is done in the N-central UI or via N-able automation policies — this skill surfaces the queue for review decisions only.
