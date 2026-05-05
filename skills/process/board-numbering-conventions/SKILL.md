---
name: GitHub Project Board Numbering Conventions
description: Release field for stack-ranked release buckets, WBS dot-decimal title prefix for within-release ordering, state:* labels for backlog/shovel-ready meta-status. Established 2026-05-03.
category: process
applicable_phases: [orchestration, planning]
applicable_stacks: [universal]
version: 1
---

GitHub project boards have three orthogonal organizing dimensions: **release**, **order within release**, and **status**. Mixing them in one field (the historic pattern of using `release:backlog` and `release:shovel-ready` as if they were releases) creates ambiguity and breaks alphabetical sort. This skill codifies the clean separation.

## Three orthogonal dimensions

```
Release field        ─→ which release bucket it ships in (single-select, manual option order)
Title prefix (WBS)   ─→ order WITHIN that release (dot-decimal hierarchy)
state:* label        ─→ meta-status (state:backlog, state:shovel-ready)
```

## Release field

Single-select field on the project. Options listed in **manual order** (drag-and-drop in UI or specify the order in `updateProjectV2Field` mutation). Sort on the Release column respects this order — no string-sort surprises.

**Convention for option names:**
- `R0` — pre-prod hardening
- `R1.1`, `R1.2`, ..., `R1.9` — current product layers (decimals for sub-releases)
- `R2`, `R3`, `R4`, ..., `R19` — future releases
- `shovel-ready`, `backlog`, `bucket-b` — kept as legacy options for closed items only; new items should use `state:*` labels instead

**Order in option array:** strict numeric ascending. R1.x decimals fit between R0 and R2.

**Adding a new option** wipes existing values via the option-ID recreation bug. See `feedback_project_field_options_recreate_ids` in agent memory and the snapshot+restore routine in `github-api-efficiency` skill.

## Title prefix (WBS)

The work-breakdown-structure dot-decimal scheme expresses **hierarchy and order within a release** in the title. The R-number is **NOT** in the title — that's the Release field's job.

**Format:**

| Depth | Pattern | Example |
|---|---|---|
| Top-level item / epic | `<n>` | `1 : Base-repo verification` |
| Sub-issue | `<n>.<m>` | `1.1 : base-repos : nextjs : verify builds locally` |
| Sub-sub-issue (rare) | `<n>.<m>.<k>` | `1.1.1 : ...` |
| Sub-sub-sub (very rare) | `<n>.<m>.<k>.<j>` | `1.1.1.1 : ...` |

**Letter mapping for legacy migrations:** `a → 1`, `b → 2`, `c → 3`, ... `z → 26`. Old `R3-1a` becomes title `1.1` in release `R3`.

**Sort behavior:** Alphabetical on titles inside a release works for single-digit segments. If a level reaches 10+ (e.g., 10 sub-issues of one epic), `1.10` sorts before `1.2`. Mitigate by:
- Most epics have ≤9 sub-issues; rarely an issue
- Or zero-pad just the affected level (`1.02`, `1.10` — only when needed)
- Or split the epic when it grows past 9 children

**Closing punctuation:** `<prefix> : <title>` with space-colon-space. Match existing style.

## State labels (`state:*`)

Items that have a release assigned but aren't actively in flight carry one of:

- **`state:backlog`** — gated, triggered, or unprioritized. Promotion needs a decision (e.g., "we'll do this when X happens"). Color: gray.
- **`state:shovel-ready`** — planned, awaiting prioritization. Ready to start once the queue clears. Color: yellow.

An item without either label is **active** in its release. Standard transitions:
- File at `state:backlog` → promoted to `state:shovel-ready` once a release has been decided → label removed once active

Multiple labels can apply (e.g., `state:backlog` + `needs:human-only`).

## Assignee model — two-account ownership

Items have **one owner** expressed via GitHub's native assignee field:

- **`@anthony-mansfield`** = human queue. Items requiring stakeholder decisions, manual testing, real-world setup (account creation, secret rotation), or judgment the agent doesn't have context for.
- **`@bytetalent-ai`** = agent queue. The "bytetalent.ai agent" as a coherent unit: agent personality + `.claude/settings.json` permissions + `bytetalent/docs/skills/` + `prompts/` + memory. Tasks here can be picked up by any session of any agent that loads that config.

**Why two accounts and not labels:**

- Native to GitHub: assignee is in every view (issue list, PR pages, search syntax, mobile app), not just the project board
- Notifications work right (each account gets pinged on its own queue)
- Permissions can be **scoped per account** — the agent account should have minimum-necessary permissions:
  - Write to feature branches, open PRs, read-only on `main` / `staging`
  - Cannot force-push, cannot bypass branch protection, cannot rotate secrets
  - This is defense-in-depth: even a compromised agent session has limited blast radius

