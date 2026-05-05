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

**Agentic coding:**

- [`guides/agentic-coding.md`](guides/agentic-coding.md) — layer cake, lesson-capture flow, what every generated repo carries, upstream feedback model, memory seeds, self-review patterns
- [`guide-agents.md`](guide-agents.md) — autonomous Claude Code (and other agentic CLI) settings: the committed `.claude/settings.json` baseline, what to allow + deny + why, the silent-stall failure mode it prevents
- [`guide-pencil.md`](guide-pencil.md) — Pencil MCP capabilities for `.pen` design files
- [`guide-connections.md`](guide-connections.md) — what each provider connection does, how to revoke, what breaks if revoked (Anthropic, GitHub App, Vercel, Cloudflare, Supabase, Clerk, Stripe, Resend, Upstash, Sentry)

## Skills

Auto-loaded SKILL.md files for agents working on Bytetalent projects. Authoring guide at [`guide-agents.md`](guide-agents.md). For skill-system mechanics (manifest, dual-source, scaffold copy), see app-flow-ai-web's `docs/guide-skills.md`.

**Philosophy + planning:**

- [`skills/module-planning`](skills/module-planning/SKILL.md) — per-module planning: check patterns, compose if multiple apply, fall back to philosophy
- [`skills/philosophy-composition-over-invention`](skills/philosophy-composition-over-invention/SKILL.md) — compose existing skills + small glue; prove need with third use case, not the first
- [`skills/philosophy-feature-construction`](skills/philosophy-feature-construction/SKILL.md) — data-first, smallest-slice-first, deferred polish for novel features
- [`skills/philosophy-when-no-pattern-fits`](skills/philosophy-when-no-pattern-fits/SKILL.md) — heuristics for novel features that don't match any named pattern

**Code conventions:**

- [`skills/typescript-strict`](skills/typescript-strict/SKILL.md) — no any, no implicit returns, exhaustive switch, Drizzle-inferred types only
- [`skills/nextjs-component-conventions`](skills/nextjs-component-conventions/SKILL.md) — `comp-` naming, kebab-case files, server-first rendering, component size limits
- [`skills/code-review-checklist`](skills/code-review-checklist/SKILL.md) — structured review categories, severity rubric, output format, and PR cadence

**Stack-specific:**

- [`skills/nextjs-app-router`](skills/nextjs-app-router/SKILL.md) — folder structure, server/client split, route handlers — Next.js 16 App Router
- [`skills/error-handling-nextjs`](skills/error-handling-nextjs/SKILL.md) — `apiError()` shape, error boundaries, Sentry capture in Next.js
- [`skills/astro-static`](skills/astro-static/SKILL.md) — content collections, MDX, no JS — brochure-friendly Astro conventions
- [`skills/cloudflare-worker`](skills/cloudflare-worker/SKILL.md) — form handlers, secrets via wrangler env, edge-ready code patterns
- [`skills/route-and-type-patterns`](skills/route-and-type-patterns/SKILL.md) — `routes.ts` centralization, type config registry, `useTheme` singleton, env validation

**Architecture + access:**

- [`skills/standardized-access-paths`](skills/standardized-access-paths/SKILL.md) — one canonical path per concern; Drizzle for DB, `resolveAccount` for auth, no parallel implementations
- [`skills/connections-framework`](skills/connections-framework/SKILL.md) — two-level credential model, three provider variants, Vault-only secret storage, `withSecrets()` boundary
- [`skills/connections/github-app-vs-oauth`](skills/connections/github-app-vs-oauth/SKILL.md) — GitHub App (not OAuth App) for repo access: bot identity, per-repo scoping, short-lived tokens, Marketplace eligibility
- [`skills/proxy-egress`](skills/proxy-egress/SKILL.md) — all LLM + provider calls route through POST /api/proxy; direct provider fetch never allowed outside adapters
- [`skills/architecture/realtime-supabase`](skills/architecture/realtime-supabase/SKILL.md) — Supabase Realtime Postgres change listeners for live updates; no polling, no Pusher/Ably
- [`skills/infra-service-defaults`](skills/infra-service-defaults/SKILL.md) — use the already-in-stack service (Vercel+Supabase+Upstash) before adding a fourth
- [`skills/clerk-auth`](skills/clerk-auth/SKILL.md) — server `auth()`, middleware, JWT template for Supabase RLS, social login
- [`skills/scope-guard`](skills/scope-guard/SKILL.md) — hard-cap on must-have feature count; surfaces silent scope creep aggressively
- [`skills/server-actions-auth`](skills/server-actions-auth/SKILL.md) — Zod validation before data touch, auth guard sequence, authz inside the action

