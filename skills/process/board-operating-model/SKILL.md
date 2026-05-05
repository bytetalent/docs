---
name: GitHub Project Board Operating Model
description: Canonical reference for the project board — release model (main R-numbers + half-step process track), priority + status semantics, title convention v2 (area : feature : description), dependency model (sub-issue links + Blocks/Blocked-by), state labels, two-account assignee model. Codifies operational decisions so they don't drift or get re-litigated.
category: process
applicable_phases: [orchestration, planning]
applicable_stacks: [universal]
version: 3
---

GitHub project boards have orthogonal organizing dimensions: **release**, **priority**, **status**, **dependencies**, **meta-status**, **ownership**, **area**. Each has its own field/mechanism — no encoding multiple dimensions into a single title prefix.

The previous version of this skill (v1, 2026-05-03) used a WBS dot-decimal title prefix scheme to express order + hierarchy in titles. That convention is **deprecated** as of 2026-05-04. GitHub's native sub-issue links + Blocks/Blocked-by relationships handle hierarchy and dependency without title duplication.

This skill (v3, expanded 2026-05-04 from `board-numbering-conventions`) covers the full board operating model — not just the title format. Read it once when joining a new agent session; reference specific sections during grooming, dispatch, or restructure passes.

## Title format — `<area> : <feature> : <description>`

Three fields, separated by space-colon-space:

| Field | Required | Source | Example |
|---|---|---|---|
| `<area>` | yes | matches an `area:*` label (no `area:` prefix in the title itself) | `frontend`, `process`, `data` |
| `<feature>` | yes | the part of the system; lowercase-hyphen-case | `project-create`, `phase-gen`, `connections-page`, `onboarding-wizard`, `branch-hygiene` |
| `<description>` | yes | concise, action-oriented summary | `update PRD view, step 3` |

**Examples**:

```
frontend : project-create : update PRD view, step 3
process : grooming : codify board title convention
infra : auth-google-oauth : register Clerk client
data : project-phases : add 'failed' value to status enum
docs : guides : strip release prefixes from titles
platform : onboarding-model : self-client auto-creation in getOrCreateAccount + backfill
```

For epics, the `<feature>` is broader and the description is more thematic:

```
platform : onboarding-model : solo creator + progressive UX + wizard wiring
infra : pre-prod-hardening : auth + IP allowlist + Cloudflare + crons
process : docs-reorg : skills + guides into category subdirs
```

## Product-area prefix on `<feature>` keys

The product divides into two complementary areas. When a feature clearly belongs to one of them, **prefix the `<feature>` key** with `agency-` or `project-`. This makes the polish-vs-core distinction visible in titles and filterable on the board.

| Product area | What it is | Examples |
|---|---|---|
| **Agency features** | Workflows for agency users themselves — onboarding, dashboard, clients management, invitations, billing, connections, settings | `agency-onboarding`, `agency-dashboard`, `agency-billing`, `agency-invitations`, `agency-connections`, `agency-clients` |
| **Project deliverables** | What a project produces — the 5-phase wizard, phase outputs, templates, code-gen, deliverable portal | `project-wizard`, `project-prd-phase`, `project-design-phase`, `project-codegen`, `project-deliverable-portal`, `project-templates` |

**When NOT to prefix** (no product-area home — cross-cutting, infra, or process):
- `auth-google-oauth`, `pre-prod-hardening`, `edge-protection` (infra)
- `branch-hygiene`, `board-grooming`, `repo-rename` (process)
- `template-system`, `manifest-schema` (platform-level, not specific to agency or project surface)
- `clerk-auth`, `vault-secrets` (security infrastructure)

**Why prefix**:
- Polish vs core: agency-* items typically read as workflow polish; project-* items typically read as the core demo path. The prefix surfaces this naturally without a separate label.
- Filterability: scan the board, immediately see which items are agency UX vs project pipeline.
- Mental model reinforcement: "show me working software" = project-* items shipping; "agency UX" = agency-* items.

