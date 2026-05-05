# GitHub board operations

GraphQL efficiency, project board (v2) mechanics, and ticket/epic conventions. Many of these rules emerge from incidents where naive sequential operations exhausted rate limits or silently corrupted board state.

For the board's naming/numbering conventions (WBS dot-decimal titles, Release/Priority/Status fields, label taxonomy), see `bytetalent/docs/skills/process/board-operating-model/` (the authoritative schema reference, renamed from `board-numbering-conventions/` in bytetalent/docs#22) and `bytetalent/docs/skills/process/backlog-grooming/` (v2 — authoritative source for grooming workflow). This doc covers *operating* the board efficiently, not its schema.

## GraphQL efficiency — bundle, cache, batch

GitHub's GraphQL API is rate-limited to 5000 points/hour and REST is 5000 calls/hour (separate quotas). Sloppy per-item lookup-then-edit blows the quota in 10 minutes. This rule is non-negotiable for any board-manipulation work.

**Why (incident, 2026-05-03):** burned ~3800 points in ~30 min by querying each project item one at a time before editing it. Each query returned all 200 board items just to find one project item ID — quadratic cost. Then a second call to mutate. Repeat 50 times → quota exhausted, work stalled.

**How to apply:**

1. **Pull board state ONCE per session** — single paginated GraphQL query that fetches all project items + their relevant fields (release, priority, status, labels, assignees, sub-issues). Cache as a local `.tmp/board-cache.json`. Re-query only when structural changes have been made.

2. **Always batch mutations via aliases** — N item updates = 1 GraphQL call with N aliases, NOT N calls. Same applies to issue creation, sub-issue links, blocked-by edges. Format:

   ```graphql
   mutation {
     a1: updateProjectV2ItemFieldValue(input: {...}) { ... }
     a2: updateProjectV2ItemFieldValue(input: {...}) { ... }
     a3: addSubIssue(input: {...}) { ... }
   }
   ```

3. **Cache stable IDs** — Project ID, field IDs, option IDs change rarely. Hardcode them in the session or the .tmp cache. Never re-query "what's the ID of the Release field" 20 times.

4. **Use REST for bulk reads when possible** — `gh api repos/.../issues?state=open&per_page=100` is rate-limit cheaper than GraphQL for plain issue lists. Save GraphQL for project-board specifics (field values, sub-issues, project items).

5. **Pre-flight rate check** — start each board session with `gh api graphql -f query='query { rateLimit { remaining limit resetAt } }'`. If <500 remaining, switch to REST or wait.

**What this rule does NOT mean:**
- Don't avoid GraphQL — it's strictly more capable than REST for projects.
- Don't skip queries actually needed — caching is per-session, not per-eternity.
- Don't pre-optimize creation flows that genuinely need sequencing (e.g. parent-then-child).

**Codified as:** skill `bytetalent/docs/skills/github-api-efficiency/SKILL.md` — agent-facing reference with worked examples.

## updateProjectV2Field on single-select wipes existing option IDs

When using GraphQL `updateProjectV2Field` mutation on a `SINGLE_SELECT` field to add a new option, the mutation **recreates the entire options array** with brand-new IDs — even for unchanged options. This **wipes the field value on every project item that had it set**, because the old option IDs no longer exist.

**Why (incident, 2026-05-04):** Migration worker on #384 added `R1.8` and `shovel-ready` options to the Release field. After the mutation, ALL project items showed empty Release. Required ~15 additional batched mutations to restore the pre-existing values using the NEW option IDs. Cost: extra ~600 GraphQL points + the cognitive load of recovering what had been there.

**How to apply:**

- **Before** running `updateProjectV2Field` on a single-select, **snapshot all items' current value** for that field. Store as `(itemId, optionLabel)` pairs.
- **Run the mutation** with the new options (including all existing ones explicitly).
- **Capture the new option IDs** from the response (or by re-querying the field).
- **Restore each item's value** by mapping the snapshotted label → new option ID, then batched `updateProjectV2ItemFieldValue` calls.
- Use **batched GraphQL** (aliases) for the restoration — N items in one call.