**API patterns:**

- [`skills/api-route-conventions`](skills/api-route-conventions/SKILL.md) — REST-only routes, list vs. detail file layout, auth guard order, CORS rules, health-check endpoint
- [`skills/api/api-cors-policy`](skills/api/api-cors-policy/SKILL.md) — specific origin (never `*`) for credentialed requests; restrict allowed methods; set `Access-Control-Max-Age`
- [`skills/api/api-healthcheck`](skills/api/api-healthcheck/SKILL.md) — `/api/healthz` with `SELECT 1` DB probe; returns `{status:"ok"}` or 503; no auth, no data exposure
- [`skills/api-input-validation`](skills/api-input-validation/SKILL.md) — Zod-based validation, centralize schemas in `api-schemas.ts`, canonical error codes
- [`skills/api-pagination-and-filtering`](skills/api-pagination-and-filtering/SKILL.md) — cursor-based pagination, default limit 25/max 100, flat sort/filter query params
- [`skills/api-sse-streaming`](skills/api-sse-streaming/SKILL.md) — `Response` + `TransformStream` for SSE; normalized event shape across providers
- [`skills/api-webhook-security`](skills/api-webhook-security/SKILL.md) — validate provider HMAC signature first; require idempotency via delivery ID; respond within 5 s
- [`skills/api-etag-optimistic-concurrency`](skills/api-etag-optimistic-concurrency/SKILL.md) — every GET returns ETag; PUT/PATCH/DELETE require If-Match; zero rows = conflict

**Data + DB:**

- [`skills/data-access-patterns`](skills/data-access-patterns/SKILL.md) — DB singleton, repository boundary, detail-page fetch with `Promise.all`/`.catch`, `useEntityList`
- [`skills/db-query-patterns`](skills/db-query-patterns/SKILL.md) — every read includes `accountId` + `deletedAt` filters; every write includes version match + `updatedAt` + `updatedBy`
- [`skills/drizzle-migration-patterns`](skills/drizzle-migration-patterns/SKILL.md) — one file per schema change, never edit applied migrations, schema + RLS in same migration
- [`skills/rls-policies`](skills/rls-policies/SKILL.md) — Supabase RLS templates for user-owned + shared resources; Drizzle defense-in-depth; service-role bypass rules
- [`skills/supabase-schema`](skills/supabase-schema/SKILL.md) — Drizzle conventions, audit fields, version column, soft-delete, partial unique indexes, JSONB typing

**Design + UX:**

- [`skills/design-tokens-and-theming`](skills/design-tokens-and-theming/SKILL.md) — CSS custom properties for all design values; never hardcode colors or magic-number spacing
- [`skills/marketing-layout-patterns`](skills/marketing-layout-patterns/SKILL.md) — spacing scale, hero heights, carousel dimensions, CTA labels, nav conventions for marketing pages
- [`skills/stat-card-and-interactive-components`](skills/stat-card-and-interactive-components/SKILL.md) — stat-card anatomy, `ConfirmDialog`, `AIProgressPanel`, AI auto-trigger patterns
- [`skills/accessibility-aa`](skills/accessibility-aa/SKILL.md) — enforces WCAG 2.1 AA on design and code outputs
- [`skills/tailwind-and-styling`](skills/tailwind-and-styling/SKILL.md) — `cn()` for class composition, `cva` for variants, CSS variables only in dynamic styles
- [`skills/form-patterns`](skills/form-patterns/SKILL.md) — validation display, submit behavior, field anatomy, accessibility, state management; no form library
- [`skills/table-patterns`](skills/table-patterns/SKILL.md) — server-side pagination/sorting/filtering, `FilterActionBar`, `SortableTableHeader`, cell templates

**Testing + security:**

- [`skills/testing-conventions`](skills/testing-conventions/SKILL.md) — unit/integration/E2E pyramid, Playwright project structure, Clerk test OTP, CI cadence, flakiness policy
- [`skills/security-review`](skills/security-review/SKILL.md) — cross-cutting security checks for any phase output

