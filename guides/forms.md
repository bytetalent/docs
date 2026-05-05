# Forms — Standards

Cross-stack form patterns. The canonical rules live in [`skills/code/form-patterns`](../skills/code/form-patterns/SKILL.md) and [`skills/api/api-input-validation`](../skills/api/api-input-validation/SKILL.md). This guide covers the philosophy behind those rules and when to reach for which approach.

Stack-specific form primitives (React + Tailwind, Astro forms, ASP.NET Razor inputs) live in each base/product's overlay.

---

## Why no form library by default

Bytetalent webapps don't use a form library (no `react-hook-form`, no Formik). The principle is **clarity over indirection** — most forms are simple enough that a library adds boilerplate without saving anything. For complex multi-step or conditional forms, an extracted custom hook keeps the page component clean without pulling in a third-party abstraction.

If you find yourself implementing `react-hook-form` features by hand (deep validation, field arrays, dirty tracking), reconsider — but at that point use Zod's `parse` + a thin custom hook, not a third-party library.

## Why inline-visible validation (not submit-time errors)

Showing validation rules as pass/fail indicators from the moment a field is focused turns validation into a guide rather than a punishment. Disabling the submit button until all rules pass eliminates the submit-then-see-red-errors flash. This is the user experience we target; the mechanics are in the skill.

## When to extract a custom hook

Extract form state into a custom hook when the form has > 5 fields, conditional field visibility, or multi-step flow. The hook owns all field state, `isDirty`, `isValid`, and a `reset` function — keeping the page component readable and the form testable in isolation.

## Server-side mirrors client-side (always)

Client-side rules are convenience — they prevent useless round-trips. Server-side validation via the same Zod schema is the actual security boundary. Every client-side constraint must be mirrored server-side.

---

## Canonical rules and patterns

**Form layout, field anatomy, submit states, error surfacing, validation patterns, forbidden patterns:** see [`skills/code/form-patterns`](../skills/code/form-patterns/SKILL.md).

**Server-side validation (Zod schema, request body parsing):** see [`skills/api/api-input-validation`](../skills/api/api-input-validation/SKILL.md).

**Accessibility (label linkage, `aria-live`, keyboard navigation):** see [`skills/design/accessibility-aa`](../skills/design/accessibility-aa/SKILL.md).

---

## Related

- [`guides/design.md`](design.md) — input tokens, button states, ConfirmDialog
- [`guides/api.md`](api.md) — server-side validation pattern, error response shape
- [`guides/code.md`](code.md) — TypeScript discipline, no-`any` rule
