---
name: API Error Handler
description: Every route handler wraps business logic in try/catch and terminates with apiError() — centralized error classification, Sentry capture, safe 5xx stripping.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Every route handler has the same `try { ... } catch (err) { return apiError(..., err) }` frame. Inline 4xx responses handle known failures; `apiError()` handles unexpected exceptions.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #13.

Related skill: `skills/error-handling-nextjs` — `apiError()` implementation + error boundaries.

## Core rules

- All business logic in a route handler lives inside a `try` block.
- The `catch` block has exactly one line: `return apiError("<METHOD> <path>", err)`.
- Known 4xx conditions (validation errors, not found, conflict, version mismatch) are returned **before** the try block or **inline** inside it — never through `apiError()`.
- `apiError()` is the only call that touches Sentry capture. Do not call `Sentry.captureException()` directly from route handlers.
- `isUniqueViolation(err)` maps Postgres 23505 errors to `409 Conflict` — use it instead of relying on `apiError()` to catch them.
- Route label passed to `apiError()` must be `"<METHOD> /api/<path>"` exactly — it appears in Sentry as the breadcrumb.

## Full route handler shape

```ts
// src/app/api/entities/route.ts
import { resolveAccount } from "@/lib/session";
import { apiError, isUniqueViolation } from "@/lib/api-error";
import { NextResponse } from "next/server";

export async function POST(req: Request) {
  // 1. Auth guard (before try block)
  const account = await resolveAccount();
  if (!account)
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  // 2. Input validation (before try block or at top of try)
  const body = await req.json();
  const parsed = CreateEntitySchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json(
      { error: "Invalid input", details: parsed.error.flatten() },
      { status: 400 },
    );
  }

  // 3. Business logic inside try/catch
  try {
    const [entity] = await db
      .insert(entities)
      .values({ ...parsed.data, accountId: account.accountId })
      .returning();

    return NextResponse.json(entity, { status: 201 });
  } catch (err) {
    // 4. Unique violation → 409
    if (isUniqueViolation(err)) {
      return NextResponse.json(
        { error: "Entity already exists" },
        { status: 409 },
      );
    }
    // 5. Everything else → apiError
    return apiError("POST /api/entities", err);
  }
}
```

## apiError() behavior

`apiError()` (defined in `src/lib/api-error.ts`):

- Classifies the error: VaultError, DB constraint, auth, generic.
- Strips internal details from 5xx response bodies (safe for clients).
- Captures to Sentry with the route label as context.
- Returns `NextResponse.json({ error: string, code?: string }, { status: 500 })`.

## Inline 4xx patterns

```ts
// Not found
const entity = await db.query.entities.findFirst({ where: /* ... */ });
if (!entity) return NextResponse.json({ error: "Not found" }, { status: 404 });

// Conflict (before DB write)
const existing = await db.query.entities.findFirst({ where: eq(t.slug, slug) });
if (existing) return NextResponse.json({ error: "Slug already taken" }, { status: 409 });

// Version mismatch (ETag)
if (existing.version !== clientVersion) {
  return NextResponse.json({ error: "Version mismatch", code: "PRECONDITION_FAILED" }, { status: 412 });
}
```

## What NOT to do

```ts
// Bad — no try/catch, unhandled exceptions crash the route
export async function POST(req: Request) {
  const account = await resolveAccount();
  const body = await req.json();
  const entity = await db.insert(entities).values(body).returning();
  return NextResponse.json(entity[0]);
}
```

```ts
// Bad — catching but not using apiError
export async function POST(req: Request) {
  try {
    // ...
  } catch (err) {
    console.error(err); // logs but doesn't capture to Sentry, returns nothing
    return NextResponse.json(
      { error: "Something went wrong" },
      { status: 500 },
    );
  }
}
```

```ts
// Bad — routing all errors through apiError including 4xx
try {
  const entity = await db.query.entities.findFirst({ where: /* ... */ });
  if (!entity) throw new Error("Not found"); // Don't throw — return inline
} catch (err) {
  return apiError("GET /api/entities/[id]", err); // apiError wraps this as 500
}
```

## Error response shape

The `{ error: string, code?: string }` shape is the contract for all error responses:

```json
{ "error": "Not found" }
{ "error": "Invalid input", "details": { ... } }
{ "error": "Version mismatch", "code": "PRECONDITION_FAILED" }
{ "error": "Internal server error" }
```

Clients check `status` first, then optionally read `code` for specific handling (e.g., 412 with `PRECONDITION_FAILED` triggers a reload, not just an error toast).

## Related skills

- `skills/paths/resolve-account-route-handler` — the auth guard that precedes this
- `skills/error-handling-nextjs` — apiError() implementation, error boundaries
- `skills/api-input-validation` — Zod validation that precedes the try block
