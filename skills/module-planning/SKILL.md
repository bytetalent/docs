---
name: Module Planning
description: Per-module planning workflow — check patterns, compose if multiple apply, fall back to philosophy, flag novel features as candidate patterns.
category: meta
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase, astro-cloudflare]
version: 1
---

This is the meta-skill that governs how the code-gen agent plans each module before writing any code. Run this planning workflow at the start of every module. It is consumed by the R1-5c module-planner runtime.

## The 4-step planning workflow

### Step 1 — Check patterns first

Open `docs/patterns-inventory.md`. Read the 21 named patterns. Ask: does this feature's shape match any of them?

A "match" means the feature is structurally the same shape — same lifecycle, same UI chrome, same data-flow direction — even if the domain content differs.

Examples of matches:

- "A list of entities with filters and bulk actions" → `paginated-data-table` + `entity-list-hook`
- "A multi-step linear user journey" → `wizard`
- "An AI generation loop with polling" → `async-job-with-banner` + `phase-editor`
- "A detail page with multiple views" → `detail-page-with-tabs`
- "An update that must guard against concurrent edits" → `optimistic-crud-with-etag`

If one pattern matches, load that pattern's SKILL.md (once authored — tracked in #240) and plan the module using it. Skip to step 4.

If no pattern matches, proceed to step 2.

### Step 2 — If multiple patterns apply, compose them

Some features are composites. A feature might be a `detail-page-with-tabs` where one of the tabs is a `paginated-data-table`. That's two patterns composing, not a new pattern.

When multiple patterns apply:

1. Identify the primary structural pattern (the outer shape of the feature)
2. Identify the secondary patterns (shapes that appear within sections of the feature)
3. Note where the patterns join — the glue code at the seam

Apply `philosophy-composition-over-invention`: do not invent a third generalization. Write the glue; let each pattern govern its own section.

If the composition feels awkward (patterns conflict, or the glue exceeds ~20% of the total module), treat it as "no pattern fits" and proceed to step 3.

### Step 3 — If no pattern fits, fall back to philosophy + general reasoning

Load these three skills as the planning foundation:

- `philosophy-feature-construction` — core principles (data-first, smallest-slice-first, ordering)
- `philosophy-when-no-pattern-fits` — step-by-step heuristics for novel features
- `philosophy-composition-over-invention` — guard against premature abstraction

Then plan the module from first principles:

1. Define the data shape (tables, columns, RLS)
2. Identify which standards skills apply (these are always active regardless of pattern)
3. Lay out the build order (schema → migration → RLS → routes → hooks → pages → components)
4. Identify the smallest working slice and what is deferred to iteration

Proceed to step 4.

### Step 4 — Flag novel features as candidate patterns

After completing a module where no pattern matched (step 3 was used), write a candidate-pattern record.

**Where to write it**: `docs/learnings/candidate-patterns.md`

If the file does not exist, create it with this header:

```markdown
# Candidate Patterns

Features generated with no matching pattern. Reviewed quarterly; 3+ similar candidates = extract a new named pattern.

---
```

Then append a new record block (see `philosophy-when-no-pattern-fits` for the exact record format).

**When to write it**: immediately after the module's PR is opened, not as a follow-up task.

**Threshold for extraction**: when 3 or more candidate records describe the same structural shape, that shape is ready to be extracted as a named pattern (via the patterns : authoring epic, #238).

## Standards skills are always active

Patterns and philosophy skills govern **structure**. Standards skills govern **correctness**. Both are always active — a pattern match does not exempt the module from standards.

Regardless of which path (pattern / philosophy) the module follows, the code must satisfy:

- `typescript-strict` — no `any`, Drizzle-inferred types
- `data-access-patterns` — Drizzle + `resolveAccount` + `apiError`; no parallel DB paths
- `rls-policies` — every new table has RLS before its route ships
- `error-handling-nextjs` — `apiError()` in every route's catch block
- `nextjs-component-conventions` — `comp-<domain>-<name>.tsx` naming; co-location rules
- `tailwind-and-styling` — design tokens; no hardcoded hex

## What this skill does NOT do

- It does not specify the within-module construction order in detail — that's `philosophy-feature-construction` and the individual pattern skills.
- It does not decide which module to build next — that's the architecture phase's `module_sequence` field (see issue #237).
- It does not define what counts as a valid pattern — that's `docs/patterns-inventory.md` plus the patterns : authoring work (#240).
