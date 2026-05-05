---
name: Worker Dispatch Orchestration
description: How to triage incoming work, pre-research tickets before handoff, author tight worker briefs, and dispatch waves of parallel agents without collisions. Pairs with branch-and-worktree-hygiene (cleanup), backlog-grooming (where the work comes from), and github-api-efficiency (board state ops).
category: process
applicable_phases: [orchestration, dispatch]
applicable_stacks: [universal]
version: 1
---

This skill captures the pre-dispatch craft that determines whether a worker spends 3 minutes or 30 minutes on the same ticket. Most of a worker's runtime is "explore the codebase to figure out what to do" — which the orchestrator can compress dramatically by doing the research up front and baking it into the brief.

The orchestrator's job is **to make the worker's job small**. Time spent pre-researching is time saved many-fold across worker runtime.

## When to invoke

- Anytime you're about to spawn an `Agent({subagent_type: "implementer"})` worker
- Especially when dispatching 3+ workers in parallel — collision risk goes up with count
- When a previous worker took longer than expected and you want to compress the next round
- When triaging a backlog of cleanup tickets into a wave plan

## The dispatch loop (DMAIC framing)

This skill is the **D (Define) + M (Measure) + I (Improve)** parts of the framework's continuous-improvement loop:

1. **Define**: Read the ticket. What's actually broken / what needs to change?
2. **Measure**: Locate it in the codebase. Which files, which lines, which patterns?
3. **Analyze**: What's the right shape of the fix? Are there related tickets that should be combined?
4. **Improve**: Author the brief. Hand the worker a tight scope so they execute, not explore.
5. **Control**: Watch for landings. Capture lessons. Update this skill (or others) when patterns recur.

## Pre-dispatch research — the time-saver

For every ticket about to be dispatched, the orchestrator should:

### 1. Pre-locate exact files

Don't write `find the invite dialog component`. Write `src/components/admin/comp-invite-agency-modal.tsx:23-47`. Save the worker the grep-and-glob phase.

```bash
# Quick recipes for finding things:
gh issue view <N> --json body --jq .body  # Get the issue body
grep -rn "ComponentName" src/             # Find component
glob "src/app/admin/**/page.tsx"          # Find admin pages
```

### 2. Identify the precise change when known

If the bug is mechanical (CSS sizing, contract-mismatch field rename, exact-string change), give the worker the diff target directly. Example:

```
Bad: "fix the cron 401 issue"
Good: "src/app/api/cron/phase-job-watchdog/route.ts:60 — change
       `req.headers.get('user-agent') !== 'vercel-cron'` to
       `!req.headers.get('user-agent')?.startsWith('vercel-cron')`.
       Vercel sends `vercel-cron/1.0` (versioned), exact-equals fails."
```

### 3. Paste relevant snippets

If the worker needs to understand a pattern, paste it instead of citing it. Saves a Read tool call.

```
Match the wizard dialog style at src/app/space/projects/components/wizard-dialog.tsx:
  <DialogContent className="max-w-2xl px-6 py-5"> ... </DialogContent>
```

### 4. Group related tickets per worker

If two tickets both touch `/admin/crons`, give them to ONE worker — saves a build/validation cycle and avoids merge conflicts. Examples that group well:

- Same page or component
- Same data model (e.g. invitations: validate + accept endpoints)
- Same area + same cleanup theme
- One depends on the other

### 5. Choose the right model

```
Opus  — schema changes, complex business logic, cross-cutting refactors,
        anything where wrong choices cascade
Haiku — CSS tweaks, copy edits, exact-known field renames, deletes,
        contract-mismatch fixes where the diff is 1-3 lines
```

Specify in the Agent call: `model: "haiku"` for trivial; default for complex.

### 6. Pre-compute the branch name

Per `multi-agent-git-workflow.md → Workers must use uniquely-named branches`, the brief MUST specify the exact branch name. Format: `claude/<N>-<slug>` for first attempt; `claude/<N>-redo-<date>` for re-dispatches. NEVER let workers pick.

For multi-issue workers: `claude/<N>-<M>-<combined-slug>` (e.g., `claude/589-590-crons-end-to-end-fix`).

## Worker brief template

Every brief follows this structure. Cut sections only if irrelevant; don't reorder.

