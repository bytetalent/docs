---
name: Philosophy — Feature Construction
description: Core principles for building a feature when no pattern fits — data-first, smallest-slice-first, deferred polish.
category: meta
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase, astro-cloudflare]
version: 1
---

This skill should always be loaded as low-priority context. It is the general reasoning fallback for novel features that don't match any named pattern in `docs/patterns-inventory.md`.

## Start from the data shape

Before writing any UI or route, define the data. Ask:

- What table(s) hold this feature's state?
- What are the column types, nullability, and constraints?
- What RLS policies govern who can read and write it?

Write the Drizzle schema definition first (even if rough). Let the schema drive the API surface — not the other way around. A route that references a column that doesn't exist yet is a dead end; a table definition that doesn't serve a route is a signal the schema is wrong.

## The smallest working slice

A "smallest working slice" is the minimum code that:

1. Stores and retrieves the feature's data correctly
2. Surfaces it to the right user with correct auth
3. Lets the user take the primary action the feature exists for

It is **not** the code that handles every edge case, loading state, empty state, or error variant. Those come next — but only after the slice works end-to-end.

Concretely for a CRUD feature, smallest working slice means:

- Schema + migration + RLS
- One route handler (GET list or GET detail + POST/PUT)
- One hook that fetches and exposes the data
- One page component that renders the data and the primary action

No bulk actions. No filters. No pagination (unless the dataset is unbounded). No empty-state illustration. Add those in iteration once the slice is confirmed to work.

## When to add error handling

Add error handling at two points — no earlier, no later:

1. **At the API boundary** — every route handler must wrap its logic in `try/catch` and call `apiError()` on the catch (see the `api-error-handler` pattern). This is never deferred.
2. **At the user action** — any mutation the user triggers (save, delete, approve) must show a toast on failure. Use `appToast.error(...)`. This is also never deferred.

Everything else (network retries, optimistic rollback, stale-data banners) is iteration. Don't add them to the first slice unless the feature is specifically about resilience.

## When to write a test first vs. last

Write a test **first** when:

- The feature involves a non-trivial calculation, state machine, or transformation (e.g., status transitions, ETag conflict logic, permission checks)
- The feature modifies data in a way that is hard to visually verify

Write a test **last** (or omit and flag) when:

- The feature is a thin CRUD wrapper with no logic beyond "store this, retrieve that"
- The test would be an integration test against a real DB, and the scaffold doesn't have that harness yet

Either way: never skip typecheck and lint. `npm run typecheck` and `npm run lint` must pass before the slice is considered done.

## Ordering within a module

When no pattern specifies otherwise, follow this default order:

1. Drizzle schema (table + enums + relations)
2. Migration SQL (`drizzle-kit generate` — never hand-write)
3. RLS policies (in the same migration file)
4. Route handlers (list route, then detail route)
5. Domain hook(s) (`use-<entity>-list` or `use-<entity>-detail`)
6. Page components (`page.tsx` thin server wrapper + `page-client.tsx`)
7. Extracted sub-components (only when the page-client grows beyond ~200 lines)

This is the same sequence as the `pipeline-patterns` skill's "code-gen module sequence." This skill restates it as philosophy because it's the right order for reasons, not just convention: each step provides the foundation the next step needs.

## What "done" means for a module

A module is done when:

- All parts listed above are present (no partial modules)
- `npm run typecheck` passes against the whole module's diff
- `npm run lint` passes
- `npm run test` passes (or tests are explicitly flagged as out of scope with a comment)
- The PR contains every part of the module — no follow-up PRs to "finish" it
