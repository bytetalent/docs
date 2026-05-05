---
name: Philosophy — Composition Over Invention
description: Prefer composing existing skills + small glue code over inventing new abstractions — prove need with the third use case, not the first.
category: meta
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase, astro-cloudflare]
version: 1
---

This skill should always be loaded as low-priority context. It governs when to compose vs. when to create something new.

## The core rule

If two or more existing skills or patterns cover 80%+ of what a feature needs, compose them — do not write a third generalization.

The code that bridges the two skills (the 20%) is fine to write fresh. That's glue code, not a new abstraction. What's not fine is wrapping the whole thing in a new hook, util, or component "for reuse" before there is a second use case.

## When a new abstraction is justified

A new abstraction is justified when **three** features have shared the same shape without an existing skill covering it.

- Two instances = coincidence (or a candidate pattern in `docs/learnings/candidate-patterns.md`)
- Three instances = extract it

Until the third instance exists, copy-and-adapt is the right answer. The cost of a premature abstraction (fragile generalization, wrong seams, over-engineered API) exceeds the cost of two copies.

This is the "rule of three" applied specifically to Bytetalent code generation. It overrides the instinct to DRY immediately.

## What "composing skills" looks like in practice

Given a feature that needs:

- A list page with server-side auth
- Inline row editing with optimistic updates

You do NOT write a new `useInlineEditableList` hook. Instead:

1. Apply `entity-list-hook` pattern for the list and selection state
2. Apply `async-toast-mutation` pattern for the per-row save
3. Apply `optimistic-crud-with-etag` if the entity needs version guarding

The glue: the row component calls the mutation handler from the page-client's save handler. That's ~20 lines. That's the new code.

## Cross-references

- When a feature has been composed this way three times and you're ready to extract, see issue #242 (patterns : validation : reverse-test) for the process of verifying that an extracted pattern actually reconstructs all three instances correctly.
- For deciding whether the feature needs a pattern vs. philosophy approach, see `module-planning`.

## What NOT to do

- Do not create a new shared hook for a use case covered by an existing hook with a slightly different type parameter. Use the existing hook.
- Do not create a new component for a layout variation that can be expressed via props on an existing component.
- Do not create a new utility file for a one-line transformation. Inline it.
- Do not write an abstraction and then file a "TODO: wire this to the second use case" comment. Build what's needed now; extract when the second use case arrives.

## On skill authoring (meta)

This principle also applies to skill files themselves. Do not write a new SKILL.md that largely restates an existing one with one paragraph changed. If a skill needs an addendum, add it to the existing SKILL.md under a new `## Addendum` section (versioning the file). Extract a new skill only when the concern is genuinely orthogonal to the existing skill's scope.
