# Multi-agent git workflow

Branches, worktrees, PRs, and promotion patterns when multiple agents (Claude Code instances + Terminal-based contributors + human reviewers) operate on the same repos in parallel. The rules in this doc share a thread: **isolation is the discipline that keeps parallel work non-destructive.**

For repo-specific branch policy (which branches exist, which use squash vs rebase merge, what's protected), see the product's own conventions doc — those vary per repo. Rules here apply universally.

## PR auto-merge by default

When opening a PR with `gh pr create`, follow up with `gh pr merge <num> --auto --merge` (or `--squash` for tiny cleanup PRs) so the PR merges automatically once requirements are met. Repos should have `allow_auto_merge: true`.

**Why:** User asked on 2026-04-28 — "when you create pr's can you set them to auto merge?" Reduces manual click-merge overhead, especially for tiny diagnostic / cleanup PRs that don't need review.

**How to apply:**
- Default: enable auto-merge on every PR opened from feature → integration branch.
- Use merge-commit method for substantial PRs (preserves per-commit history) and squash for tiny diag/cleanup PRs.
- For staging → main (prod release), still enable auto-merge — CI is the gate, and main is the deployment artifact.
- **Exception:** if a PR is large or risky, mention "I'm not auto-merging this one — want you to review first" so the user has a chance to look. Default is auto-merge; opt out only when warranted.
- Note: with branch protection that has no required reviewers, auto-merge fires almost immediately on open. That's fine functionally; just be aware there's no review window for PRs that arguably should have one.

## Workers must use uniquely-named branches

When dispatching multiple workers in parallel — especially on related issues in the same target repo — make sure each worker's branch is uniquely named. If two workers create or push to a branch with overlapping naming (e.g., `claude/332-marketing-faq-accordion` and `claude/332-...`), one worker's push can overwrite the other's commits, and the merging PR may carry the WRONG content while the original work is silently lost.

**Why (incident, 2026-05-04):** worker-template-332 (faq-accordion) and worker-template-351 (email-resend-integration) both ended up working on `claude/332-marketing-faq-accordion`. The faq-accordion worker pushed first and opened PR #5 with title "[R1.4] marketing-faq-accordion template." The email-resend worker pushed AFTER (their report noted "the branch had been claimed") and their commits became the merged PR content. PR title said faq-accordion, merged content was email-resend. #332 auto-closed (because PR title "closes 332") but the actual code never landed. Caught only when verifying file system state. Required a cleanup PR (#6) cherry-picking the surviving faq-accordion commit from the leftover branch back into a fresh branch off main.

**How to apply:**
- **Worker brief should specify the EXACT branch name**, not "claude/<issue-id>-<slug>" templated. Pre-compute the branch name in the orchestrator and put it in the brief.
- **Branch names should include the issue number AND a unique-per-attempt suffix** if the issue has been worked before. Format: `claude/<N>-<slug>` is fine for first attempt; `claude/<N>-redo-<date>` for re-dispatches.
- **When two workers operate on related issues in the same repo**, give them branches that can never collide even by typo. Don't rely on workers picking sensibly.
- **Worker briefs should cite the existing branch list** so workers can detect a name they shouldn't take. If "the branch is already claimed," **STOP and message the lead** — do not push onto someone else's work.
- **Verify the merged PR's content matches its title** in post-merge verification. Don't just check that the issue auto-closed — auto-close from "closes #N" can fire even when content is wrong.

**Mitigations to consider for the future:**
- Spawn workers with a `branchName` parameter explicitly in the dispatch (not derived from issue body)
- Have the orchestrator pre-create empty branches with reserved names so workers fast-fail if collision
- Wave-review post-merge should verify file system state, not just rely on PR titles closing issues

## Workers must stay strictly inside their assigned worktree

When dispatching multiple parallel workers via the Agent tool with `isolation: "worktree"`, the runtime auto-creates a worktree at `.claude/worktrees/agent-<id>/`. Workers MUST operate strictly there — they must never `git checkout` a branch in the parent repo's main worktree, never run mutating git commands from the parent dir, and never write files into the parent worktree.

**Why (incident, 2026-05-03):** worker-218 (portal share) reported "significant shared working-tree contamination from parallel workers (`claude/377-380-r2-5a-multi-account`)" while 4+ workers ran in parallel. They had to use a temporary isolated worktree at `C:/Windows/Temp/wt-218` plus `git cat-file blob` + `git update-index --cacheinfo` to stage exact content because the main worktree had pollution from another worker's branch checkout. Cost: ~10 extra min of recovery work.

**How to apply:**
- **In every dispatch brief**, explicitly state: "operate ONLY inside your assigned `.claude/worktrees/` worktree; do NOT `git checkout` or modify files in the parent repo's main worktree."
- **Use `Agent({isolation: "worktree"})`** (already the default for implementer agents) so the runtime creates and manages the worktree.
- **If a worker reports working-tree contamination**, the right fix is the worker creates its OWN temp worktree — not blame other workers. Workers running concurrently must be defensive about their own scope.

**For future improvements:**
- Have the implementer agent prelude include a built-in invariant: "if `git status` in the main worktree shows files you didn't write, do not stage or commit; create your own worktree for the operation."
- Consider serializing the final-mile (PR open + status flip) phase if working-tree races are recurring.

## Backport staging-unique fixes to dev before promoting

When a fix lands on `staging` (or `main`) but not on `dev` — typically a CI repair or a hotfix applied directly to the promotion branch — that fix has an unstable position. The next `dev → staging` promotion can clobber it because:

- Dev still has the old broken file (the fix only went to staging).
- A `git merge -X theirs origin/dev` from staging's side prefers dev's content for conflicts and silently drops the staging-side fix.
- A standard rebase-merge replays dev's old commit on top of staging's fix, also reverting it.

CI then fails the same way it did before staging got fixed. Whack-a-mole.

**Why (incident, 2026-05-03):** PR #525 (dev → staging). Staging had `5136503 [ci-fix] pipeline-rls.test : update for brand_kits account-level scoping` AND a prettier sweep that dev hadn't received. A `-X theirs` merge dropped both fixes; CI failed format check + the same pipeline-rls test, both of which staging already had green. Had to port both back to dev (PRs #528 + #529) AND amend the merge commit on the promotion branch.

**How to apply:**

Before opening a `dev → staging` PR (or any rebase-merge promotion):

1. **List staging-unique commits:** `git log --oneline origin/staging ^origin/dev` — anything here is at risk.
2. **Identify content-unique vs SHA-drift:**
   - SHA-drift = same PR title/work as dev, just at a different hash from a prior rebase-merge. Safe to ignore (git's patch-id detection during rebase will skip them).
   - Content-unique = commits that *don't have a matching PR title on dev*. These are the ones to backport.
3. **Cherry-pick or port** those content-unique commits to dev FIRST as small PRs (auto-merge fine). Then open the promotion PR.
4. **For inevitable promotion-time conflicts:** when the merge has the wrong file content, prefer staging's version for files where staging has a known fix that dev lacks. `git checkout origin/staging -- <path>` then amend.

Heuristic for spotting "staging-unique fix" commits: titles starting with `[ci-fix]`, `[infra]`, `[fix]`, `[hotfix]`. These almost always need backporting.

**Long-term mitigation:** after every `staging → main` rebase-merge, realign by rebasing dev onto staging. Doing that consistently keeps dev fully ahead of staging in all dimensions, so this class of issue evaporates. See the `branch-and-worktree-hygiene` skill for the realign procedure.

**Merge strategy on the promotion PR itself:** Use **squash merge**, not rebase merge, when the promotion branch is a "merge dev into staging-base" branch (single merge commit at tip). Convention says rebase for dev → staging but assumes a clean linear feature branch. Promotion branches encapsulate dev's state via a merge commit — rebase-merge struggles with that shape (PR #525 hit it; PR #543 also). Squash creates one clean diff-commit on staging; loses commit-level granularity on staging side but the dev-side history is preserved.
