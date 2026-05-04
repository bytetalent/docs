---
name: Backlog Grooming
description: How to organize the GitHub Projects backlog beyond field naming — required fields per ticket, body structure, views, workflows, periodic hygiene, anti-patterns. Pairs with board-numbering-conventions which covers Release/WBS/state:* fields specifically.
category: process
applicable_phases: [orchestration, planning, ongoing]
applicable_stacks: [universal]
version: 1
---

This skill captures the conventions for keeping a GitHub Projects v2 backlog organized, navigable, and actionable across a multi-agent / human team. Pairs with `board-numbering-conventions` which covers the Release field, WBS title prefix, and `state:*` labels in detail; this skill covers everything else.

## Core principle

**The backlog reflects current intent and recent state, not history.** Every open item should be something we plan to do; every closed item is recorded but invisible by default. A stale-but-open item is worse than no item at all because it lies about what's in flight.

## Hierarchy: epics, sub-issues

Work organizes as a tree:

- **Epic** = a thematic wave of work (e.g., "templates wave", "org-model rationalization"). Always carries the `epic` label.
- **Sub-issue** = a child ticket linked to an epic via GitHub's native sub-issue feature.
- **Sub-sub-issue** = a child of a sub-issue (rare; only for genuinely deep hierarchies).

Sub-issues are linked via the GraphQL `addSubIssue` mutation (with `GraphQL-Features: sub_issues` header), NOT just by reference in the body.

Title prefixes use dot-decimal WBS — see `board-numbering-conventions` skill for the full convention.

### When to spawn a new epic vs add to an existing one

- **New epic** when the wave has a coherent theme, ≥3 sub-issues, and would otherwise crowd a parent.
- **Add as sub-issue** when the work fits inside an existing wave's scope.
- **Don't reuse a closed audit/decision ticket as parent** — close it; create a fresh epic if the audit's findings need execution.

## Required fields per ticket

Every ticket on the project board has:

| Field | Values | Purpose |
|---|---|---|
| **Title** | dot-decimal WBS + description | Hierarchy + searchability (see `board-numbering-conventions`) |
| **Body** | structured (see below) | Context for whoever picks it up |
| **Labels** | `area:*`, `epic`, `cleanup`, others | Cross-cutting filter dimensions |
| **Status** | Todo / In Progress / Done | Workflow state |
| **Release** | R0, R1.1, ..., R19, shovel-ready, backlog | Release planning bucket |
| **Priority** | now / next / later | Active prioritization within release |
| **Estimate** | XS / S / M / L / XL | Sizing for capacity planning |
| **Parent issue** | (auto-populated when sub-issue linked) | Hierarchy |

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
- **Filter**: `-status:Done` (excludes done items at top level; group below brings hierarchy through)
- **Group by**: Parent issue (so children cluster under their epic)
- **Sort**: Release ascending, then Priority (now → later)
- **Visible columns**: Title, Release, Priority, Status, Labels, Parent issue, Estimate

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

- Filing a new wave of related work → use the hierarchy + body sections
- Configuring a new project board → use the views + workflows sections
- Running a grooming session → use the periodic hygiene routines
- Restructuring an epic mid-flight → use the hierarchy + anti-patterns sections
- After a multi-day work session → run the weekly hygiene routines on the board