```markdown
You are an implementer agent fixing issue #<N> in <repo>.

## Issue
<paste issue body, OR a tight summary if the body is long>

## Fix needed
<the precise change — file paths, exact lines/patterns, before/after>

## Working setup
- Repo: <absolute path>
- Base: `origin/<branch>` (NOT main; <branch> is integration branch)
- Branch (use exactly): `claude/<N>-<slug>`
- Stay in your `.claude/worktrees/agent-*` worktree.
  Never `git checkout` or modify files in the parent repo's main worktree.

## Scope
1. <step 1, with file path>
2. <step 2, with file path>
3. <step 3>

## Validation before opening PR
- `npx prettier --check` (or `--write` then check again)
- `npx eslint`
- `npx tsc --noEmit`

## PR
- Open against `<integration-branch>` (squash-merge target)
- Title: `[<area>] <topic> (#<N>)`
- Body: brief summary + "Closes #<N>"
- Enable auto-merge: `gh pr merge <num> --auto --squash`

## Reporting
Report when PR is opened. Include PR number + a one-line summary of what
changed. If you hit a blocker (schema confusion, unclear behavior),
surface it before forging ahead.

## Constraints
- Don't skip pre-commit hooks (husky/lint-staged)
- Don't read .env files (per never-read-env-files seed)
- Stay in your assigned worktree
```

## Wave-based dispatch

When dispatching N tickets, plan in waves rather than firing all at once.

### Wave composition

- **5-7 workers per wave** is the sweet spot — enough parallelism to maximize throughput, few enough to track + diagnose stalls
- **Non-overlapping file scopes** within a wave — workers in the same file can collide on `dev` even with squash-merge (each squash adds a commit; the SECOND merge's rebase against the FIRST may conflict)
- **Mix of high-priority and small tickets** — keeps cycle time even

### Wave 1 vs Wave 2 sequencing

Wave 2 dispatches after Wave 1 starts to land (PRs auto-merging). Reasons:
- Wave 1 reveals patterns the orchestrator can fold into Wave 2 briefs
- Wave 2 sees a fresher `dev` (with Wave 1's fixes) — less merge conflict risk
- If Wave 1 surfaces a blocker (e.g., a finding that Wave 2 depended on), can adjust before dispatch

### Active worker tracking

Maintain a mental (or written) list of in-flight worker IDs. Used for:
- **Worktree preservation**: when running cleanup (per `branch-and-worktree-hygiene`), preserve active worker worktrees
- **Stall detection**: per orchestration guide — ping at 2x expected duration, escalate at 3x
- **Wave 2 timing**: dispatch when Wave 1 is roughly half-landed

## Anti-patterns

### Vague briefs that force exploration

```
Bad: "fix the cron issue"
     "make the popup bigger"
     "polish the cards"

Good: "src/.../route.ts:60 — change X to Y because Vercel sends..."
      "max-w-md → max-w-2xl, py-4 → py-5; match wizard-dialog.tsx"
      "audit text-size utilities on /space/clients/.../projects/...;
       converge on text-base/text-sm/text-xs (no arbitrary text-[Npx])"
```

### Over-parallelizing into the same file

If 3 tickets all touch `src/app/admin/page.tsx`, don't dispatch 3 workers — combine into 1. Even with squash-merge, parallel PRs against the same file race on the merge order.

### Forgetting to cite branch name

Per `multi-agent-git-workflow.md`, branch collision destroys work silently. ALWAYS pre-compute branch name and put it in the brief. Never `claude/<your-issue>-<your-slug>` — too easy for workers on related tickets to collide.

### Reading .env files

Per `engineering-discipline.md → Never read whole .env files`. Workers should never read .env*; if a worker's brief implies they need to, the orchestrator failed to give them what they need.

### Not enabling auto-merge

Workers should `gh pr merge <n> --auto --squash` after opening. Otherwise PRs pile up at the review gate and the queue stalls. Per the auto-merge seed: opt out only for risky/substantial changes.

### Forgetting to clean up after a wave

After a wave's PRs all land, run `branch-and-worktree-hygiene` cleanup. Stale worktrees + orphaned branches accumulate fast (50+ in a month is normal without cleanup).

## Working example — Wave dispatch from cleanup epic

Scenario: 26 cleanup tickets across 5 sub-epics. The orchestrator's pre-work:

1. **Triage** by area: phase-gen (3) / connections (6) / admin-nav (6) / project-create (5) / invitations (2)
2. **Group related tickets**: invitations #581+#582 → one worker; crons #589+#590 → one worker
3. **Wave 1 = launch-blockers + small wins** (6 workers):
   - Phase-gen retry (P0, complex — Opus)
   - Crons end-to-end (P0, contract fix — Opus)
   - /admin index page (small new page — Opus or Haiku)
   - Invitation flow (medium, related pair — Opus)
   - Invite popup sizing (CSS only — Haiku)
   - Project card text sizes (CSS audit — Haiku)
4. **Pre-research each**: file paths, exact lines, related code patterns
5. **Dispatch in parallel** with detailed briefs
6. **Wave 2** ready while Wave 1 lands: connections cluster (one worker, page-scoped), project-create cluster (one worker, flow-scoped), nav IA (later, design-heavy)

Time savings: workers in this wave averaged 2-3 min each instead of 5-15 because the briefs were tight.

## What this skill does NOT cover

- **Per-ticket implementation guidance** — that lives in product code conventions + the specific skill libraries (data-access-patterns, form-patterns, etc.)
- **Backlog grooming** — see `backlog-grooming` skill (how tickets get authored + organized in the first place)
- **Branch + worktree cleanup** — see `branch-and-worktree-hygiene` skill (the destructive cleanup half)
- **Stall detection on long-running workers** — see project-specific orchestration guide (e.g., `<product>/docs/guide-orchestration.md`)

## Codified seed memories that pair with this

- `bytetalent/docs/agent-framework/memory-seeds/multi-agent-git-workflow.md → Workers must use uniquely-named branches + Workers must stay strictly inside their assigned worktree`
- `bytetalent/docs/agent-framework/memory-seeds/multi-agent-git-workflow.md → PR auto-merge by default`
- `bytetalent/docs/agent-framework/memory-seeds/engineering-discipline.md → Never read whole .env files + No shortcuts on infrastructure setup`
- `bytetalent/docs/agent-framework/memory-seeds/collaboration-style.md → When blocked on user input, clear other work in parallel`
