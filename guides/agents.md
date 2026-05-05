# Agents

How Bytetalent repos are configured for autonomous Claude Code (and other agentic CLI) sessions: what permissions to allow, why, the failure mode this prevents.

---

## TL;DR

Every Bytetalent repo — this `docs` repo, the `templates` repo, every `*-base` repo, every generated client repo, and `app-flow.ai` itself — ships with a committed `.claude/settings.json` that pre-approves the commands an autonomous agent needs to:

1. Claim a GitHub issue (`gh issue view`, `gh issue edit`)
2. Branch and commit (`git ...`)
3. Run typecheck / lint / tests (`npm run`, `npx tsc`, `pnpm`, `bun`)
4. Open a PR (`gh pr create`, `gh pr view`, etc.)

Without these committed allows, agents running in background mode hit permission prompts, can't get user approval, and stall silently.

---

## The failure mode this prevents

App-flow.ai's PR #259 (2026-05-03) is the canonical example. Three workers in a row stalled silently on issue #246. Root cause: the committed `.claude/settings.json` allow list was nearly empty.

Two things make this trap easy to fall into:

1. **Workers spawned in isolated worktrees don't inherit the user-level `bypassPermissions` mode.** The human running Claude Code interactively never sees prompts; the autonomous worker hits one on the very first command.
2. **`.claude/settings.local.json` is gitignored.** Whatever the human approved over time lives there. A worker spawned into a fresh worktree gets a stale or empty snapshot — the file isn't checked into git, so it doesn't follow.

Net effect: the human's session works fine, but every autonomous worker stalls on the first restricted command. The orchestrator has no auto-alert for this state — only completion notifications, which never arrive.

---

## The standard `.claude/settings.json`

Every base and generated client repo commits this baseline. App-flow.ai extends it with project-specific additions (see "Per-repo additions" below).

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

### Why each allow group

- **`git` core** — branch, commit, push (non-force), fetch, pull, status, log, diff, etc. Everything an agent needs to do source control without footguns.
- **`gh issue` / `gh pr` / `gh api` / `gh run` / `gh repo`** — claim issues, open + verify PRs, query CI state. `gh api` covers GraphQL escape hatches.
- **`npm` / `npx` / `pnpm` / `bun`** — every Bytetalent stack uses one of these. Pre-approving all four means a worker doesn't stall on package-manager mismatch.

### Why each deny

- **Secret-file reads** — `.env*`, `*.pem`, `*.key`, `*.p12`, `credentials.json`, `service-account*.json`. Protects the agent transcript from leaking secrets even if the human accidentally `cat`s one in chat.
- **`git push --force` / `git push -f`** — agents must never force-push. A bad rebase shouldn't be able to destroy history on a shared branch.

---

## What does NOT belong in the committed `.claude/settings.json`

- **MCP allows.** Pencil, Supabase staging/prod, Vercel — those are personal-environment config. They live in `~/.claude/settings.json` (user-level) where the human has them anyway. Committing them per-repo creates noise and gives no benefit.
- **`gh pr merge`.** Agents must not merge their own PRs. The rule is enforced at the prompt layer (`.claude/agents/implementer.md` "Never merge your own PR"). Adding it as a harness deny would also block the orchestrator/human from setting auto-merge, which is the standard workflow.
- **Anything project-specific that other repos won't have.** Project-board commands, custom tooling — those belong in the repo that uses them.

---

## Per-repo additions

A repo can extend the baseline with allows specific to its tooling. The base set should never shrink — repo-specific files only add.

### App-flow.ai adds

- **`Bash(gh project item-edit:*)`** + **`Bash(gh project item-list:*)`** + **`Bash(gh project field-list:*)`** — workers must flip issue status on Project #3 (`app-flow.ai` board) when claiming. See the work-tracking standard in `app-flow.ai`'s `docs/guide-process.md`.
- **MCP entries for connected providers** (Pencil, Supabase staging/prod, Vercel) — only the read-only ones used during PR verification. Write-side MCPs stay user-level.

### A base running its own dogfood site might add

- **MCP entries for its host** (Cloudflare, Vercel deploy logs) if dogfood verification depends on querying live state.

### A generated client repo might add

- Whatever the consultant configured at scaffold time. The pipeline can extend the baked-in baseline based on the project's connection set.

---

## Per-agent definition (`.claude/agents/*.md`)

Agent definitions also live in `.claude/`, also committed. The implementer agent's hard rules ("don't merge your own PR", "don't push to dev/staging/main directly") are enforced at the prompt layer — that's the *prompt-time* defense. The harness deny list is the *runtime* defense for things that can never be acceptable (force-push, secret reads). They complement each other; neither replaces the other.

---

## When you're setting up a new repo

1. Copy the canonical `.claude/settings.json` from `bytetalent/bytetalent-nextjs-clerk-supabase-base` (or any other Bytetalent repo with this guide already applied).
2. Add only what's repo-specific to the allow list.
3. Don't trim the baseline — every entry above came from a real failure or threat model.
4. Verify by spawning a no-op worker (e.g., dispatch a doc-only PR) and confirming it completes without hitting prompts.

---

## When you're investigating a stalled worker

1. Check if the worker is in an isolated worktree. If yes, `.claude/settings.local.json` and `~/.claude/settings.json` are NOT visible to it — only the committed `.claude/settings.json`.
2. Check what command the worker last attempted. Match it against the committed allow list. Mismatch = the source of the stall.
3. Add the missing allow to the committed file. Open a PR. Don't put it in `.claude/settings.local.json` — the next worker won't see it either.

---

## Reference

The canonical `.claude/settings.json` lives in `bytetalent/bytetalent-nextjs-clerk-supabase-base/.claude/settings.json`. Other repos copy it and extend.

Failure-mode reference: `bytetalent/bt-ai-web` PR #259 ("Unblock implementer workers — add gh/git/npm permissions") and the `.claude/agents/implementer.md` workflow it unblocks.
