---
description: Author a token-minimal, convention-correct skill for this N-able/N-central library. Triggers on "create a skill", "new skill", "write a skill", "build a skill", "scaffold a skill", "add a skill for [task]".
---

# Create Skill

North star: **the smallest `SKILL.md` that still runs end-to-end.** Tight, not terse — keep the runnable core verbatim; cut or defer everything else.

## Token-cost model — design for the tier

| Tier | Loads | Cost | Rule |
|---|---|---|---|
| `description` | always, for **every** skill, every turn | highest per char (summed across the whole library) | shortest that still discriminates |
| `SKILL.md` body | once, when this skill is picked | moderate | runnable/decision content only; bake in defaults |
| `docs/` files, scripts | only when read / run (a script runs without loading its code) | ~zero until used | large, conditional, or shared content goes here |

Before adding a line, ask: **is it runnable or decision-bearing, and unique to this skill?** If not, cut it or move it down a tier.

## Inline vs. defer (this is the real token lever)

Linking is not free — it makes the agent read a whole `docs/` file at run time. Pick the cheaper tier:

- **Inline** a *short* block that's needed on almost every run (e.g. the one-line "join across MCPs by name, never id"). A one-liner beats forcing a big-doc read.
- **Link to `docs/`** when content is large, only sometimes needed, or would be copied verbatim into 3+ skills (rule of three) — then it lives once and loads only when relevant.
- **Ship a script** for deterministic, repeated-verbatim code (validation, formatting, paging) — call it instead of regenerating identical code (and its output tokens) every run.

Heavy shared reference already lives in `docs/` — link it, don't re-explain:

| Need | Link |
|---|---|
| Which MCP for the task | [which-mcp.md](../docs/which-mcp.md) |
| Cross-MCP / two-ID-space join | [#cross-mcp-correlation](../docs/ncentral-mcp-reference.md#cross-mcp-correlation) |
| Org tree + `orgUnitId`/`soId`/`customerId` | [#org-hierarchy--ids](../docs/ncentral-mcp-reference.md#org-hierarchy--ids) |
| Call conventions (`all:true`, `pageSize` max 200) | [#global-call-conventions](../docs/ncentral-mcp-reference.md#global-call-conventions) |
| Full tool catalog + per-tool quirks | [ncentral-mcp-reference.md](../docs/ncentral-mcp-reference.md) |

> Today most skills inline these blocks. Linking large/shared ones is the biggest available saving — prefer it for new skills, but don't bloat a skill by linking a one-liner.

## Process

1. **Infer defaults, don't interview.** Defaults: scope = whole fleet, output = table, mode = read-only/preview. Ask at most one question, only when it changes the output — each round-trip reloads context.
2. **Draft `SKILL.md`** from the template: numbered steps, real GraphQL/tool calls. Pick the backend via [which-mcp.md](../docs/which-mcp.md).
3. **Run the checklist**, then confirm with the user.

## Description (always-resident — spend chars wisely)

Short **and** discriminative. Lead with the unique capability + the entity/API the skill owns, then 3–6 exact trigger phrases. Third person, no filler ("Helps with", "This skill"). An under-discriminative description mis-triggers and loads the **wrong** whole body — pure waste.

- Target ~175–340 chars (1024 hard cap); median in this library is ~230.
- ✅ `Report patch compliance and installation status for a customer or the fleet. Triggers on "patch status", "missing patches", "patch report for [client]".`
- ❌ `Helps with patches.`

## Template

````md
---
description: <capability + entity/API it owns>. Triggers on "<phrase>", "<phrase>", "<phrase>".
---

# <Skill Name>

<One line: what it queries, via which MCP.>

## Step 1 — answer with the smallest query

```graphql
query { ... totalCount }   # prefer aggregations/counts when a number answers the question
```

REST equivalent — single page unless asked:
`{ "tool": "list_devices_by_org_unit", "args": { "orgUnitId": 1234 } }`  # add "all": true only on request

## Step 2 — detail (only if needed)

```graphql
query { nodes(first: 100) { ... } }   # cap results; request only the fields you use
```

Validate every GraphQL query before execute. Preview + confirm before any write.

## Output format

<smallest table that answers the question + a one-line summary; widen only if asked.>
````

## Review checklist

- [ ] One `SKILL.md`; folder name is the skill (**no `name:` field**); frontmatter is `description:` only.
- [ ] Description: capability first, then `Triggers on "..."` with concrete phrases; ~175–340 chars, discriminative.
- [ ] Large / conditional / 3+-skill-shared content is linked to `docs/`, not copied; short always-needed blocks may inline.
- [ ] Deterministic repeated code is a script, not regenerated inline.
- [ ] Numbered, runnable steps; counts/aggregations before node lists when they suffice; `first:` cap or single REST page by default; only fields you use.
- [ ] Writes preview + confirm; GraphQL validates before execute.
- [ ] No time-sensitive info; consistent terminology; links resolve and are one level deep.
- [ ] Leanest body that still runs (library median 99 lines, range 67–136 — shorter is better).
