---
name: API Health Check
description: Every project ships /api/healthz with a lightweight DB probe; returns {status:"ok"} or 503 degraded; used by uptime monitors and post-deploy smoke checks.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Health-check endpoint pattern for Bytetalent projects. Source: `bytetalent/docs/guide-api.md` §10.

## When to apply

Every deployed project. Wire it up when scaffolding the project — uptime monitors, load balancer checks, and post-deploy smoke checks all depend on it.

## Pattern

```ts
// src/app/api/healthz/route.ts
import { db } from "@/lib/db";
import { sql } from "drizzle-orm";

export async function GET() {
  try {
    await db.execute(sql`SELECT 1`);
    return Response.json({ status: "ok" });
  } catch {
    return Response.json({ status: "degraded" }, { status: 503 });
  }
}
```

## Rules

- **Path is `/api/healthz`** (or stack equivalent). Don't vary the name — uptime monitors expect a predictable URL.
- **Probe must be lightweight** — `SELECT 1` is enough to verify DB connectivity without touching app tables.
- **Return `503` on any DB error**, not `500`. Load balancers treat `503` as "remove from pool"; `500` is ambiguous.
- **Don't expose data** — response is `{ status: "ok" | "degraded" }` only. No user counts, version hashes, or internal details.
- **No auth required** — the endpoint is public. No `resolveAccount()` guard.

## Anti-patterns

```ts
// Bad — complex query; slow and can fail for non-infra reasons
await db.query.accounts.findMany(); // touches app data

// Bad — returns internal details
return Response.json({ status: "ok", dbVersion: "15.2", env: process.env.NODE_ENV });

// Bad — throws instead of returning 503
// catch block re-throws; load balancer gets a 500 or a hang
```

## See also

- [`skills/api/api-route-conventions`](../api-route-conventions/SKILL.md) — route layout; healthz is listed alongside other `/api/` routes
- [`skills/infra/deploy-conventions`](../../infra/deploy-conventions/SKILL.md) — post-deploy smoke check that hits `/api/healthz`
- `bytetalent/docs/guide-api.md` §10 — source section