**Known cases:**
- Adding a new release option (R1.8, shovel-ready) — affected the entire board
- Same likely applies to other single-select fields if updated this way: Status, Priority, Iteration, custom enums

**Mitigation if forgotten:**
- Symptom: items appear with blank field after the mutation
- Recovery: query the project for current state (now blank), reconstruct the old values from a snapshot or guess from issue metadata, batched restore via the new option IDs

**Future avoidance:** if GitHub adds an option-only mutation that doesn't touch existing options, use that. Currently `updateProjectV2Field` is the only path. Consider scripting this routine: `script add-release-option <name> <position>` that handles the snapshot + restore automatically.

## Verify issue title before referencing in dispatch brief

When opening multiple sub-issues in a batch (e.g., decomposing an epic into N sub-issues), GitHub assigns numbers in creation order — but the order may NOT match the mental order in the draft. **Always verify the title-to-number mapping with `gh issue view <N> --json title` before dispatching a worker brief that references the issue number.**

**Why (incident, 2026-05-03):** Drafted 6 R1-5c sub-issues mentally as `dispatcher → context-loader → planner → validation → non-feature-handlers → pr-opener`, but the actual creation order put validation as #290 and non-feature-handlers as #291. Then dispatched a "validation gates" worker against #289 (which was actually planner). PR #309 closed #289 (planner) when its work was validation gates. PR title said "Closes #289" so on merge GitHub auto-closed planner — work that hadn't been done. Took 30 minutes to unwind: reopen #289, close #290 manually, re-parent the 3 sub-issues that had been linked to wrong parent.

**How to apply:**

- After creating a batch of issues with `gh issue create`, dump the title-to-number map: `for n in <numbers>; do echo "#$n: $(gh issue view $n --json title --jq .title)"; done`
- When drafting a worker brief, paste the issue body (`gh issue view N --json body`) into the brief or read it cold — don't reference the number from memory.
- When decomposing an epic into sub-issues, link the new sub-issues to the parent IMMEDIATELY (sub-issue mutation) so the relationship is recorded before further work pollutes the issue space.
- Trust the gh title verification, not draft order. Issue numbers are assigned in creation order; titles travel with the issue.

## Bundle related batch tickets under an explicit epic

When filing a batch of 3+ related tickets (e.g. ship-now templates, schema retrofit, arch-phase consumption all under one theme), create a dedicated **epic** ticket and link the batch as sub-issues of that epic.

**Why:** Anthony 2026-05-03 — wants proper epic structure for related work. Reasoning:
- Epics show up correctly in board filters and sub-issue progress widgets
- A closed audit/decision ticket is awkward as a parent (links still navigate but the parent is "done", which is misleading)
- Labels (`epic`) drive board grouping and reports

**How to apply:**
- Before filing the batch, file the epic first (one ticket, label `epic`, title summarizing the wave, body listing planned children)
- Add the epic to the project board with appropriate release/priority/estimate
- File the children with bodies that reference `Sub-issue of #<epic>` and link them via the GraphQL `addSubIssue` mutation
- For follow-on tickets that emerge later in the same theme, link them under the same epic (not a sibling)
- Distinct from audit/decision tickets: the audit ticket asks the questions; the epic carries the implementation. They can coexist (epic refs the audit), or once decisions land, the audit gets closed and the epic continues.

## Don't apply the lane:claude label to new tickets

Don't apply `lane:claude` when filing new GitHub issues. The label is being retired.

**Why:** Anthony 2026-05-03 — flagged that "we were dropping lane:claude, still appears on new items". I'd been applying it by default on every new issue (especially ticket batches). The label predates a process change and shouldn't propagate to new work.

**How to apply:**
- New issues: don't pass `--label lane:claude`. Use only domain labels (`area:platform`, `area:frontend`, `epic`, `cleanup`, etc.)
- Existing issues with the label: leave as-is unless explicitly cleaning, but strip when already editing the issue for other reasons
- Worker briefs / dispatch flow: an issue's claudability is implicit (any open issue I dispatch a worker for IS claudable); the label was redundant overhead
- If catching myself writing `lane:claude` in a script or template, stop and remove it
