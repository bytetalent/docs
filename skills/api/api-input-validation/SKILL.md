---
name: API Input Validation and Error Codes
description: Validate all inputs with zod; centralize schemas in api-schemas.ts; use the canonical error code set; console.warn before 4xx; never leak DB column names in 5xx.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Input validation and error response rules for Route Handlers. Source: `bytetalent/docs/guide-api.md`.

## Input validation

Validate **all** inputs with zod (or stack equivalent) before touching data. This includes route params, query strings, and request body. Use `.safeParse()` so you control the error response.

```ts
// src/lib/api-schemas.ts — centralize for reuse across routes, docs, and tests
import { z } from "zod";

export const createFlowSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().optional(),
  type: z.enum(["automation", "pipeline"]),
});

export const updateFlowSchema = createFlowSchema.partial();

export const flowIdSchema = z.object({ id: z.string().uuid() });
```

```ts
// Route Handler
import { createFlowSchema } from "@/lib/api-schemas";

export async function POST(req: Request) {
  const body = await req.json();
  const parsed = createFlowSchema.safeParse(body);
  if (!parsed.success) {
    console.warn("[POST /api/flows] validation error", parsed.error.flatten());
    return apiError("VALIDATION_ERROR", parsed.error.message);
  }
  // proceed with parsed.data
}
```

## Schema centralization

Define schemas in `src/lib/api-schemas.ts`. This file is the single source of truth for:

- Route Handler validation
- OpenAPI / documentation generation
- Test fixture construction

Never inline `z.object({ ... })` directly in a route file. Extract to `api-schemas.ts` before reusing even once.

## Canonical error code set

Use exactly these codes in `apiError(code, message)` — no synonyms, no free-form strings:

| Code                   | HTTP | When to use                                          |
| ---------------------- | ---- | ---------------------------------------------------- |
| `UNAUTHORIZED`         | 401  | No valid session / missing token                     |
| `FORBIDDEN`            | 403  | Authenticated but not authorized for this resource   |
| `NOT_FOUND`            | 404  | Resource does not exist or caller cannot see it      |
| `VALIDATION_ERROR`     | 400  | Input failed zod schema validation                   |
| `MISSING_PRECONDITION` | 428  | Required header absent (e.g. `If-Match`)             |
| `VERSION_CONFLICT`     | 412  | `If-Match` version mismatch — resource was modified  |
| `CONFLICT`             | 409  | Duplicate / state conflict not related to versioning |
| `RATE_LIMITED`         | 429  | Caller exceeded rate limit                           |
| `AUTH_ERROR`           | 401  | Auth provider error (distinct from missing session)  |
| `DB_ERROR`             | 500  | Database operation failed                            |
| `INTERNAL_ERROR`       | 500  | Unclassified server error                            |

## 4xx logging

Always `console.warn` before returning a 4xx response. This surfaces client-caused errors in server logs without making them user-visible noise.

```ts
// Good
if (!parsed.success) {
  console.warn("[PATCH /api/flows/:id] validation:", parsed.error.message);
  return apiError("VALIDATION_ERROR", parsed.error.message);
}

// Bad — silent 4xx
if (!parsed.success) return apiError("VALIDATION_ERROR", parsed.error.message);
```

## 5xx safety

Never include DB column names, SQL fragments, table names, or internal stack traces in error messages returned to clients.

```ts
// Good
try {
  await db.insert(flows).values(data);
} catch (err) {
  console.error("[POST /api/flows] db insert failed:", err);
  return apiError("DB_ERROR", "Failed to create flow");
}

// Bad — leaks internals
return apiError("DB_ERROR", String(err)); // might include "column flows.account_id..."
```
