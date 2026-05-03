---
name: API Route Conventions
description: REST-only routes, file layout for list vs. detail, auth guard order (validate → auth → authz → logic), CORS rules, and health-check endpoint.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Conventions for structuring Next.js Route Handlers. Source: `bytetalent/docs/guide-api.md`.

## Shape: REST only

Use REST (not RPC, tRPC, or GraphQL) for all public API routes. Server Actions are acceptable for in-page mutations only (co-located with the page that calls them).

## Route file layout

| Endpoint type                            | File path                            |
| ---------------------------------------- | ------------------------------------ |
| List (`GET`, `POST`)                     | `src/app/api/<entity>/route.ts`      |
| Detail (`GET`, `PUT`, `PATCH`, `DELETE`) | `src/app/api/<entity>/[id]/route.ts` |

```
src/app/api/
  flows/
    route.ts          ← GET (list), POST (create)
    [id]/
      route.ts        ← GET (detail), PUT, PATCH, DELETE
```

## Auth guard order

Every handler executes in this exact sequence — never reorder:

1. **Schema validate** — parse and validate inputs with zod; return 400 on failure before touching auth
2. **Auth check** — call `resolveAccount()`; return 401 if unauthenticated
3. **Authz check** — verify the caller owns or has permission for the resource; return 403 if not
4. **Business logic** — DB reads, writes, side-effects

```ts
// Good — correct guard order
export async function PATCH(req: Request, { params }: { params: { id: string } }) {
  // 1. Schema validate
  const body = await req.json();
  const parsed = updateFlowSchema.safeParse(body);
  if (!parsed.success) return apiError("VALIDATION_ERROR", parsed.error.message);

  // 2. Auth check
  const { accountId } = await resolveAccount();

  // 3. Authz check (DB read scoped to accountId)
  const flow = await db.query.flows.findFirst({
    where: and(eq(flows.id, params.id), eq(flows.accountId, accountId)),
  });
  if (!flow) return apiError("NOT_FOUND", "Flow not found");

  // 4. Business logic
  const result = await db.update(flows).set(parsed.data).where(eq(flows.id, params.id)).returning();
  return Response.json(result[0]);
}

// Bad — business logic before auth check
export async function PATCH(req: Request, { params }: { params: { id: string } }) {
  const flow = await db.query.flows.findFirst({ where: eq(flows.id, params.id) }); // no auth yet
  const { accountId } = await resolveAccount();
  // ...
}
```

## CORS

Never use `Access-Control-Allow-Origin: *` for credentialed requests. Only allow the methods the endpoint actually serves.

```ts
// Good — restrict to served methods
const corsHeaders = {
  "Access-Control-Allow-Origin": process.env.ALLOWED_ORIGIN!,
  "Access-Control-Allow-Methods": "GET, POST",
  "Access-Control-Allow-Headers": "Content-Type, Authorization",
};

// Bad — wildcard origin with credentials
"Access-Control-Allow-Origin": "*"  // never for credentialed requests
```

## Health check

Every app ships `/api/healthz` returning `{ status: "ok" }` with a lightweight DB query. Used by uptime monitors.

```ts
// src/app/api/healthz/route.ts
import { db } from "@/lib/db";

export async function GET() {
  await db.execute(sql`SELECT 1`);
  return Response.json({ status: "ok" });
}
```
