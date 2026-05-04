# Memory seeds — durable agent memory, git-tracked

Claude Code maintains a per-project memory directory at:

```
~/.claude/projects/<encoded-working-directory-path>/memory/
```

That directory is **path-encoded**, meaning if you move the project (rename, reorganize, fresh-clone on a new machine), the memory dir is effectively orphaned — a fresh empty one is created at the new encoded path, and everything previously accumulated is stranded under the old name.

This `memory-seeds/` directory is the durable home for memory entries that should survive that move. It's also git-tracked + reviewable, so the team has visibility into the operating rules the agent has been calibrated against.

## Inheritance model

Memory seeds compose by layer the same way skills, templates, and code do. A generated app's memory is reconstituted by **stacking** all applicable layers:

```
universal       → bytetalent/docs/agent-framework/memory-seeds/    (this dir; any agentic project)
base            → bytetalent/<base-repo>/memory-seeds/              (any app built on this base)
template        → bytetalent/templates/<template>/memory-seeds/     (apps that include this template)
product         → <product-repo>/docs/ops/claude-context/           (this specific product)
```

When a session starts, the agent's effective memory = `universal ∪ base(s) ∪ templates_in_use ∪ product`. More-specific layers can extend or override more-universal ones, but the universal layer is always loaded.

For a non-generated project (e.g., `app-flow.ai` itself), the chain is just `universal + product`. For a generated client app produced by app-flow.ai, the chain is `universal + base(s) + each_included_template + product`.

## What's a "seed"

A coalesced operating-instruction document with the same shape as any structured doc. Each doc covers a related cluster of rules. Format:

```markdown
# <topic>

<intro — what this doc covers>

## <rule heading>

<one-line rule statement>

**Why:** <reason — usually an incident or stable preference>
**How to apply:** <concrete behaviors that follow from the rule>

## <next rule heading>
...
```

Earlier iterations stored each rule as its own `feedback_<name>.md` file with frontmatter. We've consolidated into thematic docs because related rules cross-reference, share context (incident dates, prior reasoning), and read more naturally as a manual than a stack of post-its.

## What goes here vs the active memory dir

| Where | What | Lifecycle |
|---|---|---|
| `~/.claude/projects/<encoded>/memory/` (active) | All memory — durable + ephemeral, fresh observations, in-flight project state | Read every conversation; mutated frequently |
| `bytetalent/docs/agent-framework/memory-seeds/` (this dir) | Universal feedback rules + durable principles | Reviewed in PRs; updated when rules evolve |
| `<product-repo>/docs/ops/claude-context/` | Product-specific durable context (env topology, schema, role model) | Reviewed in PRs; promotes from active when stable |
| `<base-repo>/memory-seeds/` (future) | Base-specific (stack, conventions) | Inherited by all generated apps on this base |
| `<templates>/<template>/memory-seeds/` (future) | Template-specific (data-binding rules, integration gotchas) | Inherited by apps that include the template |

The active memory dir is the **working overlay**. Seeds are the **base layer** that re-instantiates whenever the working overlay needs to be rebuilt.

## Rehydration — re-seeding a fresh memory dir

After a path move, machine swap, or new contributor onboarding:

```bash
# 1. From the new working directory, find the path-encoded memory dir Claude Code created:
ls ~/.claude/projects/ | grep <some-fragment-of-new-path>

# 2. Copy seeds into it
cp -r bytetalent/docs/agent-framework/memory-seeds/*.md \
      ~/.claude/projects/<new-encoded-path>/memory/

# 3. Also copy product-specific seeds if you have them
cp -r <product-repo>/docs/ops/claude-context/*.md \
      ~/.claude/projects/<new-encoded-path>/memory/

# 4. Regenerate MEMORY.md index. Each doc gets one line; titles + topics from the doc itself.
```

After rehydration, the next Claude Code session at the new path picks up all the seeded content automatically.

## Promotion workflow — active memory → seeds

When a new feedback rule, project decision, or reference fact emerges in conversation and gets saved to active memory, it lives there until validated as durable. Periodically (during cleanup / housekeeping passes), promote durable entries to seeds:

1. Pick an active-memory entry that's proven worth keeping (referenced multiple times across sessions, codifies a stable rule, captures a non-obvious invariant).
2. Decide layer: universal (here), base, template, or product?
3. Find the matching consolidated doc in the chosen layer (e.g., a new git-workflow rule joins `multi-agent-git-workflow.md`).
4. Add a section in the doc with the same rule / **Why** / **How to apply** structure.
5. Open a PR with the addition — peer review acts as the durability check.
6. After merge, the active-memory entry can stay as-is or be deleted; on next rehydration, the seeded doc is the source of truth.

## What does NOT belong as a seed

- **Ephemeral state** — "morning agenda for 2026-05-04", "redesign in flight", "stale-open triage as of <date>". These decay too fast.
- **Code conventions** — anything derivable from reading the codebase, CLAUDE.md, or skill SKILL.md files. Memory is for things the code itself doesn't reveal.
- **Git history facts** — `git log` is authoritative; don't seed who-changed-what summaries.
- **Workflow snapshots** — board state, open-PR counts, in-flight wave status. These belong in dashboards, not seeds.
- **User memories** — personal facts about the human (their role, knowledge, preferences). Not portable to a shared seed library.

## Current docs (universal layer)

- `collaboration-style.md` — how to talk + plan + react with the user
- `engineering-discipline.md` — quality + safety standards
- `tooling-permissions.md` — what to autorun vs what needs confirmation
- `multi-agent-git-workflow.md` — branches, worktrees, PRs, backports for parallel-agent work
- `github-board-operations.md` — GraphQL efficiency + project board mechanics

## Related

- Strategy: `bytetalent/docs/strategy/agent-framework.md` — channel 3 framework framing; memory seeds are part of the asset bundle
- Skill (future): `bytetalent/docs/skills/auto-memory/SKILL.md` — the authoring side of memory entries
