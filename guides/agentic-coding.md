# Agentic coding — architecture and discipline

How the Bytetalent framework operates, how lessons move through it, and what every agentic codebase carries.

This is the HOW counterpart to [`strategy/agent-framework.md`](../strategy/agent-framework.md), which explains the WHY (channels, DMAIC loop, monetization model). Read that first if you want the strategic frame.

---

## The layer cake

Every project that uses this framework operates against a layered stack of standards. Layers are loaded from widest to narrowest. A single rule might appear in multiple layers — that's intentional, because mechanical enforcement and readable documentation serve different audiences.

```
┌─────────────────────────────────────────────────────────────────┐
│ Tooling  (CI, pre-commit, lint rules, type guards)              │
│   → catches violations mechanically before they can merge       │
├─────────────────────────────────────────────────────────────────┤
│ Guide  (bytetalent/docs/guides/*.md)                            │
│   → explains the "why" — humans + agents researching a pattern  │
├─────────────────────────────────────────────────────────────────┤
│ Skill  (bytetalent/docs/skills/<area>/SKILL.md)                 │
│   → copy-pasteable pattern an agent applies per-task            │
├─────────────────────────────────────────────────────────────────┤
│ CLAUDE.md  (repo root)                                          │
│   → repo-specific deviations; always loaded for this codebase   │
├─────────────────────────────────────────────────────────────────┤
│ Personal memory  (~/.claude/projects/<encoded>/memory/)         │
│   → how this specific human works — preferences, rituals        │
└─────────────────────────────────────────────────────────────────┘
```

| Layer | Audience | When it kicks in |
|---|---|---|
| **Tooling** | Everyone, mechanically | Always — non-bypassable |
| **Guide** | Humans + agents | When researching "how do we do X?" |
| **Skill** | Agents in context | Auto-loaded by phase/stack |
| **CLAUDE.md** | Agents in this codebase | Always loaded for the repo |
| **Memory** | Me, across sessions | Where applicable |

Layers are additive, not exclusive. A single root cause can warrant a CI check (tooling), a guide section explaining the underlying model (guide), a skill callout for agents (skill), and a repo-specific note if there's a local deviation (CLAUDE.md).

**When a new lesson appears, the question is not "where does this go?" but "what's the most mechanical layer that prevents recurrence?"** Start at tooling and work down.

---

## Lesson capture — from incident to framework artifact

The capture flow runs after any debug session that identifies a root cause:

```
1. Can it be detected mechanically (CI, pre-commit, lint, type guard)?
      YES → ship the check AND note it in the relevant skill
      NO  → continue

2. Is the pattern universal (applies across stacks and projects)?
      YES → update the skill in bytetalent/docs/skills/<area>/SKILL.md
            if complex enough to need explanation → also add a guide section
      NO  → update the consuming repo's CLAUDE.md

3. Is it about how this specific human works or communicates?
      YES → write a personal memory entry (feedback_<topic>.md)
      NO  → one-off; document in the PR description, no framework artifact needed
```

**Always start at step 1.** A three-line CI check that catches the error before a PR merges is worth more than a paragraph in a guide.

