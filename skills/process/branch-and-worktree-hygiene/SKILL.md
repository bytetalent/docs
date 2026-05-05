---
name: Branch and Worktree Hygiene
description: Periodic audit + cleanup of long-lived branch divergence, stale local worktrees, and merged-but-not-deleted remote branches across multi-repo projects. Includes the realign procedure for keeping main → staging → dev SHA-aligned after promotions. Pairs with backlog-grooming and github-api-efficiency.
category: process
applicable_phases: [orchestration, periodic, maintenance]
applicable_stacks: [universal]
version: 1
---

This skill captures the periodic maintenance that keeps a multi-repo, multi-agent project from accumulating debris. Over time, branches don't auto-delete, worktrees don't unlock, SHAs drift between long-lived branches — left untreated, the workspace becomes a forest of stale state that misleads tooling and hides real changes.

## When to invoke

- **Weekly**: lightweight survey (no destructive actions); detect drift, list candidates
- **After every staging → main promotion**: realign step (rebase staging onto main, then dev onto staging) — see "Realign procedure" below
- **Monthly or end-of-release**: full cleanup pass with destructive actions (delete merged remote branches, prune local worktrees)

The survey half is automation-friendly (read-only); the cleanup half should stay human-supervised at first since it involves force-pushes and worktree deletions.

## Three things to audit

### 1. Branch divergence (long-lived branches)

For projects with `dev` / `staging` / `main`:

```bash
git fetch origin --prune main staging dev

git log --oneline origin/staging ^origin/main | wc -l   # commits to promote main-ward
git log --oneline origin/dev ^origin/staging  | wc -l   # commits to promote staging-ward
git log --oneline origin/main ^origin/staging | wc -l   # SHOULD be 0
```

**Healthy state:**
- main → staging: 0 commits (main is a strict subset of staging or aligned)
- staging → main: 0 right after a promotion+realign; grows with each merge to staging
- dev → staging: ~5–15 commits at any time (active work since last promotion)

**Unhealthy signals:**
- main → staging > 0 → someone hot-fixed prod without backporting; address immediately (cherry-pick to staging + dev)
- dev → staging much larger than the actual work shipped → SHA drift from prior squash-merges; resolve via realign
- staging → main → 50+ → production is far behind; promote sooner rather than later (each delay compounds)

### 2. Local worktrees

Worker dispatches via `Agent({ isolation: "worktree" })` create `.claude/worktrees/agent-<id>` directories. Most are locked after the worker completes; many never get cleaned.

```bash
git worktree list                # see all worktrees including locked
git worktree list --porcelain    # parseable output
```

A typical state: ~50–80 stale worktrees over a few months. None of them are doing anything; they just take disk space and clutter `git branch -a`.

**Cleanup steps:**

```bash
# Step 1 — for each worktree we want to remove:
git worktree unlock <path>       # if locked
git worktree remove <path>       # this also cleans up the branch ref if no other ref holds it

# Step 2 — sweep dead worktrees (paths that no longer exist on disk):
git worktree prune
```

Don't bulk-remove without checking the branch ref first — if a worktree has uncommitted work or a branch that doesn't exist elsewhere, removing it loses data. Filter to those whose branch is `worktree-agent-<id>` or whose underlying PR is merged.

### 3. Remote branches

After PRs squash-merge, GitHub usually auto-deletes the source branch — but only if "Automatically delete head branches" is enabled in the repo settings. Some repos don't have it on; merged branches accumulate.

```bash
gh api repos/<owner>/<repo>/branches --paginate --jq '.[] | .name'
```

Cross-reference against PRs:

```bash
gh pr list --state merged --base main --limit 100 --json number,headRefName,mergedAt
```

Branches that match a merged PR's `headRefName` are safe to delete:

```bash
gh api -X DELETE repos/<owner>/<repo>/git/refs/heads/<branch>
```

**Always verify auto-delete-on-merge is enabled** in the repo settings going forward — Settings → General → Pull Requests → "Automatically delete head branches". Set it once, save manual cleanup forever.

## Realign procedure

This is the canonical "after staging → main promotion" cleanup. CLAUDE.md references it; documenting in detail here.

