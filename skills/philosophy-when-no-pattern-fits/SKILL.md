---
name: Philosophy — When No Pattern Fits
description: Heuristics for novel features that don't match any named pattern — data-first, standards-anchored, candidate-pattern flagging.
category: meta
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase, astro-cloudflare]
version: 1
---

This skill should always be loaded as low-priority context. It applies when `module-planning` has checked `docs/patterns-inventory.md` and found no matching pattern for the current feature.

## Step 1 — Start with the data shape

Define the table(s), column types, and RLS policies before writing any route or component. The schema is the contract everything else derives from. See `philosophy-feature-construction` for the full reasoning.

## Step 2 — Follow the standards skills

Before writing any code, identify which standards skills apply to this feature. Every feature touches at least some of:

- `typescript-strict` — type safety rules
- `data-access-patterns` — Drizzle as the only DB path; `resolveAccount` for auth
- `error-handling-nextjs` — `apiError()` for route error handling
- `rls-policies` — row-level security on every new table
- `nextjs-component-conventions` — naming, co-location, "use client" placement
- `tailwind-and-styling` — design tokens, no hardcoded colors
- `server-actions-auth` — auth guard on every mutation

These are non-negotiable regardless of whether a pattern applies. Novel features do not get to skip standards.

## Step 3 — Build the smallest working slice first

See `philosophy-feature-construction` for the definition of "smallest working slice." In the no-pattern context this means being disciplined about what is NOT in the first slice:

- Not the filtering UI (add later once the basic list works)
- Not the bulk actions (add later)
- Not the advanced empty states (a simple "No items yet" is enough for slice 1)
- Not the confirmation dialog for the destructive action (add it before the slice ships — safety gates are not deferred)

## Step 4 — Iterate

Once the smallest slice is working:

1. Add error recovery (retry buttons, error banners — see `fetch-error-retry` pattern)
2. Add the secondary actions (filters, bulk actions, sort)
3. Add tests for any non-trivial logic introduced in iteration
4. Refactor the page-client into sub-components if it exceeds ~200 lines

## Step 5 — Flag as a candidate pattern

After completing a novel feature (no pattern matched), write a candidate-pattern record to `docs/learnings/candidate-patterns.md`. The record documents the feature so the team can identify when the same shape recurs and extract it as a named pattern.

### Candidate-pattern record format

```markdown
## [date] [short feature name]

**Feature description**: one sentence describing what the feature does

**Skills composed**: list every skill used (e.g. `typescript-strict`, `data-access-patterns`, `rls-policies`)

**What was hard / what required judgment calls**: bullet list — the things that weren't obvious from the standards skills alone

**What felt repeated / familiar**: anything that looked like an existing pattern but didn't quite fit; what would need to change in the existing pattern to cover this

**Candidate pattern name** (if one emerges): e.g. `inline-editable-list-item`, `multi-tenant-settings-section`

**Recurrence threshold**: flag this for review after \_\_ more similar features (suggest 2–3)
```

Write a new `##` block per novel feature. Do not create one file per feature — append to the shared file. If `docs/learnings/candidate-patterns.md` does not exist, create it.

## What NOT to do

- Do not invent a new abstraction because the feature feels special. Novel ≠ needs a new framework.
- Do not skip standards skills because no pattern matches. The pattern library is about structure; the standards skills are about correctness. Both apply.
- Do not write the candidate-pattern record as a task to do later. Write it when the module is done, as part of closing out the module.