**When ambiguous**: pick the dominant side. If a feature is "agency UX in service of project creation" (e.g., the project wizard's client picker), it's `project-wizard` because the dominant deliverable is the project. If it's "project signal surfaced into agency dashboard" (e.g., recent-projects list), it's `agency-dashboard` because the dominant home is the dashboard surface.

## Area set (12)

Aligns with `area:*` labels on the repo. Tight boundaries:

| Area | What goes here |
|---|---|
| `infra` | Third-party service setup + config (Resend account, Vercel project, GitHub PAT, OAuth client, Cloudflare WAF, DNS, Supabase IP allowlist, Clerk webhook signing keys) |
| `process` | Internal workflow + maintenance (board grooming, branch hygiene, walkthroughs, agent orchestration, codification, lessons-learned, weekly hygiene audit — even when implemented as a GitHub Action) |
| `data` | DB schema, migrations, queries, RLS policies |
| `api` | API endpoints, route handlers, webhook handlers |
| `frontend` | React components, pages, UI flows, layout/sizing |
| `design` | Design tokens, UX patterns, design tools (Pencil) |
| `docs` | Markdown documentation (guides, README, CLAUDE.md, product docs, strategic docs) |
| `platform` | Templates, bases, scaffolding, cross-cutting platform code, manifest schemas |
| `prompts` | Phase prompts (PRD, brand-kit, design, arch, code-gen) |
| `skills` | Skill authoring (universal/base/template/product), skill metadata, skill consolidation |
| `tests` | Test files + test infrastructure |
| `security` | Auth, secrets handling, prompt-injection review, RLS bypass review |

Notes on boundaries:
- **`infra` vs `process`**: `infra` is *external* services + config. `process` is *internal* workflow. The branch-hygiene weekly audit Action lives in `.github/workflows/` (CI infra) but its WORK is `process` (auditing branches). Categorize by what the work does, not where it runs.
- **`process` vs `docs`**: `process` is workflow rules + their codification. `docs` is human-readable narrative. A skill update is `process` (the rule changed); a guide rewrite is `docs` (the prose changed).
- **`frontend` vs `design`**: `design` is visual decisions + tokens; `frontend` is implementation. A new color in the palette is `design`; applying it across components is `frontend`.

## Release field (single-select on the project)

Captures **which release bucket** an item ships in. Single source of truth — **not encoded in titles**.

**Convention for option names**:
- `R0` — pre-prod hardening
- `R1.1`, `R1.2`, ..., `R1.9` — current product layers (decimals for sub-releases)
- `R1.5.5`, `R1.6.5`, `R1.7.5`, `R1.8.5` — half-step releases for the parallel **process / docs / platform-improvement track** (see below)
- `R2`, `R3`, `R4`, ..., `R19` — future releases
- `shovel-ready`, `backlog`, `bucket-b` — legacy options for closed items only; new items use `state:*` labels

**Order in option array**: strict numeric ascending. R1.x decimals fit between R0 and R2; half-step `R1.x.5` fits between `R1.x` and `R1.(x+1)`.

**Adding a new option** wipes existing values via the option-ID recreation bug. See `feedback_project_field_options_recreate_ids` in agent memory and the snapshot+restore routine in `github-api-efficiency` skill.

### Half-step releases — parallel process / doc track

Main R-numbers (R1.5, R1.6, R1.7, ...) are the **product feature track** — what's visible to users and demos. Half-step releases (R1.5.5, R1.6.5, R1.7.5, R1.8.5) run **alongside** as a parallel **process / docs / platform-improvement track** that doesn't gate the product cadence.

**What goes in a half-step release**:
- Skill authoring + skill updates (`area:skills`, `area:process`)
- Documentation guides + audits (`area:docs`)
- Board grooming, branch hygiene, lessons-learned codification (`area:process`)
- Internal tooling, scripts, automation (`area:infra` when the work is internal-facing)
- Platform extensions to bases / templates that don't ship a user-facing feature (`area:platform`)
- Any work whose stakeholder is "agent / consultant / future-self" rather than "end user"

**What does NOT go in a half-step release**:
- Anything on the demo path or release-cut path — that's the main R-number
- Bugfixes for shipping features — main R-number
- New user-facing features — main R-number

**Why the split matters**:
- Lets process/doc work proceed in parallel without competing with feature priority
- Keeps a clean answer to "what's in R1.6?" — only feature work
- Gives `area:process` and `area:docs` items their own home so they don't slip when feature releases are tight
- Half-step releases can bundle freely (10–20 items each) without diluting feature focus

**Sequencing**: a half-step release nominally targets the timing of the next main R-number after it (R1.5.5 closes around the same time as R1.6). But it's **not gated** — items can move between half-step releases as priorities shift, and a half-step release can stay open after its companion R-number ships.

## Priority field (single-select on the project)

Captures **when within a release** an item should run. Three options:

- **`now`** — actively claimable. Top of the queue. Workers picking up the next task scan `now` items first.
- **`next`** — queued behind `now`. Promoted to `now` when something lands or scope shifts. Default for items added to a release without an immediate slot.
- **`later`** — release-tagged but explicitly de-prioritized within the release. Won't be picked up unless `now`/`next` is empty or priorities change.

**Stack-ranking within a priority bucket**: the project board UI's vertical order acts as a tie-breaker. Within `now`, items at the top of the column run before items below. Drag-and-drop in the board UI is the canonical way to re-rank — no priority sub-numbers, no title encoding.

**When to use `later`**: the item belongs in this release (it's in scope, the work fits the release's theme), but it's clearly going to slip if the release is tight. Better than re-tagging to a different release because it preserves the scope decision.

**Don't use Priority for sequencing dependencies**: if A blocks B, that's a `Blocks/Blocked-by` relationship (see Hierarchy section), not "A=now, B=next." Priority answers *when* the human/agent should pick it up; dependencies answer *what must finish first*.

## Status field (single-select on the project)

Captures **lifecycle phase** of an item within a release:

- **Todo** — not started. Default for new items. Workers and humans pick from `Todo`.
- **In Progress** — actively being worked. Set when a worker is dispatched or a human starts the task. Multiple items can be `In Progress` if multiple agents/humans are working in parallel.
- **Done** — closed. Auto-set when the issue closes. Don't manually set without closing the issue.

**Transitions**:
- `Todo → In Progress` — when work starts. **Workers flip Status to `In Progress` when they begin** — they don't reassign the issue (assignee is durable categorical ownership; Status is lifecycle).
- `In Progress → Todo` — when blocked or paused. Add a comment explaining why; consider demoting Priority to `later`.
- `Anything → Done` — close the issue; Status auto-syncs.

**Status vs `state:*` labels — different concerns**:
- **Status** = lifecycle of an active item (where in the work-cycle it is)
- **`state:backlog` / `state:shovel-ready`** = meta-status for items not yet active (pre-prioritization gate)

An item is either active (no `state:*` label, has a Status of Todo/In Progress) or non-active (`state:backlog` or `state:shovel-ready`, Status typically Todo).

**Don't use Status for "this needs review"** — that's a PR concern, not a board concern. The board reflects whether the *work* is in progress, not whether the *PR* is open.

## Hierarchy + dependencies — use native links, not titles

### Sub-issue links (parent-child)

GitHub natively supports sub-issue relationships via `addSubIssue` GraphQL mutation. Use this for **all parent-child relationships** instead of encoding hierarchy in the title.

A sub-issue:
- Shows up under its parent's "Sub-issues" panel
- Is reachable from the parent in the issue UI
- Aggregates parent progress (X of N done) automatically
- Doesn't need its title to encode "1.1" or "5.2" — the relationship is in the graph

**Filing pattern**:
```graphql
mutation {
  addSubIssue(input: {issueId: "<parent-id>", subIssueId: "<child-id>"}) { ... }
}
```

Batch via aliases when filing multiple children of an epic.

### Blocks / Blocked-by (non-parent dependencies)

For dependencies that aren't parent-child (e.g., "this needs the auth wave to land first"), use GitHub's native `Blocks` / `Blocked by` issue references. Add a comment on the dependent issue:

```
Blocked by #123 (auth wave needs to land first)
```

GitHub renders this as a structured cross-reference. The dependency graph is queryable.

**Don't use**: title prefixes like `(blocked)` or `(after #123)` — drift-prone, not queryable.

## State labels (`state:*`)

Items that have a release assigned but aren't actively in flight carry one of:

- **`state:backlog`** — gated, triggered, or unprioritized. Promotion needs a decision (e.g., "we'll do this when X happens"). Color: gray.
- **`state:shovel-ready`** — planned, awaiting prioritization. Ready to start once the queue clears. Color: yellow.

An item without either label is **active** in its release. Standard transitions: filed at `state:backlog` → promoted to `state:shovel-ready` → label removed once active.

Multiple labels can apply (e.g., `state:backlog` + `needs:human-only`).

## Assignee model — two-account ownership

Items have **one owner** expressed via GitHub's native assignee field:

- **`@anthony-mansfield`** = human queue. Items requiring stakeholder decisions, manual testing, real-world setup (account creation, secret rotation), or judgment the agent doesn't have context for.
- **`@bytetalent-ai`** = agent queue. The "bytetalent.ai agent" as a coherent unit: agent personality + `.claude/settings.json` permissions + skills + prompts + memory. Tasks here can be picked up by any session of any agent that loads that config.

**Why two accounts and not labels**:
- Native to GitHub: assignee is in every view (issue list, PR pages, search syntax, mobile app), not just the project board
- Notifications work right (each account gets pinged on its own queue)
- Permissions can be **scoped per account** — the agent account should have minimum-necessary permissions:
  - Write to feature branches, open PRs, read-only on `main` / `staging`
  - Cannot force-push, cannot bypass branch protection, cannot rotate secrets
  - Defense-in-depth: even a compromised agent session has limited blast radius

**In-flight tracking**: assignee = **owner** (categorical, durable). Status field = **lifecycle** (Todo / In Progress / Done). Workers DON'T self-reassign on claim — they flip Status to In Progress and rely on the existing assignee.

**Filtering**:
- "Anthony's queue" = `assignee:anthony-mansfield is:open`
- "Agent queue ready to pick up" = `assignee:bytetalent-ai is:open` filtered to `Status: Todo`
- "Unassigned (needs triage)" = `no:assignee is:open`

## Other labels — what to use, what to skip

**Use**:
- **`needs:human-only`** — categorical "no agent should ever take this" (security-sensitive setup, real-account creation, manual UAT, decisions requiring stakeholder context). Typically paired with `@anthony-mansfield` assignee but the label is broader.
- **`area:*`** — architectural area for filtering (set listed above). Each item gets ONE area label matching the title's first field.
- **`cleanup`** — small follow-up issues to be batched in cleanup waves.

**Don't use** (deprecated):
- **`epic`** — redundant with sub-issue links. Removed 2026-05-03.
- **`lane:claude`** — historically used to mark autonomous-dispatchable. Replaced 2026-05-03 by the assignee model.
- Title prefixes like `[infra]`, `[cleanup]`, `[r1.6]`, `R1.6-3.4`, `5.2`, etc. — duplicates the area label / Release field / sub-issue link.

## Putting it together — examples

### Onboarding model epic + children (R1.6, all priority:now)

| Issue | Release | Title | Labels |
|---|---|---|---|
| #533 (epic) | R1.6 | `platform : onboarding-model : solo creator + progressive UX + wizard wiring` | `area:platform` |
| #534 (sub-issue of #533, closed) | R1.6 | `platform : onboarding-model : self-client auto-creation in getOrCreateAccount + backfill` | `area:platform` |
| #535 (sub-issue of #533) | R1.6 | `frontend : project-wizard : auto-resolve to self-client when solo` | `area:frontend` |
| #537 (sub-issue of #533) | R1.6 | `platform : onboarding-wizard : wire DB writes on completion` | `area:platform` |

Note: each child has its own `<area>` field — they vary based on what the work touches, even within an epic.

### Pre-prod hardening epic + children (R1.6, infra)

| Issue | Release | Title | Labels |
|---|---|---|---|
| #258 (epic) | R1.6 | `infra : pre-prod-hardening : auth + IP allowlist + Cloudflare + crons` | `area:infra` |
| #80 (sub-issue) | R1.6 | `infra : auth-google-oauth : register Clerk client` | `area:infra`, `needs:human-only` |
| #90 (sub-issue) | R1.6 | `infra : edge-protection : Cloudflare in front of Vercel` | `area:infra`, `needs:human-only` |
| #393 (sub-issue) | R1.6 | `infra : pencil-worker-cron : verify config on staging + prod` | `area:infra` |

### Standalone process item (no epic)

| Issue | Release | Title | Labels |
|---|---|---|---|
| #585 (closed) | R1.6 | `process : branch-hygiene : weekly audit GitHub Action` | `area:process` |
| #623 | R1.5 | `infra : github-pat : add BYTETALENT_REPOS_TOKEN secret + enable cross-repo audit` | `area:infra`, `cleanup`, `needs:human-only` |

### Standalone backlog (no roadmap release decided)

| Issue | Release | Title | Labels |
|---|---|---|---|
| #428 | R13 | `frontend : skills-management : consultant-facing skill UI` | `area:frontend`, `state:backlog` |
| #219 | R16 | `process : repo-rename : bt-ai-web → app-flow-web + path rehome + docs scrub` | `area:process`, `state:backlog`, `needs:human-only` |

## Migration history

**v1 (2026-05-03)** — WBS dot-decimal title prefix:
- Format: `<n>.<m> : <slug> : <description>` (e.g., `1.1 : base-repos : nextjs : verify builds`)
- Release in board field (already correct)
- Letter mapping for legacy: `a→1`, `b→2`, etc.
- Skill name: `board-numbering-conventions`

**v2 (2026-05-04)** — area-feature-description title format:
- Format: `<area> : <feature> : <description>` (e.g., `platform : base-repos-nextjs : verify builds locally`)
- Hierarchy moves from title to sub-issue links
- Order within release: managed by Priority field (`now` / `next` / `later`), not title
- Release stays in board field
- One `<area>` label per item, mirroring the title
- Skill name: still `board-numbering-conventions` (transitional)

**v3 (2026-05-04)** — full operating-model expansion (this version):
- Skill renamed: `board-numbering-conventions` → `board-operating-model` (the rules cover more than numbering)
- Half-step release model added (`R1.5.5` / `R1.6.5` / etc. — parallel process/doc track)
- Priority semantics codified (`now` / `next` / `later` + UI stack-rank)
- Status semantics codified (`Todo` / `In Progress` / `Done` lifecycle, distinct from `state:*` labels)

The transition: existing items keep their old titles until a re-titling pass; new items use v2 immediately. The re-titling pass is itself a `process : board-grooming : <description>` task.

## Operating notes for agents

When **creating** an issue:
1. Decide which `<area>` matches the work
2. Pick a `<feature>` key (lowercase-hyphen-case, matches an existing pattern if one exists, or coin a new one)
3. Write a concise `<description>`
4. Set Release field — main R-number for product feature work, half-step (R1.x.5) for process/doc/platform work
5. Set Priority (`now` / `next` / `later`)
6. Set Status (`Todo` for new items)
7. Apply `area:<area>` label, plus `cleanup` / `needs:human-only` / `state:*` as relevant
8. Assign — `@anthony-mansfield` for human-required work, `@bytetalent-ai` for agent-claimable
9. Link as sub-issue of parent epic (if any) via `addSubIssue`
10. Don't put R-number, parent number, or `[prefix]` in the title

When **restructuring** (renaming or moving an item):
- Title: `gh issue edit <n> --title "..."` (REST, separate quota from GraphQL)
- Release / Priority / Status field: GraphQL `updateProjectV2ItemFieldValue` (batchable via aliases)
- Labels: `gh issue edit <n> --add-label area:frontend` or `--remove-label state:shovel-ready`
- Assignee: `gh issue edit <n> --add-assignee bytetalent-ai`
- Sub-issue link: GraphQL `addSubIssue` mutation

When **picking the next task**:
1. Filter: `Status: Todo`, `Priority: now`, assignee = your account
2. Within that set, top-of-column items run first (UI rank)
3. Flip Status to `In Progress` when starting (don't reassign)
4. Status auto-flips to `Done` on issue close

## See also

- `backlog-grooming/SKILL.md` — assessment process for ticket completeness, dependencies, design requirements (uses the rules in this skill)
- `worker-dispatch-orchestration/SKILL.md` — pre-dispatch research + brief authoring; consumes the `Status` and `Priority` fields documented here
- `github-api-efficiency/SKILL.md` — bundle, cache, batch — essential for batch re-titling and field-update passes
- `lessons-learned-discipline/SKILL.md` — where lessons go (this scheme is the resolution of one)
