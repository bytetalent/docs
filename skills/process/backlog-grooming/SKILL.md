---
name: Backlog Grooming
description: How to keep a GitHub Projects v2 backlog organized AND assess each ticket's readiness for dispatch. Covers required fields per ticket, body structure, the eight-dimension completeness assessment (title / fields / body / human deps / relationships / design / sizing / fit-for-dispatch), views, workflows, periodic hygiene, anti-patterns. Pairs with `board-operating-model` which codifies the rules this skill enforces.
category: process
applicable_phases: [orchestration, planning, ongoing]
applicable_stacks: [universal]
version: 2
---

This skill captures the conventions for keeping a GitHub Projects v2 backlog organized, navigable, and actionable across a multi-agent / human team — AND for **assessing whether any individual ticket is ready to dispatch**. Pairs with `board-operating-model/SKILL.md` (the rules: title format, fields, labels, hierarchy, assignee model) which this skill enforces.

## Core principle

**The backlog reflects current intent and recent state, not history.** Every open item should be something we plan to do; every closed item is recorded but invisible by default. A stale-but-open item is worse than no item at all because it lies about what's in flight.

A second principle, closely related: **a ticket that isn't ready to dispatch isn't really on the backlog yet.** It's a draft. Until it passes the eight-dimension assessment below, treat it as a placeholder — not as work that any agent or human can pick up.

## Hierarchy: epics, sub-issues

Work organizes as a tree:

- **Epic** = a thematic wave of work (e.g., "templates wave", "org-model rationalization"). Always carries the `epic` label.
- **Sub-issue** = a child ticket linked to an epic via GitHub's native sub-issue feature.
- **Sub-sub-issue** = a child of a sub-issue (rare; only for genuinely deep hierarchies).

Sub-issues are linked via the GraphQL `addSubIssue` mutation (with `GraphQL-Features: sub_issues` header), NOT just by reference in the body.

Title format: `<area> : <feature> : <description>` v2 — see `board-operating-model/SKILL.md` for the full convention. The previous WBS dot-decimal title prefix scheme is **deprecated** as of 2026-05-04.

### When to spawn a new epic vs add to an existing one

- **New epic** when the wave has a coherent theme, ≥3 sub-issues, and would otherwise crowd a parent.
- **Add as sub-issue** when the work fits inside an existing wave's scope.
- **Don't reuse a closed audit/decision ticket as parent** — close it; create a fresh epic if the audit's findings need execution.

## Required fields per ticket

Every ticket on the project board has:

| Field | Values | Purpose |
|---|---|---|
| **Title** | `<area> : <feature> : <description>` v2 | Searchability + scope-at-a-glance (see `board-operating-model`) |
| **Body** | structured (see below) | Context for whoever picks it up |
| **Labels** | `area:*` (required), `cleanup`, `needs:human-only`, `state:*` | Cross-cutting filter dimensions; one `area:*` per item matching title |
| **Status** | Todo / In Progress / Done | Lifecycle state (see `board-operating-model`) |
| **Release** | R0, R1.1, ..., R1.5.5, R1.6, R1.6.5, ..., R19 | Release bucket — main R-numbers for product features, half-step for process/doc/platform work |
| **Priority** | now / next / later | When-within-release; UI rank within priority bucket |
| **Assignee** | `@anthony-mansfield` or `@bytetalent-ai` | Owner / queue (see `board-operating-model` two-account model) |
| **Parent issue** | (auto-populated when sub-issue linked via `addSubIssue`) | Hierarchy |

### Body structure (reader-first, not implementer-first)

```markdown
## Why
<1-2 sentences explaining the motivation>

## What
<bullet list of deliverables — strategic level, not implementation>

## Done when
- [ ] <verifiable acceptance criterion>
- [ ] <another>

## Refs
- #<related issue>
- <other context>
```

For epic bodies, also include a **Children** section listing planned sub-issues with one-line summaries.

Implementer-level detail (specific files, function signatures, exact migrations) goes in the worker dispatch prompt, NOT in the issue body. The body stays reader-first; the dispatch brief can be expansive.

#### Bug ticket variant

