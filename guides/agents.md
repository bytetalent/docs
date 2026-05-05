# Agents

How Bytetalent repos are configured for autonomous Claude Code (and other agentic CLI) sessions: what permissions to allow, why, the failure mode this prevents.

For the canonical allow/deny list, per-repo extension rules, and stalled-worker diagnostic, see [`skills/process/agent-settings-json`](../skills/process/agent-settings-json/SKILL.md).

For worker dispatch orchestration patterns (how the orchestrator coordinates multiple workers), see [`skills/process/worker-dispatch-orchestration`](../skills/process/worker-dispatch-orchestration/SKILL.md).

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

## Per-agent definition (`.claude/agents/*.md`)

Agent definitions also live in `.claude/`, also committed. The implementer agent's hard rules ("don't merge your own PR", "don't push to dev/staging/main directly") are enforced at the prompt layer — that's the _prompt-time_ defense. The harness deny list is the _runtime_ defense for things that can never be acceptable (force-push, secret reads). They complement each other; neither replaces the other.

---

## Reference

The canonical `.claude/settings.json` lives in `bytetalent/bytetalent-nextjs-clerk-supabase-base/.claude/settings.json`. Other repos copy it and extend.

Failure-mode reference: `bytetalent/bt-ai-web` PR #259 ("Unblock implementer workers — add gh/git/npm permissions") and the `.claude/agents/implementer.md` workflow it unblocks.

See [`skills/process/agent-settings-json`](../skills/process/agent-settings-json/SKILL.md) for the full allow/deny list, what NOT to commit, per-repo extension rules, new-repo setup steps, and stalled-worker diagnostic.