The full decision flow with worked examples is in [`skills/process/lessons-learned-discipline/SKILL.md`](../skills/process/lessons-learned-discipline/SKILL.md). The self-review tooling that runs it mechanically is in [`guides/agents.md`](agents.md) and sub-issue [#468](https://github.com/bytetalent/bt-ai-web/issues/468).

### When to write a guide section vs a skill callout

A skill callout (`**Pitfall:** see guide-db.md § "Journal when semantics"`) is a pointer. Write a full guide section when:

- The bug involves a non-obvious invariant in a third-party tool
- Correct behavior requires understanding a sequence of events, not just a rule
- Multiple agents have hit the same bug from different starting points

If the pointer is enough, prefer the pointer.

---

## What every generated repo carries — the agentic kit

Code-gen doesn't just scaffold code. It scaffolds **agentic capability**. Every generated client repo inherits:

| Capability | Where it ships | Sub-issue |
|---|---|---|
| Agent config (`.claude/settings.json`) | base scaffold | [#467](https://github.com/bytetalent/bt-ai-web/issues/467) |
| Project instruction (`CLAUDE.md`) | base scaffold | |
| Universal skill snapshot (`skills/_universal/`) | base scaffold | [#466](https://github.com/bytetalent/bt-ai-web/issues/466) |
| Self-review script (`bin/self-review`) | base scaffold | [#468](https://github.com/bytetalent/bt-ai-web/issues/468) |
| Lesson-capture script (`bin/lesson`) | base scaffold | [#469](https://github.com/bytetalent/bt-ai-web/issues/469) |
| Pre-commit gate (husky + lint-staged + `.gitattributes`) | base scaffold | |
| CI standards check | base GitHub Actions | |
| Upstream feedback channel | universal skill + cross-repo PR pattern | [#470](https://github.com/bytetalent/bt-ai-web/issues/470) |

**The `.claude/settings.json`** pre-approves the commands agents need to work autonomously (branch, commit, claim issues, open PRs). Without it, workers in isolated worktrees stall silently on the first restricted command — the worktree doesn't inherit the human's interactive session permissions. See [`guides/agents.md`](agents.md) for the canonical allow/deny matrix and the incident that motivated it (bt-ai-web PR #259).

**The skill snapshot** is a vendored copy of `bytetalent/docs/skills/` at a known commit. Agents working in the generated repo load skills from their local snapshot — no live dependency on this repo. A drift-check script (`npm run skills:check-drift`) detects when the snapshot is stale.

**`bin/self-review`** loads the applicable skills (by stack manifest) and runs a structured code review pass. It outputs findings by severity, file:line, and the skill that flagged the issue. `npm run review` invokes it. See [`skills/code-review-checklist`](../skills/meta/code-review-checklist/SKILL.md) for the review categories.

**`bin/lesson`** implements the lesson-capture flow above. It prompts for the routing decision and generates a starter PR diff or ticket body for the chosen layer. Cross-repo PRs to `bytetalent/docs` require the upstream feedback channel (#470) to be operational.

---

## Upstream feedback — how lessons flow back

```
                    bytetalent/docs (universal)
                  guides + skills + tooling
                            │
              ┌─────────────┼─────────────┐
              │             │             │
       app-flow-ai-web   bases (3)    templates
              │             │             │
              └──── code-gen ──────────────┘
                            │
                      ┌─────▼──────┐
                      │ generated  │
                      │ client repo│
                      │ (agentic)  │
                      └─────┬──────┘
                            │
              proposes lessons back upstream
                            │
                  bytetalent/docs (universal)
```

Lessons don't only flow downward. A generated client repo that discovers a reusable pattern can propose it back to `bytetalent/docs` via a cross-repo PR. The flow:

1. Agent in the generated repo runs `bin/lesson` and routes the finding as "universal skill".
2. `bin/lesson` opens a PR against `bytetalent/docs` (authenticated via the upstream feedback channel).
3. Bytetalent staff reviews. If merged, the lesson propagates to all future generated repos.

This requires a service account or GitHub App with write scope on `bytetalent/docs`. The mechanics, rate-limit concerns, and stewardship questions are tracked in [#470](https://github.com/bytetalent/bt-ai-web/issues/470).

### Cross-repo PR pattern

When a worker in one repo needs to propose a change to a sibling repo:

1. Fetch + worktree the sibling repo at a known-good commit.
2. Branch using the pre-computed unique branch name from the dispatch brief.
3. Implement, push, open the PR against the sibling repo's `main`.
4. PR body must use `Closes owner/repo#N` (full form) — cross-repo close keywords require the full `owner/repo` prefix.
5. Clean up the worktree after the PR is open.

See the implementer agent definition (`.claude/agents/implementer.md`) for the full cross-repo dispatch workflow.

---

## Memory seeds — durable agent memory, git-tracked

The per-project memory directory (`~/.claude/projects/<encoded>/memory/`) is path-encoded. A machine swap or directory rename orphans it. **Memory seeds** are the git-tracked base layer that survives those moves.

Seeds compose by layer:

```
universal  →  bytetalent/docs/agent-framework/memory-seeds/
base       →  bytetalent/<base-repo>/memory-seeds/
template   →  bytetalent/templates/<template>/memory-seeds/
product    →  <product-repo>/docs/ops/claude-context/
```

The agent's effective memory at session start = `universal ∪ base ∪ templates_in_use ∪ product`. More-specific layers can extend more-universal ones; the universal layer is always loaded.

To re-seed a fresh memory dir after a path move:

```bash
cp -r bytetalent/docs/agent-framework/memory-seeds/*.md \
      ~/.claude/projects/<new-encoded-path>/memory/
cp -r <product-repo>/docs/ops/claude-context/*.md \
      ~/.claude/projects/<new-encoded-path>/memory/
```

Full details in [`agent-framework/memory-seeds/README.md`](../agent-framework/memory-seeds/README.md).

### Promotion — active memory → seeds

Active memory entries live in `~/.claude/projects/<encoded>/memory/`. When an entry proves durable (referenced across sessions, codifies a stable rule), promote it:

1. Pick the right layer (universal / base / template / product).
2. Find the matching consolidated doc in that layer.
3. Add a rule section: one-line rule, **Why:** line, **How to apply:** line.
4. Open a PR — peer review is the durability check.
5. After merge, the seeded doc is the source of truth on next rehydration.

---

## Self-review patterns

`bin/self-review` runs mechanically. But agents also have a lighter-weight self-review discipline that runs at key decision points:

- **Before opening a PR**: run the [code-review-checklist skill](../skills/meta/code-review-checklist/SKILL.md). Verify the change is scoped to what the issue asks. No new abstractions for hypothetical needs.
- **After a debug session**: run the lessons-learned decision flow. Don't close the ticket without routing the lesson.
- **After a wave completes**: run the wave-review protocol (10-PR threshold triggers a structured review; see the orchestrator's dispatch docs).

The self-review skill and the `bin/self-review` script are complementary: the script catches drift against skills mechanically; the skill disciplines the agent's judgment about what to do next.

---

## Dogfooding — app-flow-ai-web runs the loop

`app-flow-ai-web` is its own generated-repo-equivalent in this framework. It runs `bin/self-review`, uses `bin/lesson` when debugging itself, and feeds lessons back upstream to `bytetalent/docs`.

Sub-issue [#472](https://github.com/bytetalent/bt-ai-web/issues/472) tracks the formal application. The principle: if we don't dogfood the loop on the codebase we touch most, we won't notice when it breaks.

---

## Sub-issues and related reading

This guide is part of the agentic-framework epic (#464):

| Sub-issue | What it ships |
|---|---|
| [#466](https://github.com/bytetalent/bt-ai-web/issues/466) | Skill vendoring — universal skills snapshot in generated repos |
| [#467](https://github.com/bytetalent/bt-ai-web/issues/467) | `.claude/settings.json` template per base |
| [#468](https://github.com/bytetalent/bt-ai-web/issues/468) | `bin/self-review` — repos self-review against loaded skills |
| [#469](https://github.com/bytetalent/bt-ai-web/issues/469) | `bin/lesson` — lesson-capture protocol for generated repos |
| [#470](https://github.com/bytetalent/bt-ai-web/issues/470) | Upstream feedback channel — cross-repo PR to bytetalent/docs |
| [#471](https://github.com/bytetalent/bt-ai-web/issues/471) | This guide |
| [#472](https://github.com/bytetalent/bt-ai-web/issues/472) | Apply to app-flow-ai-web — dogfood the loop |

**Related docs:**

- [`strategy/agent-framework.md`](../strategy/agent-framework.md) — the strategic frame (channels, DMAIC loop, monetization)
- [`guides/agents.md`](agents.md) — `.claude/settings.json` standard; the silent-stall failure mode
- [`agent-framework/memory-seeds/README.md`](../agent-framework/memory-seeds/README.md) — seed inheritance model, rehydration, promotion workflow
- [`skills/process/lessons-learned-discipline/SKILL.md`](../skills/process/lessons-learned-discipline/SKILL.md) — the per-debug decision flow with worked examples