**Why:** The staging→main PR rebase-merges (or squash-merges in the merge-commit case) which creates new SHAs on main. Staging and dev still hold the OLD SHAs. Future dev→staging or staging→main PRs show false divergence — every "old" commit appears as new commits at different SHAs.

**Fix:** Rebase staging onto main, force-push. Then rebase dev onto staging, force-push. After this, all three branches share SHA history; the only commits "ahead" of main are the genuinely new ones.

```bash
# 1. Temporarily relax branch protection on staging + main (UI: Settings → Branches → edit rule → uncheck "Restrict force pushes")

# 2. Realign staging
git fetch origin main staging --prune
git checkout staging
git reset --hard origin/main          # staging tip = main tip
git push --force-with-lease origin staging

# 3. Realign dev
git fetch origin staging dev --prune
git checkout dev
git rebase origin/staging              # replays dev's actually-new commits onto staging
# (resolve any conflicts — they should be rare since dev was already ahead)
git push --force-with-lease origin dev

# 4. Re-enable branch protection
```

After this, `git log --oneline origin/dev ^origin/staging` shows only the genuine new commits (e.g., 3–10), not the inflated count caused by SHA drift.

**Force-push safety**: use `--force-with-lease` (not `--force`) so the push fails if someone else pushed in the meantime. Also coordinate with anyone working on the affected branch — their local copy needs `git fetch && git reset --hard origin/<branch>` after the realign.

## Multi-repo coordination

For projects with multiple repos (e.g., `<org>/web-app` + `<org>/templates` + `<org>/docs`), run the audit across all of them. Each has its own remote-branch state and may have different auto-delete settings.

Common pattern: the primary application repo has auto-delete-on-merge ON (clean); satellite repos (templates, docs, infrastructure) often don't (debris accumulates). Worth a one-time sweep + setting check across all of them.

```bash
for repo in web-app templates docs; do
  echo "=== $repo ==="
  gh api repos/<owner>/$repo/branches --paginate --jq '.[] | .name' | wc -l
done
```

## Anti-patterns

- **Force-pushing without `--force-with-lease`** — risks clobbering teammates' commits silently
- **Bulk-deleting worktrees with locked status** — locks usually exist for a reason; verify first
- **Skipping the realign after staging→main** — guarantees the SHA-drift problem recurs on next promotion
- **Merging dev→staging without first checking dev/staging divergence** — large divergence often means there's a backport hole (a commit on staging that dev doesn't have)
- **Running cleanup during active worker dispatches** — wait until in-flight workers complete; their worktrees are not stale

## Tooling

- `git worktree list` / `--porcelain`
- `git worktree prune` / `unlock` / `remove`
- `git remote prune origin` — removes local refs whose remote is gone
- `git fetch --prune` — same effect during a fetch
- `gh api repos/<owner>/<repo>/branches --paginate` — remote branch listing
- `gh pr list --state merged --json number,headRefName` — merged PR cross-reference
- `gh api -X DELETE repos/<owner>/<repo>/git/refs/heads/<branch>` — delete remote branch

## When to automate

The **audit half** (read-only survey) is well-suited to a scheduled GitHub Action — it can post a weekly issue or comment summarizing drift + stale candidates without any destructive action.

The **cleanup half** (delete worktrees, force-push, delete remote branches) is best kept human-supervised, at least until the audit has run reliably for several cycles and false-positive rates are understood.

A typical automation:

```yaml
# .github/workflows/branch-hygiene-audit.yml — runs weekly
on:
  schedule: [{ cron: "0 14 * * 1" }]   # Mondays 14:00 UTC

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with: { fetch-depth: 0 }
      - run: |
          # count drift, list stale branches across configured repos
          # post results as comment on a tracking issue
```

The implementation lives at `.github/workflows/branch-hygiene-audit.yml` in `bytetalent/bt-ai-web`. It runs on the `cron: "0 14 * * 1"` schedule (Mondays 14:00 UTC), is also `workflow_dispatch`-able for manual testing, and posts results as a comment on a `hygiene-tracking`-labeled issue (creating the issue on first run).
