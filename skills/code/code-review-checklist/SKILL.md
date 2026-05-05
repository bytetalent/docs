---
name: Code Review Checklist
description: Structured review categories, severity rubric, output format, and cadence for PR reviews; covers maintainability, design, performance, and version audits.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

## Review categories

Each PR review covers these categories in order. Security and auth reviews are in the `security-review` skill.

### Maintainability

- Files exceeding ~150 lines (or ~300 lines for complex page-client files) should be split
- `comp-` prefix and kebab-case file naming must be followed (see `nextjs-component-conventions` skill)
- Flag any ad-hoc naming that breaks the domain entity vocabulary (e.g., `History` vs. `Runs`, `Integrations` vs. `Connections`)

### Dead code

- Flag unused imports
- Flag unexported functions with no callers
- Flag zero-consumer components (components defined but never rendered)

### Performance

- Flag `"use client"` on components that have no state, no hooks, and no browser event handlers — they should be Server Components
- Flag client-side data fetches that could be server-component renders
- Flag missing `cache` annotations on repeated reads

### Design

- Flag every hardcoded hex color, `rgb()`, or `rgba()` value — replace with a CSS variable or Tailwind token
- Flag bare magic-number spacing (e.g., `py-20` without a responsive pair) — see `marketing-layout-patterns` skill
- Flag any component missing `dark:` variants on custom background, border, or text colors
- See `design-tokens-and-theming` skill for full token rules

### Component consolidation

- Flag near-duplicate UI patterns — components with identical layout but different hard-coded data — as consolidation candidates
- Propose a shared component with props rather than two near-identical implementations

### XSS

- Flag `dangerouslySetInnerHTML` — always requires an explicit safe-origin comment or sanitization
- Flag `href` attributes built from user-controlled strings — validate before use

## Version audit (required in every review)

Check all framework and key library versions against current stable releases. Report each in the review table with an upgrade priority:

| Priority    | Meaning                                                |
| ----------- | ------------------------------------------------------ |
| Now         | Security fix or API-breaking change in latest stable   |
| Next sprint | Deprecation, meaningful perf gain, or 2+ majors behind |
| Defer       | Minor or patch bump with no functional impact          |

## Output format

All reviews are written as a **prioritized table** sorted by severity (Critical → High → Medium → Low):

| Severity | File (line)                                      | Issue                          | Recommended fix                         |
| -------- | ------------------------------------------------ | ------------------------------ | --------------------------------------- |
| Critical | `src/app/api/projects/route.ts:42`               | No `If-Match` check on DELETE  | Add 428 guard per API conventions       |
| High     | `src/features/projects/comp-project-card.tsx:18` | `bg-[#1a1a2e]` hardcoded color | Replace with `bg-background`            |
| Medium   | `src/features/flows/comp-flow-list.tsx`          | File is 420 lines              | Extract filter panel into sub-component |
| Low      | `src/features/runs/comp-run-table.tsx:5`         | Unused import `Badge`          | Remove                                  |

Always include the file path and line number when available.

## CI pipeline — required steps on every PR

All of these must pass before merge:

1. Lint (`eslint`)
2. Type-check (`tsc --noEmit`)
3. Format check (`prettier --check`)
4. Unit tests (`vitest run`)
5. Build (`next build`)

E2E tests run on a separate cadence (schedule or pre-release), not on every PR.

## Review cadence and artifacts

- Dated review files: `docs/app-review-YYYY-MM-DD.md`
- Open items from a review are tracked in `docs/app-plan.md` under the relevant release section
- A new review file is opened at least once per sprint; items resolved before the next review are marked closed inline