Bug tickets keep the canonical sections (`## Why`, `## Done when`, `## Refs`) but **replace `## What` with two diagnostic sections** so the next reader can reproduce and start investigating without backchannel:

```markdown
## Why
<1-2 sentences — what regression or defect was hit, where, and what blocks because of it>

## Repro
1. <step>
2. <step>
3. **Expected:** <what should happen>
4. **Actual:** <what does happen>

Concrete example: <URL / project / data point that triggered it>

## Investigation hints
- <likely-cause hypothesis tied to a file/path>
- <2nd hypothesis>
- <pointer to relevant route/component to start from>

## Done when
- [ ] Repro path no longer reproduces
- [ ] <root-cause-specific criterion>
- [ ] Regression test covers the failure mode

## Refs
- <walkthrough doc / log / related issue>
```

Why split this out: a bug ticket without a repro is a guess; a bug ticket without investigation hints makes the next picker-up redo discovery the reporter already did. Both sections are cheap to write at filing time and expensive to reconstruct later.

For non-bug tickets (features, refactors, design, docs), keep `## What` as in the canonical structure — the diagnostic sections don't apply.

## Ticket completeness assessment — nine dimensions

Before dispatching a worker on a ticket — or during a sweep grooming pass — assess each ticket against nine dimensions. **A ticket failing on any dimension is not ready** and either needs editing, gating, or demoting.

This is the heart of grooming. Every other section in this skill (fields, body, hierarchy, hygiene routines) feeds into these dimensions. If you only have time for one thing during grooming, run this assessment.

### The nine dimensions

**1. Title convention** — matches v2 format `<area> : <feature> : <description>` (see `board-operating-model/SKILL.md`). No `[prefix]`, no R-numbers, no parent refs in title. The first field matches an `area:*` label.

**2. Field coverage** — Release set, Priority set (`now` / `next` / `later`), Status set (`Todo` for new), `area:*` label matching the title's first field, parent issue linked if it's a sub-issue, assignee set per the two-account model.

**3. Body completeness** — has `## Why`, `## What` (or scope), `## Done when` with verifiable acceptance criteria, `## Refs`. Body is reader-first, not implementer-detail. For epics, also includes a `## Children` (or sub-issue rollup) section.

**4. Human dependencies** — does the work require a human?
- **Account creation** (Clerk users, GitHub accounts, third-party service accounts) → human
- **OAuth / API client registration** at a third-party dashboard → human
- **Secret rotation / sensitive config** that the agent can't view → human
- **Stakeholder decision** that needs Anthony's judgment (scope, design direction, prioritization) → human
- **Real-account UAT** that needs Anthony's email/identity → human

If any apply: `needs:human-only` label + assign `@anthony-mansfield`. Otherwise: assignable to `@bytetalent-ai` for agent dispatch.

**5. Ticket relationships** — this ticket's place in the graph:
- Parent epic linked via `addSubIssue` GraphQL mutation (NOT just `Refs #N` in body)
- Non-parent dependencies declared via Blocks/Blocked-by (comment with `Blocked by #N (rationale)`)
- Related tickets cross-referenced in `## Refs`
- If 3+ related tickets exist without an umbrella, file an epic first, link as sub-issues

**6. Design requirements** — does the work need a design pass first?
- Visual changes / new screens / token changes / new component shapes → gate on a `design : <feature> : <description>` ticket
- Pencil/Figma artifacts referenced in body if applicable
- If design is unclear: file the design ticket, comment `Blocked by #<design-ticket>` on this one, demote Priority to `later` until design is decided

**7. Sizing / scope** — appropriate for a single PR / single dispatch?
- XS/S/M → ship as-is
- L → consider splitting; ship as-is if scope is genuinely cohesive
- XL → break into sub-issues; this one becomes an epic

