# Design Standards

Cross-stack visual design and UX standards. Tokens, typography, theming, stat-card anatomy, marketing-page spacing, and CTA discipline.

Stack-specific component implementations (React + Tailwind + shadcn vs. Astro + Tailwind vs. native iOS) live in each base or product's `docs/` overlay.

---

# Part 1 — Design System

## 1–3. Design tokens, typography, and theming

For design token categories, typography (Inter + JetBrains Mono, icon system), the three-source theming model (localStorage → auth metadata → system), flash prevention, and cross-tab sync, see [`skills/design/design-tokens-and-theming`](../skills/design/design-tokens-and-theming/SKILL.md).

For the `useTheme()` hook implementation (three-priority reconciliation, `theme-init.js` beforeInteractive script), see the stack-specific overlay in `bytetalent-nextjs-clerk-supabase-base/docs/guide-stack.md` §12.

## 4–5. Stat card anatomy and UI component patterns

For stat card layout rules (label, primary value, context text, progress bar, spacing) and established UI patterns (collapsible chart card, dark chart panel, slide-in edit panel), see [`skills/design/stat-card-and-interactive-components`](../skills/design/stat-card-and-interactive-components/SKILL.md).

---

# Part 2 — Tables

Standard component contracts and tokens for data tables in dashboards. See [`guide-tables.md`](tables.md) for the full spec — pagination, sorting, filtering, FilterActionBar, SortableTableHeader, PaginationBar, and cell content templates.

---

# Part 3 — Marketing / Landing Pages

## 6–11. Spacing, hero, carousel, CTA, navigation, grid layouts

For the vertical spacing scale, hero section standards, carousel (fixed-height, auto-advance, dots), CTA discipline (approved labels, banned labels, button pairing rules), navigation height, and section grid layouts, see [`skills/design/marketing-layout-patterns`](../skills/design/marketing-layout-patterns/SKILL.md).

---

# Part 4 — Pipeline UX Patterns

Pipeline-specific UX patterns (ConfirmDialog, AIProgressPanel, phase data refetch on focus) are specific to the app-flow.ai product. They have been migrated to:

`bt-ai-web/docs/product/pipeline-patterns.md` in the [bytetalent/bt-ai-web](https://github.com/bytetalent/bt-ai-web) repo.

---

## Related Standards

- [`guide-code.md`](code.md) — naming, component standards
- [`guide-tables.md`](tables.md) — full table spec
- [`guide-forms.md`](forms.md) — validation + form patterns
- [`guide-arch.md`](arch.md) — data flow, hooks pattern