**Deployment + base/template development:**

- [`skills/deploy-conventions`](skills/deploy-conventions/SKILL.md) — environment topology, branch promotion, env-var discipline, migration-on-deploy, health check
- [`skills/base-development`](skills/base-development/SKILL.md) — how to structure, version, and evolve a Bytetalent base app — manifest keys, semver rules, CI smoke test
- [`skills/template-development`](skills/template-development/SKILL.md) — template anatomy, `feature.json` fields, bundle file actions, token bindings, pairing rule
- [`skills/pencil-design`](skills/pencil-design/SKILL.md) — working with `.pen` design files via Pencil MCP — encrypted access, batch limits, session hygiene

**Orchestration + process:**

- [`skills/github-api-efficiency`](skills/github-api-efficiency/SKILL.md) — bundle, cache, batch — survive the 5000 pts/hour GraphQL limit during board manipulation, issue grooming, orchestrator dispatch
- [`skills/lessons-learned-discipline`](skills/lessons-learned-discipline/SKILL.md) — after a debug session finds root cause, run this decision flow to determine where the fix must land (tooling → guide → skill → CLAUDE.md → memory)
- [`skills/board-numbering-conventions`](skills/board-numbering-conventions/SKILL.md) — Release field for stack-ranked buckets, WBS dot-decimal title prefix for within-release ordering, `state:*` labels for backlog/shovel-ready meta-status; deprecates `epic` + `lane:claude` labels

**Pattern skills (`paths/`)** — recurring UX/data patterns code-gen agents apply per-feature:

- [`skills/paths/server-client-page-split`](skills/paths/server-client-page-split/SKILL.md) — RSC + `"use client"` page-pair convention; thin server wrapper delegates to interactive client component
- [`skills/paths/detail-page-with-tabs`](skills/paths/detail-page-with-tabs/SKILL.md) — multi-view entity detail page with a tab bar; each tab renders a distinct view of the same entity
- [`skills/paths/settings-sections-page`](skills/paths/settings-sections-page/SKILL.md) — settings page as a vertical stack of named section cards; each section is isolated with its own state
- [`skills/paths/paginated-data-table`](skills/paths/paginated-data-table/SKILL.md) — list-of-entities page with stat cards, toolbar, `DataTable` rows, and empty state
- [`skills/paths/entity-list-hook`](skills/paths/entity-list-hook/SKILL.md) — three-layer generic→domain→page data-access hook for list surfaces
- [`skills/paths/wizard`](skills/paths/wizard/SKILL.md) — multi-step form with URL-driven step state, progress indicator, back/next nav, terminal action
- [`skills/paths/collapsible-section-stack`](skills/paths/collapsible-section-stack/SKILL.md) — dense entity-editing pages as a vertical stack of named `CollapsiblePanel` sections
- [`skills/paths/modal-dialog-action`](skills/paths/modal-dialog-action/SKILL.md) — overlay-based action pattern: `ConfirmDialog`, slide-over panel, read-only modal
- [`skills/paths/async-toast-mutation`](skills/paths/async-toast-mutation/SKILL.md) — set-busy → fetch → show toast → clear-busy frame with `appToast` as the single toast path
- [`skills/paths/fetch-error-retry`](skills/paths/fetch-error-retry/SKILL.md) — persistent `FetchErrorBanner` + retry button when the initial load fails
- [`skills/paths/resolve-account-route-handler`](skills/paths/resolve-account-route-handler/SKILL.md) — every protected Route Handler opens with `resolveAccount()` + 401 guard
- [`skills/paths/api-error-handler`](skills/paths/api-error-handler/SKILL.md) — every route handler wraps business logic in `try/catch` and terminates with `apiError()`
- [`skills/paths/optimistic-crud-with-etag`](skills/paths/optimistic-crud-with-etag/SKILL.md) — integer version + `If-Match` header; API returns `ETag`, client sends `If-Match` on PUT, handles 412
- [`skills/paths/soft-delete-api-resource`](skills/paths/soft-delete-api-resource/SKILL.md) — DELETE sets `deletedAt`/`isArchived`, never hard-deletes; `notDeleted()` helper filters all SELECTs

## License

Proprietary — Bytetalent internal use.