**8. Fit for dispatch** — could a worker pick this up without backchannel?
- Done-when criteria are testable (a worker can know when it's done without asking)
- Refs link to all needed context (related tickets, prior PRs, design docs)
- No ambiguous "we'll figure it out" wording
- Implementer-level detail (file paths, function names, schemas) goes in the dispatch brief, not the body — but the body should still be unambiguous about *what* to build

**9. Body readability** — does the body actually render?
- **Real newlines, not literal `\n` strings.** Bodies authored via `gh issue create --body "..."` with escaped `\n` show the escape literally on github.com and become a wall of unreadable text. Always pass bodies via `--body-file path/to/body.md` or a heredoc (`--body "$(cat <<'EOF' ... EOF)"`) — never inline `\n`-escaped strings.
- Headings, tables, code fences render properly when viewed on github.com. Spot-check by opening the issue in the browser, or via `gh issue view <n> --web`.
- No copy-paste artifacts: smart quotes that break inline code, mojibake from bad encoding, stray backslash escapes, broken column alignment in tables.
- Scannable structure: section headings (`## Why`, `## What`) make the shape obvious. No walls of unbroken text spanning more than ~5 lines without a heading or list break.
- Length-appropriate: short, sharp body for an XS/S ticket; longer body OK for an epic, but still scannable. Implementer-level detail goes in the dispatch brief, not the body (revisit dimension 8).

If a body is unreadable, **fix it on sight** — the cost of editing is far lower than the cost of the next reader (human or agent) bouncing off it.

### Failure modes & remediation

When a ticket fails an assessment dimension:

| Failure | Remediation |
|---|---|
| Wrong title format | Retitle in place (`gh issue edit <n> --title "..."`). Fix on sight, don't ask permission. |
| Missing fields (Release / Priority / Status / `area:*`) | Set them inline. Use GraphQL batch via aliases when grooming many tickets. |
| Body missing `## Done when` | Comment requesting clarification; if blocking demo, demote Priority to `later`; if non-blocking, leave for human. |
| Has human dependency, no `needs:human-only` | Apply label + reassign to `@anthony-mansfield`. |
| Should be sub-issue, isn't | `addSubIssue` GraphQL mutation; remove redundant parent ref from body if there. |
| 3+ related tickets, no epic | File the epic first; bulk-link tickets as sub-issues; back-fill the umbrella body. |
| Needs design, no design ticket | File design ticket (`design : <feature> : <description>`); comment `Blocked by #<n>` on this one; demote Priority to `later` until design lands. |
| Scope is XL | Break into sub-issues; promote this one to epic (add `## Children` section to body). |
| Ambiguous done-when | Comment requesting clarification; demote Priority to `later` until clarified. Don't dispatch. |
| Stale label (`lane:claude`, `epic`-as-label) | Strip on sight when editing for any other reason. |
| Body has literal `\n` (escape strings instead of newlines) | Fix on sight: `gh issue edit <n> --body-file fixed.md`. Reads as a wall of text on github.com — every reader bounces off it. |
| Body fails to render (broken tables, unfenced code, mojibake) | Fix on sight. Spot-check the rendered view via `gh issue view <n> --web` after editing. |

### Two grooming modes

**Sweep mode** — periodic pass across all open tickets in a release (or the whole board). Use the nine dimensions as a checklist; produce a punch list of tickets needing edits, then bulk-fix. Best run when starting a new release, after a wave lands, or when the board has accumulated drift.

**Pre-dispatch mode** — single-ticket assessment before sending a worker. Run through the nine dimensions for *this one ticket*; if any fail, fix in place or escalate to human before dispatching. Saves the cost of a worker getting stuck mid-flight on missing context, ambiguous done-when, or a hidden human dependency.

The same dimensions apply in both modes; the cadence and breadth differ.

### Quick checklist (printable)

```
[ ] 1. Title — `<area> : <feature> : <description>` v2, no prefixes
[ ] 2. Fields — Release, Priority, Status, area:* label, assignee set
[ ] 3. Body — Why, What, Done when, Refs (epic: + Children · bug: Repro + Investigation hints replace What)
[ ] 4. Human deps — needs:human-only if applicable
[ ] 5. Relationships — parent linked, blockers declared, refs cross-linked
[ ] 6. Design — gated on design ticket if visual work
[ ] 7. Scope — fits one PR, or epic'd if not
[ ] 8. Dispatch-ready — testable done-when, complete refs, no ambiguity
[ ] 9. Readable — real newlines (no literal `\n`), tables/headings render, scannable
```

If all nine pass → ready. Any fail → fix or escalate before dispatch.

## PR title conventions

Bracket the type, then describe:

```
[r1.6] org-model 3.1 : rename consultancy → agency
[fix] /onboarding/agency: wrap client in Suspense
[infra] CI: journal monotonicity check
[ci-fix] prettier sweep on 3 files
[promote] dev → staging — R1.6 wave
[cleanup] wave 3 : type-tighten ArchitectureReviewOutput
```

Tags by purpose:

| Tag | Use for |
|---|---|
| `[r1.x]` | Feature work tied to a release |
| `[fix]` | Bug fix |
| `[infra]` | Infrastructure / deploy / tooling |
| `[ci-fix]` | Fixing broken CI specifically |
| `[promote]` | Branch promotion (dev → staging → main) |
| `[cleanup]` | Refactor / type-tighten / dead-code removal |
| `[review]` | Review-driven changes |
| `[shovel-ready]` | Small, self-contained, quick to ship |

## Branch names

Match the worker / topic:

```
claude/<topic>           → worker-driven branch (squash-merge to dev)
terminal/<topic>         → human-on-terminal branch
hotfix/<topic>           → emergency fix (cherry-picked to staging or main)
promote-<source>-to-<dest>  → branch-promotion branches
```

Always pre-compute a UNIQUE branch name when dispatching parallel workers — collisions hijack PRs.

## Required views

A project should have at least these views:

### `tasks` — primary working table

- **View type**: Table
- **Filter**: `-status:Done` (excludes done items at every level)
- **Group by**: **Release** (gives natural release-bucket chunking at the top level)
- **Sort**: Release ascending (within group, Priority now → later if a secondary sort is supported)
- **Show sub-issues / Hierarchy**: **ON** (epic → sub-issue nesting renders inside each release group)
- **Visible columns**: Title, Priority, Status, Sub-issues progress, Labels, Assignees, Linked pull requests

Why this combination works:
- Release as primary grouping is the natural top-level unit ("what's R1.6 vs R1.7")
- Hierarchy-ON inside each release group preserves epic → sub-issue context
- The `-status:Done` filter cleanly hides done items at all levels (since each level operates within its release group, the filter applies sensibly without the cross-hierarchy leak that occurs in flat / parent-grouped views)
- Sub-issues progress column gives at-a-glance epic completion %

### `boards` — kanban by status

- **View type**: Board
- **Group by**: Status (Todo / In Progress / Done)
- **Filter**: current release (e.g., `release:R1.6` during R1.6 work)
- **Use case**: at-a-glance "what's moving"

### Optional: `epics-only`

- **View type**: Table or Roadmap
- **Filter**: `label:epic`
- **Sort**: Release
- **Use case**: high-level view of wave structure across releases

## Required workflows

GitHub Projects v2 ships built-in workflows. Turn ON:

| Workflow | What it does | Why |
|---|---|---|
| **Item closed** | Closed issue → Status: Done | Auto-syncs the board with reality |
| **Item reopened** | Reopened → Status: In Progress | Surfaces work that came back |
| **Auto-add to project** | New issue in repo → added to project | Zero-friction backlog capture |
| **Auto-add sub-issues** | Sub-issue link created → child added to project | Sub-issues stay visible |
| **Item added to project** | Set default Status to Todo | Sane default |
| **Auto-archive items** | Closed >2 weeks → archived | Keeps board tidy |
| **PR linked to issue** | Status flip on PR action | Mirrors PR lifecycle |

Plus a **custom GitHub Action** for what GitHub Projects v2 doesn't do natively — auto-close parent epic when all sub-issues are closed:

```yaml
# .github/workflows/close-epic-when-children-done.yml
# Triggers on issues.closed; checks if parent's sub-issues are all closed; closes parent.
# (GitHub Projects v2 has no native rollup automation as of 2026.)
```

Reference implementation pattern available; ~60 lines of YAML using `actions/github-script` + GraphQL queries against `subIssues`.

## Filter-syntax quirks (GitHub Projects v2)

Worth knowing:

- **`-status:Done` filters at the item level**, but if an open parent contains closed children, **the children render under the parent in hierarchy view regardless of the filter**. This is intentional v2 behavior, not a bug.
- Workaround: turn OFF "Show sub-issues" in view settings for true flat filtering. OR group by Parent issue (children cluster under parents but filter applies cleanly).
- The `-` negation prefix is reliable on labels (`-label:bug`); less reliable on single-select fields. Prefer positive includes (`status:Todo,"In Progress"`) when negation acts up.
- `is:open` is rock-solid — filters at issue-state level, not Project field level.

## Periodic hygiene routines

### Weekly

- **Reconcile stale board status**: scan for issues where `state=CLOSED` but `Status≠Done`, flip them. (Should be unneeded once "Item closed" workflow is on, but catches edge cases.)
- **Surface stale-open issues**: open issues with merged PRs untouched >30 days. Don't bulk-close; surface for manual triage during the next walkthrough.
- **Branch cleanup**: delete merged feature branches whose PRs closed >7 days ago.

### Monthly

- **Audit `epic` items** for accuracy: epics where all children are done should be closed; epics with no children probably shouldn't exist.
- **Audit closed-as-duplicate** items: confirm linked PRs reference the surviving issue, not the dup.
- **Review `release: shovel-ready`**: anything that's been there >2 months either gets a real release or moves to `backlog`.

### Per-release transition (e.g., R1.6 → R1.7)

- Close any R1.6 items that are practically done (work shipped, just not formally closed).
- Move R1.6 items that won't ship to R1.7 or later, with a short rationale comment.
- Confirm all R1.6 epics are closed before declaring the release done.

## Anti-patterns to avoid

- **Filing related batch tickets without an umbrella epic** — when 3+ tickets relate, file the epic first, link as sub-issues.
- **Reusing a closed audit/decision ticket as parent** — confusing; child status doesn't roll up to a closed parent intuitively.
- **Applying retired labels** (e.g., `lane:claude` on projects that have moved off it) — strip when editing existing tickets for other reasons.
- **Triaging during walkthroughs** — capture findings as a flat list, batch-file later. (See `interactive-walkthrough-findings` skill when authored.)
- **Pre-trimming epics to "minimum viable"** — file what the plan actually says; sequence in waves not via cuts.
- **Smuggling deferred work back into related tickets** — if a feature is deferred, don't pad an unrelated UI ticket with deferred items.
- **Verbose timeline estimates in bodies** — sequence + dependencies + leverage, not days/weeks.

## Verifying issue title before dispatching a worker

Issue numbers don't always match the order tickets were drafted in. Before dispatching a worker against `#N`, confirm via `gh issue view N` that the title matches the planned scope.

## Tooling

Common scripts and APIs used in grooming:

- **Project field IDs / option IDs**: `gh project field-list <num> --owner <org> --format json` — needed for `gh project item-edit`
- **Set field**: `gh project item-edit --id <itemId> --project-id <projId> --field-id <fieldId> --single-select-option-id <optId>`
- **Add to project**: `gh project item-add <num> --owner <org> --url <issue-url>`
- **Sub-issue link**: GraphQL `addSubIssue` mutation with `GraphQL-Features: sub_issues` header
- **Bulk fetch**: `gh api graphql --paginate -F query=@query.graphql` for snapshotting board state

Cache board state to a local JSON snapshot once per session; mutate via aliases to batch GitHub API calls. See `github-api-efficiency` skill for details.

## When to invoke this skill

- **Pre-dispatch on a ticket** → run the eight-dimension assessment; if any fail, fix or escalate before sending a worker
- **Sweep grooming session** → run the eight-dimension assessment across all open tickets in a release; bulk-fix the failures
- Filing a new wave of related work → use the hierarchy + body sections
- Configuring a new project board → use the views + workflows sections
- Restructuring an epic mid-flight → use the hierarchy + anti-patterns sections
- After a multi-day work session → run the weekly hygiene routines on the board

## See also

- `board-operating-model/SKILL.md` — the rules this skill enforces (title format, fields, labels, hierarchy, two-account assignee model, half-step releases)
- `worker-dispatch-orchestration/SKILL.md` — pre-dispatch research + brief authoring (the next step after pre-dispatch grooming passes)
- `github-api-efficiency/SKILL.md` — bundle, cache, batch — essential when sweep-grooming many tickets
- `lessons-learned-discipline/SKILL.md` — where lessons go when grooming surfaces recurring failure modes
