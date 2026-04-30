# bytetalent/docs

Universal cross-stack standards for every Bytetalent project. The patterns, conventions, and decisions that apply regardless of framework — Next.js webapps, Astro marketing sites, ASP.NET Core APIs, Expo mobile apps.

## What lives here vs. in a base or product repo

This repo holds the **universal layer** — things true regardless of stack:

- **Coding standards** — TypeScript discipline, naming, file size, accessibility, security mindset
- **Architecture decisions** — single-canonical-path rule, ETag/If-Match concurrency, RLS-first security, audit fields, soft-delete
- **Symbols and contracts** — `resolveAccount()`, `apiError()`, `useEntityList`, the `comp-` component prefix, the `withSecrets()` boundary
- **DB patterns** — version columns, audit + soft-delete, RLS templates, JSONB shape conventions
- **API patterns** — REST route handlers, `If-Match` rules, error response shape, validation discipline
- **Table & form patterns** — pagination/sorting/filtering, validation rules, input states
- **Design discipline** — token-driven color, theming sources of truth, hero sizing, CTA labels, stat-card anatomy
- **Review & testing** — PR checklist, code review categories, Playwright conventions

What does **not** live here:

- **Stack-specific implementation** — App Router vs Astro Islands vs Razor; Drizzle migrations vs EF Core; Clerk middleware matchers; deploy commands. Those go in each base/product's own `docs/` overlay.
- **Product-specific** — entity schemas (`clients`, `projects`, etc.), feature plans, sample data, product vision. Those stay in the consuming repo.

## How a base or product repo uses this

Every base and product repo carries a thin `docs/` overlay that:

1. **References this repo** for cross-stack standards (linked at the top of the overlay's index).
2. **Adds stack-specifics** — file conventions for that framework, deployment commands, build setup, stack-specific component patterns.
3. **Adds product-specifics** (if a product, not a base) — entity schemas, feature plans, etc.

Example structure for a base repo's `docs/`:

```text
{base}/docs/
  README.md            # points at bytetalent/docs for universal standards;
                       # lists this base's stack-specific deviations
  guide-stack.md       # framework-specific patterns (App Router, Astro Islands, etc.)
  guide-deploy.md      # actual deploy commands for this stack (Vercel, CF Pages, …)
  lessons.md           # findings captured during the base's pattern-proving phase
```

## Index

**Standards (cross-stack patterns):**

- [`guide-code.md`](guide-code.md) — coding standards, naming, file size, types, security, performance, DoD
- [`guide-arch.md`](guide-arch.md) — standardized access paths, ETag/If-Match, proxy egress, vault, RLS, connections framework
- [`guide-db.md`](guide-db.md) — schema conventions, audit fields, soft-delete, RLS templates, JSONB shape rules
- [`guide-api.md`](guide-api.md) — REST handlers, ETag/If-Match contract, error response shape, validation, auth pattern
- [`guide-design.md`](guide-design.md) — design tokens, typography, theming, stat cards, marketing spacing scale, CTA discipline
- [`guide-tables.md`](guide-tables.md) — server-side pagination/sorting/filtering, FilterActionBar, SortableTableHeader, PaginationBar, cell templates
- [`guide-forms.md`](guide-forms.md) — validation rules pattern, no-submit-flash discipline, custom hook ownership, accessibility
- [`guide-testing.md`](guide-testing.md) — unit/integration/E2E split, Playwright project structure, Clerk test mode, CI gating
- [`guide-review.md`](guide-review.md) — review categories, severity rubric, output format, version currency audit
- [`guide-deploy.md`](guide-deploy.md) — deployment principles (commands live in stack overlays)

**Building the building blocks:**

- [`guide-base-development.md`](guide-base-development.md) — how a Bytetalent base repo is structured, modified, versioned; how to add a base for a new stack
- [`guide-template-development.md`](guide-template-development.md) — how a template is structured + paired with design; the matrix concept; pairing rule + lint; versioning + pinning

**Tooling:**

- [`guide-pencil.md`](guide-pencil.md) — Pencil MCP capabilities for `.pen` design files

## License

Proprietary — Bytetalent internal use.
