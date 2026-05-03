---
name: Deploy Conventions
description: Environment topology, branch promotion flow, env-var discipline, migration-on-deploy rule, hosting defaults, observability stack, health check.
category: infra
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase, astro-cloudflare]
version: 1
---

Deployment principles for all Bytetalent-generated projects. Per `bytetalent/docs/guide-deploy.md`. Hosting-specific service selection is in the `infra-service-defaults` skill.

## Environment topology

Three environments, always:

| Environment | Purpose     | DB isolation                              |
| ----------- | ----------- | ----------------------------------------- |
| dev (local) | active dev  | may share a DB with staging               |
| staging     | pre-prod QA | may share a DB with dev                   |
| prod        | live        | **always isolated** — own DB, own secrets |

Never share prod infrastructure with any other environment. Staging may share with dev to reduce cost on early projects.

## Branch promotion flow

```
dev  →  PR  →  staging  →  PR  →  main (prod)
```

- Work on `dev` (or a feature branch off `dev`).
- Never push directly to `staging` or `main` — PR-only.
- Merge strategy: squash into dev; rebase-merge dev → staging; rebase-merge staging → main.

## Environment variable discipline

- **Platform-level secrets** (API keys, DB URLs, Clerk keys): stored in the hosting platform's env settings (Vercel, Cloudflare, EAS).
- **Per-user secrets** (OAuth tokens, provider API keys entered by users): stored in Supabase Vault, accessed only via `withSecrets()`.
- Never commit `.env` files to source control. `.env.example` with placeholder values is fine.
- Required vars are validated at startup via `src/lib/env.ts`; apps fail fast on missing config.

```typescript
// Good — startup validation catches missing config
import { env } from "@/lib/env";
const apiKey = env.OPENAI_API_KEY;

// Bad — silent undefined at runtime
const apiKey = process.env.OPENAI_API_KEY;
```

## Migrations on deploy

- Run migrations as part of the deploy pipeline (pre-deploy script or CI step).
- Never run schema-changing SQL by hand on production.
- RLS policies are migrations too — they ship in the same migration file as the schema change.
- One migration per schema change; never manually edit a migration that has already been applied.

## Hosting defaults by stack

| Stack               | Hosting                |
| ------------------- | ---------------------- |
| Next.js             | Vercel                 |
| Astro               | Cloudflare Pages       |
| ASP.NET             | Azure App Service      |
| React Native (Expo) | EAS Build + EAS Update |

Do not introduce a fourth hosting platform without an explicit architecture decision.

## Observability stack

| Concern             | Tool                                      |
| ------------------- | ----------------------------------------- |
| Error tracking      | Sentry                                    |
| Logs/metrics/traces | Grafana Cloud (Loki + Prometheus + Tempo) |
| Product analytics   | PostHog                                   |

All three must be wired in before a project ships to production.

## Health check

Every app ships `/api/healthz` returning `{ "status": "ok" }` with a lightweight DB query. This endpoint is used by uptime monitors and load balancers.

```typescript
// src/app/api/healthz/route.ts
import { db } from "@/lib/db";
import { sql } from "drizzle-orm";

export async function GET() {
  await db.execute(sql`SELECT 1`);
  return Response.json({ status: "ok" });
}
```