**In-flight tracking:** assignee = **owner** (categorical, durable). Status field = **lifecycle** (Todo / In Progress / Done). Workers DON'T self-reassign on claim — they flip Status to In Progress and rely on the existing assignee.

**Filtering:**
- "Anthony's queue" = `assignee:anthony-mansfield is:open`
- "Agent queue ready to pick up" = `assignee:bytetalent-ai is:open` filtered to `Status: Todo`
- "Unassigned (needs triage)" = `no:assignee is:open`

## Other labels — what to use, what to skip

**Use:**
- **`needs:human-only`** — categorical "no agent should ever take this" (security-sensitive setup, real-account creation, manual UAT, decisions requiring stakeholder context). Typically paired with `@anthony-mansfield` assignee but the label is broader (any human, not just Anthony).
- **`area:*`** — architectural area for filtering (e.g., `area:db`, `area:api`, `area:platform`, `area:process`, `area:infra`, `area:docs`).
- **`cleanup`** — small follow-up issues to be batched in cleanup waves.

**Don't use:**
- **`epic`** — redundant with title prefix structure (`1` is an epic, `1.1` is a sub-issue) + sub-issue links. Removed 2026-05-03.
- **`lane:claude`** — historically used to mark autonomous-dispatchable. Replaced 2026-05-03 by the assignee model above.

## Putting it together — examples

### Currently active R1.6 (smoke + base verify)

| Issue | Release | Title | Labels |
|---|---|---|---|
| #220 | R1.6 | `1 : Base-repo verification` | (none — active) |
| #221 | R1.6 | `1.1 : base-repos : nextjs : verify builds locally + IDE-runnable` | (none) |
| #222 | R1.6 | `1.2 : base-repos : astro : verify builds locally + IDE-runnable` | (none) |
| #226 | R1.6 | `2 : Per-phase wizard walk-through ...` | (none) |
| #375 | R1.6 | `4 : smoke test — run a Bytetalent self-project end-to-end ...` | (none) |

### Future release R3 (Resend setup, planned)

| Issue | Release | Title | Labels |
|---|---|---|---|
| #364 | R3 | `1 : Resend : create account + verify domain ...` | `state:backlog` |
| #365 | R3 | `1.1 : bt-ai-web : audit + complete Resend integration` | `state:backlog` |
| #366 | R3 | `1.2 : bytetalent.ai : add Resend ...` | `state:backlog` |

### Standalone backlog (no roadmap release decided)

| Issue | Release | Title | Labels |
|---|---|---|---|
| #428 | R13 | `1 : skills : ui : consultant-facing skill management` | `state:backlog` |
| #219 | R16 | `1 : Rename bt-ai-web → app-flow-web` | `state:backlog`, `needs:human-only` |

## Migration history

Original scheme (pre-2026-05-03) used title prefixes like `R3-1`, `R3-1a` with `release:backlog` to capture both bucket and order. This collapsed two dimensions into one and broke alphabetical sort (`R10-1` sorted before `R2-1`). On 2026-05-03 we migrated to the current scheme:
- Renamed all 101 open items
- Added 18 release options (R1.9, R2-R19) via `updateProjectV2Field` (recreated all option IDs)
- Restored 202 release values (101 open + 101 closed) post-recreation
- Created `state:backlog` and `state:shovel-ready` labels and applied to 64 items

## Operating notes for agents

When creating an issue:
1. Decide which release it belongs in → set Release field
2. Decide its order within that release → set title prefix (next free number, or `<parent>.<sub>` if it's a sub-issue)
3. If it's not yet active, apply `state:backlog` or `state:shovel-ready`
4. Don't put R-number in the title
5. Don't apply `epic` or `lane:claude` labels (deprecated)

When restructuring (renumbering or moving an item):
- Title prefix: `gh issue edit <n> --title "..."`  (REST, separate quota from GraphQL)
- Release field: GraphQL `updateProjectV2ItemFieldValue` (batchable via aliases)
- Labels: `gh issue edit <n> --add-label state:backlog` or `--remove-label state:shovel-ready`

When adding a new release option:
- Use the snapshot + restore routine (see `feedback_project_field_options_recreate_ids` memory)
- Option-ID recreation is unavoidable; the routine makes it safe

## See also

- `github-api-efficiency/SKILL.md` — bundle, cache, batch
- `lessons-learned-discipline/SKILL.md` — where lessons go (this scheme is the resolution of one)
