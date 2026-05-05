---
name: Agent Settings JSON
description: Canonical .claude/settings.json allow/deny baseline every Bytetalent repo must commit; per-repo extension rules; stalled-worker diagnostic.
category: process
applicable_phases: [all]
applicable_stacks: [universal]
version: 1
---

Every Bytetalent repo ships a committed `.claude/settings.json` that pre-approves the commands autonomous Claude agents need to claim issues, branch, typecheck, and open PRs. Without it, agents spawned into isolated worktrees stall silently — the worktree does not inherit the user's `.claude/settings.local.json` or `~/.claude/settings.json`. Source: `bytetalent/docs/guides/agents.md`.

## Canonical baseline

Copy this block verbatim into `.claude/settings.json` at repo root. Extend it; never shrink it.

```json
{
  "permissions": {
    "allow": [
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git push:*)",
      "Bash(git fetch:*)",
      "Bash(git pull:*)",
      "Bash(git status:*)",
      "Bash(git log:*)",
      "Bash(git diff:*)",
      "Bash(git show:*)",
      "Bash(git stash:*)",
      "Bash(git checkout:*)",
      "Bash(git switch:*)",
      "Bash(git branch:*)",
      "Bash(git rev-parse:*)",
      "Bash(git restore:*)",
      "Bash(git rebase:*)",
      "Bash(git merge-tree:*)",
      "Bash(gh issue view:*)",
      "Bash(gh issue edit:*)",
      "Bash(gh issue list:*)",
      "Bash(gh issue comment:*)",
      "Bash(gh issue create:*)",
      "Bash(gh pr create:*)",
      "Bash(gh pr view:*)",
      "Bash(gh pr list:*)",
      "Bash(gh pr edit:*)",
      "Bash(gh pr comment:*)",
      "Bash(gh pr checks:*)",
      "Bash(gh api:*)",
      "Bash(gh run list:*)",
      "Bash(gh run view:*)",
      "Bash(gh repo view:*)",
      "Bash(npm install:*)",
      "Bash(npm run:*)",
      "Bash(npx prettier:*)",
      "Bash(npx eslint:*)",
      "Bash(npx tsc:*)",
      "Bash(pnpm install:*)",
      "Bash(pnpm run:*)",
      "Bash(pnpm build:*)",
      "Bash(pnpm list:*)",
      "Bash(pnpm add:*)",
      "Bash(bun install:*)",
      "Bash(bun run:*)",
      "Bash(bun add:*)",
      "Bash(bun x:*)"
    ],
    "deny": [
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/*.pem)",
      "Read(**/*.key)",
      "Read(**/*.p12)",
      "Read(**/credentials.json)",
      "Read(**/service-account*.json)",
      "Bash(git push --force *)",
      "Bash(git push -f *)"
    ]
  }
}
```

## Why each allow group

- **`git` core** — branch, commit, push (non-force), fetch, pull, status, log, diff. Everything an agent needs for source control without footguns.
- **`gh issue` / `gh pr` / `gh api` / `gh run` / `gh repo`** — claim issues, open + verify PRs, query CI. `gh api` covers GraphQL escape hatches.
- **`npm` / `npx` / `pnpm` / `bun`** — all four package managers are pre-approved so a worker never stalls on a package-manager mismatch.

## Why each deny

- **Secret-file reads** — `.env*`, `*.pem`, `*.key`, `*.p12`, `credentials.json`, `service-account*.json`. Prevents agent transcripts from leaking secrets.
- **Force-push** — `git push --force` and `git push -f` are permanently blocked. A bad rebase must never destroy shared branch history.

## What does NOT belong in committed settings

- **MCP allows** (Pencil, Supabase, Vercel) — personal-environment config; lives in `~/.claude/settings.json`. Committing them per-repo creates noise with no benefit.
- **`gh pr merge`** — agents must not merge their own PRs. The rule is enforced at the prompt layer in the implementer agent definition; a harness deny would also block the orchestrator from setting auto-merge.
- **Anything project-specific that other repos won't have** — project-board commands, custom tooling, generated-client secrets. Those belong in the repo that uses them.

## Per-repo extension rules

A repo extends the baseline by adding entries to the `allow` array only. Never remove baseline entries — each one exists because a real failure or threat model required it.

Common patterns for per-repo additions:
- Project-board commands (`gh project item-edit`, `gh project item-list`) for repos whose workers must flip issue status.
- Repo-specific build tools (e.g., `Bash(supabase:*)` for a db-admin repo).

## Setting up a new repo

1. Copy the canonical block above into `.claude/settings.json` at repo root.
2. Add only what is repo-specific to the `allow` list.
3. Don't trim the baseline.
4. Verify by spawning a no-op worker (e.g., a doc-only PR) and confirming it completes without hitting prompts.

## Stalled-worker diagnostic

If an autonomous worker stops making progress and never reports completion:

1. Confirm the worker is in an isolated worktree. If yes, `.claude/settings.local.json` and `~/.claude/settings.json` are NOT visible — only the committed `.claude/settings.json` applies.
2. Find the last command the worker attempted. Match it against the committed `allow` list.
3. If there is a mismatch, add the missing entry to the committed file and open a PR. Do not add it to `.claude/settings.local.json` — the next worker won't see it either.

## Anti-patterns

- Omitting the committed file and relying on user-level `~/.claude/settings.json` — workers in worktrees never see that file.
- Adding MCP server entries (Pencil, Supabase) to the committed file — these are personal-environment config, not repo config.
- Trimming a baseline entry to "reduce noise" — every entry has a documented reason; shrinking the list guarantees a future worker stalls on the removed command.

## Refs

- `bytetalent/docs/guides/agents.md` — canonical guide with incident narrative (bt-ai-web PR #259)
- `skills/process/worker-dispatch-orchestration/SKILL.md` — orchestration layer that depends on this baseline being in place
