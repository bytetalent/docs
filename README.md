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

- [`guide-pipeline.md`](guide-pipeline.md) — end-to-end picture: how `bt-ai-web`, `bytetalent/docs`, the bases, the templates, and generated client repos fit together; setup wizard → AI phases → provisioning → code-gen → deploy
- [`guide-base-development.md`](guide-base-development.md) — how a Bytetalent base repo is structured, modified, versioned; how to add a base for a new stack
- [`guide-template-development.md`](guide-template-development.md) — how a template is structured + paired with design; the matrix concept; pairing rule + lint; versioning + pinning

**Tooling:**

- [`guide-agents.md`](guide-agents.md) — autonomous Claude Code (and other agentic CLI) settings: the committed `.claude/settings.json` baseline, what to allow + deny + why, the silent-stall failure mode it prevents
- [`guide-pencil.md`](guide-pencil.md) — Pencil MCP capabilities for `.pen` design files
- [`guide-connections.md`](guide-connections.md) — what each provider connection does, how to revoke, what breaks if revoked (Anthropic, GitHub App, Vercel, Cloudflare, Supabase, Clerk, Stripe, Resend, Upstash, Sentry)

## Skills

Auto-loaded SKILL.md files for agents working on Bytetalent projects. Authoring guide at [`guide-agents.md`](guide-agents.md). For skill-system mechanics (manifest, dual-source, scaffold copy), see app-flow-ai-web's `docs/guide-skills.md`.

**Philosophy + planning:**

- [`skills/module-planning`](skills/module-planning/SKILL.md) — how to decompose a feature into modules before coding
- [`skills/philosophy-composition-over-invention`](skills/philosophy-composition-over-invention/SKILL.md) — prefer composing existing primitives over building new abstractions
- [`skills/philosophy-feature-construction`](skills/philosophy-feature-construction/SKILL.md) — step-by-step feature construction discipline
- [`skills/philosophy-when-no-pattern-fits`](skills/philosophy-when-no-pattern-fits/SKILL.md) — how to handle cases outside existing patterns

**TypeScript + code standards:**

- [`skills/typescript-strict`](skills/typescript-strict/SKILL.md) — strict TypeScript settings + enforcement rules
- [`skills/code-review-checklist`](skills/code-review-checklist/SKILL.md) — PR checklist used during code review

**API patterns:**

- [`skills/api-route-conventions`](skills/api-route-conventions/SKILL.md) — REST route shape, naming, auth guard placement
- [`skills/api-etag-optimistic-concurrency`](skills/api-etag-optimistic-concurrency/SKILL.md) — ETag/If-Match optimistic concurrency contract
- [`skills/api-input-validation`](skills/api-input-validation/SKILL.md) — Zod-based request validation discipline
- [`skills/api-pagination-and-filtering`](skills/api-pagination-and-filtering/SKILL.md) — server-side pagination + filter query patterns
- [`skills/api-sse-streaming`](skills/api-sse-streaming/SKILL.md) — Server-Sent Events streaming from Next.js route handlers
- [`skills/api-webhook-security`](skills/api-webhook-security/SKILL.md) — webhook signature verification + replay protection
- [`skills/route-and-type-patterns`](skills/route-and-type-patterns/SKILL.md) — shared route + inferred type conventions

**Data access + DB:**

- [`skills/data-access-patterns`](skills/data-access-patterns/SKILL.md) — standardized DB query layer structure
- [`skills/db-query-patterns`](skills/db-query-patterns/SKILL.md) — common Drizzle query idioms
- [`skills/drizzle-migration-patterns`](skills/drizzle-migration-patterns/SKILL.md) — migration workflow + common schema patterns
- [`skills/rls-policies`](skills/rls-policies/SKILL.md) — Row Level Security policy templates
- [`skills/supabase-schema`](skills/supabase-schema/SKILL.md) — Supabase schema conventions (audit fields, soft-delete, JSONB)

**Auth + security:**

- [`skills/clerk-auth`](skills/clerk-auth/SKILL.md) — Clerk middleware, `resolveAccount()`, auth guard patterns
- [`skills/scope-guard`](skills/scope-guard/SKILL.md) — permission scope-guard pattern for role-gated actions
- [`skills/security-review`](skills/security-review/SKILL.md) — security checklist applied during review
- [`skills/server-actions-auth`](skills/server-actions-auth/SKILL.md) — Server Actions auth + input validation discipline

**Connections + infrastructure:**

- [`skills/connections-framework`](skills/connections-framework/SKILL.md) — `withSecrets()` boundary, Vault reads, credential shape
- [`skills/proxy-egress`](skills/proxy-egress/SKILL.md) — all provider traffic through proxy egress layer
- [`skills/standardized-access-paths`](skills/standardized-access-paths/SKILL.md) — single canonical path per concern; no parallel access
- [`skills/infra-service-defaults`](skills/infra-service-defaults/SKILL.md) — Vercel/Supabase/Cloudflare defaults for new projects
- [`skills/cloudflare-worker`](skills/cloudflare-worker/SKILL.md) — Cloudflare Worker patterns (KV, env bindings, fetch)

**Error handling + observability:**

- [`skills/error-handling-nextjs`](skills/error-handling-nextjs/SKILL.md) — `apiError()` shape, error boundaries, Sentry capture

**Next.js + App Router:**

- [`skills/nextjs-app-router`](skills/nextjs-app-router/SKILL.md) — App Router conventions, layouts, loading states, server vs client components
- [`skills/nextjs-component-conventions`](skills/nextjs-component-conventions/SKILL.md) — `comp-` prefix, co-location rules, barrel exports

**Design + UI:**

- [`skills/tailwind-and-styling`](skills/tailwind-and-styling/SKILL.md) — Tailwind utility discipline, no inline styles, token mapping
- [`skills/design-tokens-and-theming`](skills/design-tokens-and-theming/SKILL.md) — CSS variable tokens, dark mode, theming sources of truth
- [`skills/form-patterns`](skills/form-patterns/SKILL.md) — form validation, no-submit-flash, React Hook Form conventions
- [`skills/table-patterns`](skills/table-patterns/SKILL.md) — server-side table patterns, FilterActionBar, SortableTableHeader
- [`skills/marketing-layout-patterns`](skills/marketing-layout-patterns/SKILL.md) — hero sizing, CTA discipline, marketing spacing scale
- [`skills/stat-card-and-interactive-components`](skills/stat-card-and-interactive-components/SKILL.md) — stat-card anatomy, interactive component conventions
- [`skills/accessibility-aa`](skills/accessibility-aa/SKILL.md) — WCAG AA accessibility checklist + component requirements

**Astro:**

- [`skills/astro-static`](skills/astro-static/SKILL.md) — Astro Islands conventions for static/marketing sites

## License

Proprietary — Bytetalent internal use.
