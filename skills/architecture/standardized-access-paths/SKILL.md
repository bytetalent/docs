---
name: Standardized Access Paths
description: One canonical path per concern — Drizzle for DB+Vault, resolveAccount for auth, apiError for errors, proxy for provider egress. Never add a parallel implementation.
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Every cross-cutting concern has **one** canonical path. Parallel paths for the same concern create drift, security gaps, and hard-to-trace bugs (e.g. the PGRST002 Vault incident, 2026-04-28). Per `bytetalent/docs/guide-arch.md` access-paths section.

## Canonical path table

| Concern                     | Canonical path                                    | Never use                                              |
| --------------------------- | ------------------------------------------------- | ------------------------------------------------------ |
| Database reads/writes       | Drizzle (`src/lib/db`)                            | PostgREST, supabase-js for app tables                  |
| Vault secrets               | Drizzle via `src/lib/connections/secrets.ts` only | PostgREST, direct SQL outside secrets.ts               |
| Auth resolution             | `resolveAccount()` in `src/lib/session.ts`        | Calling `auth()` then manually looking up account      |
| API error responses         | `apiError()` in `src/lib/api-error.ts`            | `return Response.json({ error: String(err) })`         |
| Frontend → backend          | `fetch('/api/...')` from hooks                    | Calling providers or Supabase REST from browser        |
| Provider egress (LLM/tools) | `POST /api/proxy/...`                             | Direct `fetch` to Anthropic/OpenAI from route handlers |
| ID generation               | `crypto.randomUUID()`                             | nanoid, cuid2, uuid packages                           |
| Service-role DB ops         | Drizzle with documented code comment              | Creating a parallel admin supabase-js client           |

## Rules

- If you think you need a new path for an existing concern, **stop and propose it** as an architecture change first — don't quietly add a parallel implementation.
- The only exception to proxy-as-egress is `adapter.test()` during BYOK validation (no project context, validating the credential itself). Everything else goes through the proxy.
- `src/lib/supabase/{server,client}.ts` — deprecated; zero callers; don't use them.
- `src/lib/supabase/admin.ts` — one legitimate caller only: `src/lib/storage/project-files.ts` for Supabase Storage (Drizzle doesn't cover Storage). Don't add other callers.

## Good

```ts
// Auth + DB access
import { resolveAccount } from "@/lib/session";
import { db } from "@/lib/db";

export async function GET(req: Request) {
  const { accountId } = await resolveAccount();
  const project = await db.query.projects.findFirst({
    where: and(eq(projects.id, id), eq(projects.accountId, accountId)),
  });
  if (!project) return apiError("NOT_FOUND", "Project not found");
  return Response.json(project);
}
```

## Bad

```ts
// Never: raw supabase-js for app tables, String(err) in 5xx
const supabase = createServerClient(); // deprecated path
const { data } = await supabase.from("projects").select("*"); // bypasses Drizzle
return Response.json({ error: String(err) }, { status: 500 }); // leaks internals
```
