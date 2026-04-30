# Deployment Standards

Cross-stack deployment principles. Actual commands, env-var lists, and CI configs are stack-specific and live in each base or product's `docs/` overlay.

---

## 1. Environments

Every Bytetalent project ships with at least two environments. Three is typical:

| Environment | Purpose | Branch | DB |
|---|---|---|---|
| **dev** (local) | Daily development | `dev` (default working branch) | Shared dev DB or local |
| **staging** | Pre-prod verification, QA | `staging` (PR-only) | Shared with dev (or isolated, depending on product) |
| **prod** | Production | `main` (PR-only) | Isolated production DB |

### Branch flow

```
work commits → dev → PR → staging → PR → main
```

Rules:
- **All work commits go on `dev`.** Don't create short-lived feature branches (`feat/*`, `fix/*`) for normal work — they're how changes get lost when PRs land on the wrong base.
- **Never push directly to `staging` or `main`.** Both are PR-only.
- **Promotion path is one direction** — `dev → staging → main`. No back-merging from main to dev.
- **CI runs on PRs into `staging` and `main`** (not on every dev push, by design — dev is the working branch).

For pure-infrastructure repos (the bases), `main` + feature-PR-per-change is acceptable since churn is low and stability is the goal.

---

## 2. Secrets & Env Vars

- **Platform-level secrets** (auth provider keys, GitHub App private key, payment processor keys, error-tracking DSN) live in the host's environment-variable system (Vercel Environment Variables, Cloudflare Pages env vars, Azure App Service config).
- **Per-user / per-account secrets** (user OAuth tokens, BYOK API keys) live in **Supabase Vault** — encrypted at rest, accessed only via the proxy and adapter layer.
- **Required env vars are validated at startup** in `src/lib/env.ts` (or stack equivalent) — apps fail fast on missing config, not silently misbehave.
- **Never commit `.env` files.** `.env.example` is committed; `.env.local` and `.env.test.local` are gitignored.

See [`guide-arch.md`](guide-arch.md) §5 (Vault secret storage) for the full pattern.

---

## 3. Database Migrations

- Migrations live in a tracked folder (e.g., `drizzle/` for Drizzle-based projects).
- **Migrations are committed source files**, not generated and discarded. Each change to schema = a new migration file.
- **Apply migrations as part of deploy** — never run schema-changing SQL by hand on production.
- **RLS policies are migrations too** — they're part of the schema, not application code.
- Test migrations in dev → staging → prod, in that order. Don't skip staging.

---

## 4. Domain & Hosting

| Stack | Hosting | Notes |
|---|---|---|
| Astro marketing site | Cloudflare Pages | Best CDN + Workers integration; Astro deploys cleanly |
| Next.js webapp | Vercel | Native App Router, Edge Functions, etc. |
| ASP.NET Core API | Azure App Service | First-class .NET hosting |
| Expo mobile | EAS Build + EAS Update | OTA updates over the air |

Each hosting choice is set per-stack in the base; products inherit unless they explicitly choose otherwise.

---

## 5. CI Pipeline (Required Steps)

Every project's CI runs at minimum:

1. **Lint** — ESLint for TS/JS, equivalent for other stacks
2. **Typecheck** — `tsc --noEmit` for TS projects
3. **Format check** — Prettier `--check`
4. **Unit tests** — Vitest, xUnit, or stack equivalent
5. **Build** — production build succeeds

E2E tests run on a separate cadence (scheduled or pre-release) — see [`guide-testing.md`](guide-testing.md).

---

## 6. Operational Defaults

- **Sentry** for error tracking (or stack-equivalent for non-JS)
- **Grafana Cloud** for logs + metrics + traces
- **PostHog** for product analytics
- **Health check endpoint** at `/api/healthz` (or stack equivalent) returning `{ status: "ok" }` with a low-latency DB query — used by uptime monitors

---

## 7. Stack-Specific Deploy Commands

This file lists principles, not commands. Each base or product's `docs/guide-deploy.md` overlay spells out:

- Exact deploy command (`vercel --prod`, `wrangler pages deploy`, etc.)
- Environment-variable checklist for that stack
- Required service accounts and how to provision them
- Rollback procedure
- Migration apply timing (pre-deploy vs. post-deploy)
- Custom domain DNS setup
