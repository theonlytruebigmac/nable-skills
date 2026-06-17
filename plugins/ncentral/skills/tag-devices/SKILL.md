---
description: Assign tags to managed devices via the N-able MCP (GraphQL). Triggers on "tag [devices/customer]", "add tag to [device]", "label these devices", "tag all servers for [client]", "apply tag [name] to [filter]".
---

# Tag Devices

Find assets and assign tags to them using the N-able MCP. Always preview before tagging — this is a write operation.

## Step 1 — Find available tags

```graphql
query TagSearch($name: String) {
  tagSearch(
    first: 50
    where: { name: { contains: $name } }
    orderBy: [{ field: NAME, direction: ASC }]
  ) {
    totalCount
    nodes {
      id
      name
      category
      description
      organization { id name }
      hasPolicies
    }
  }
}
```

Omit the `where` clause to list all tags. If the desired tag doesn't exist, inform the user — tag creation is not available via MCP (create tags in the N-central UI, then re-run).

## Step 2 — Find target assets

Scope by customer, name pattern, OS type, chassis type, or agent status as needed:

```graphql
query AssetsForTagging($customerId: ID!, $namePattern: String) {
  assetSearch(
    first: 200
    inOrganizations: [$customerId]
    where: { name: { contains: $namePattern } }
  ) {
    totalCount
    nodes {
      id
      name
      customer { name }
      tags { nodes { id name } }
      operatingSystemInfo { name type }
      chassis { types }
      agentConnection { status }
    }
  }
}
```

Adjust the `where` clause to match the user's intent:
- All servers: filter by `chassis.types` containing server-class types, or OS name containing "Server"
- By OS type: `operatingSystemInfo: { type: { equals: WINDOWS } }`
- Disconnected only: `agentConnection: { status: { equals: DISCONNECTED } }`
- Omit `where` and `inOrganizations` to target the full fleet

## Step 3 — Preview and confirm

Before calling the tag tool, show the user:

```
Tag to apply: "[tag name]" (ID: xxx)
Devices to be tagged (X total):
  - Device A | Customer | OS | Currently tagged: [existing tags]
  - Device B | Customer | OS | Currently tagged: [existing tags]

Devices already carrying this tag (will be skipped): X
```

**Always ask for explicit confirmation before proceeding.** This is a production change.

## Step 4 — Assign tags

Once confirmed, call the `assign_tags_to_assets` tool:

- `tagIds`: list of tag IDs from Step 1
- `assetIds`: list of asset IDs from Step 2 (exclude assets that already have the tag)

The tool applies the tags in bulk. After it completes, re-query a sample of the assets to confirm the tags appear on `tags.nodes`.

## Step 5 — Verify

```graphql
query VerifyTagging($assetId: ID!) {
  asset(id: $assetId) {
    name
    tags { nodes { id name category } }
  }
}
```

Run this on 2–3 of the tagged assets to confirm the operation succeeded.

Validate every GraphQL query with `validate` before running it with `execute`.

## Notes

- Tags are scoped to an organization — a tag created under Customer A cannot be assigned to Customer B's assets. If `tagSearch` returns no results for a customer's assets, check the `organization` field on the tag.
- `hasPolicies: true` means the tag controls automation policies — flag this to the user before applying at scale, as tagging may trigger automated workflows.
- Tagging does not remove existing tags. To replace tags, the existing tags must be removed in the N-central UI (tag removal is not available via MCP).
