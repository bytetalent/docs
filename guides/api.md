# API Standards

Cross-stack REST API conventions — error response shape, ETag/If-Match contract, validation discipline, auth pattern. Stack-specific routing details (Next.js Route Handlers, Astro endpoints, ASP.NET controllers) live in each base/product's overlay.

---

## 1. REST Route Handlers

REST is the chosen API approach — not RPC, not tRPC, not GraphQL.

```
src/app/api/                      (Next.js App Router)
  flows/
    route.ts                      → GET /api/flows, POST /api/flows
    [id]/
      route.ts                    → GET /api/flows/:id, PUT, DELETE
  channels/
    route.ts
  …
```

**Why REST over RPC / tRPC / Server Actions for the canonical API shape:**

1. ETags and `If-Match` are designed into the system — they belong in HTTP headers, not function calls.
2. OAuth callbacks and webhook endpoints require Route Handlers regardless — one consistent pattern.
3. Mobile apps and external integrations are realistic — same API works.

Server Actions are still acceptable for in-page mutations where an HTTP boundary adds no value (e.g., a profile-page form). But the canonical entity API surface is REST.

---

## 2. ETag / If-Match Contract

All endpoints returning a single resource must include an `ETag` header. The ETag value is the row's `version` integer, stringified.

### Read

```
GET /api/flows/123
→ 200 OK
  ETag: "42"
  Content-Type: application/json
  
  { "id": "123", "name": "...", "version": 42, … }
```

### Write requirements

`PUT`, `PATCH`, `DELETE` must include `If-Match`:

```
PUT /api/flows/123
If-Match: "42"
Content-Type: application/json

{ "name": "Updated" }
```

| Outcome | HTTP status | Response |
|---|---|---|
| Version matches — update succeeds | `200` | New ETag header `ETag: "43"`, updated row body |
| Version mismatch — concurrent edit | `412 Precondition Failed` | `{ "error": "Resource was modified", "code": "VERSION_CONFLICT" }` |
| `If-Match` header missing | `428 Precondition Required` | `{ "error": "If-Match header required", "code": "MISSING_PRECONDITION" }` |

### Rules

- `If-Match: *` is **not** accepted — a specific version must always be supplied.
- List endpoints (`GET /api/flows`) do **not** return ETags per item. Use the individual resource endpoint to get the current version before writing.
- `DELETE` follows the same rule — soft-delete is still a write requiring `If-Match`.

### Server-side implementation

```typescript
export async function PUT(req: Request, { params }: { params: { id: string } }) {
  const ifMatch = req.headers.get("if-match");
  if (!ifMatch) {
    return Response.json(
      { error: "If-Match header required", code: "MISSING_PRECONDITION" },
      { status: 428 }
    );
  }
  const expectedVersion = parseInt(ifMatch.replace(/"/g, ""), 10);

  const result = await db.update(flows)
    .set({ ...input, version: sql`${flows.version} + 1`, updatedAt: new Date() })
    .where(and(eq(flows.id, params.id), eq(flows.version, expectedVersion)))
    .returning();

  if (result.length === 0) {
    return Response.json(
      { error: "Resource was modified by another request", code: "VERSION_CONFLICT" },
      { status: 412 }
    );
  }

  return Response.json(result[0], {
    headers: { ETag: `"${result[0].version}"` },
  });
}
```

---

## 3. Auth Guard Pattern

Every server-side handler follows:

```typescript
export async function GET(req: Request) {
  try {
    // 1. Schema validation on input
    const params = QuerySchema.parse(Object.fromEntries(new URL(req.url).searchParams));

    // 2. Auth check — reject unauthenticated requests
    const account = await resolveAccount();
    if (!account) {
      return Response.json({ error: "Unauthorized", code: "UNAUTHORIZED" }, { status: 401 });
    }

    // 3. Authz check — verify access to the resource (handled inline or via repository)

    // 4. Business logic
    const data = await db.query.flows.findMany({
      where: and(eq(flows.accountId, account.id), isNull(flows.deletedAt)),
    });

    return Response.json({ items: data });
  } catch (err) {
    return apiError("GET /api/flows", err);
  }
}
```

Rules:
- **Always validate input** with zod (or stack equivalent) before touching data.
- **Auth check first**, before business logic.
- **Wrap every handler in try/catch** with `apiError()` in the catch.
- **Don't put authz only in the UI** — UI checks are convenience; server handlers are the security boundary.

---

## 4. Error Response Shape

All errors return:

```json
{
  "error": "human-readable message",
  "code": "ERROR_CODE",
  "details": { /* optional, for 4xx validation errors only */ }
}
```

Error codes (the set is small on purpose):

| Code | When | Status |
|---|---|---|
| `UNAUTHORIZED` | No session / not logged in | 401 |
| `FORBIDDEN` | Logged in but no access to the resource | 403 |
| `NOT_FOUND` | Resource doesn't exist | 404 |
| `VALIDATION_ERROR` | Input failed schema check | 400 |
| `MISSING_PRECONDITION` | Required header (e.g., `If-Match`) absent | 428 |
| `VERSION_CONFLICT` | Optimistic lock violation | 412 |
| `CONFLICT` | Slug collision, unique violation, etc. | 409 |
| `RATE_LIMITED` | Too many requests | 429 |
| `AUTH_ERROR` | Auth resolution / external auth provider failure | 500 (logged), 503 sometimes |
| `DB_ERROR` | Database operation failure | 500 (logged) |
| `INTERNAL_ERROR` | Catch-all | 500 (logged) |

`apiError()` (in `src/lib/api-error.ts`) classifies the caught error into one of these codes based on its shape — instanceof checks, error-name matching, and finally substring matching as last resort.

### What `apiError()` does

```typescript
import { apiError } from "@/lib/api-error";

// In any catch block:
return apiError("GET /api/route", err);

// Returns a structured JSON response with safe message + code,
// logs the full error server-side (for Vercel/Sentry observability).
```

Behavior:
- **Structured response** — never leaks raw `String(err)` or stack traces.
- **Safe 5xx messages** — internal details stay server-side.
- **Full server logging** — complete error logged via `console.error` for observability (or Sentry).
- **Phase tracking** — `resolveAccount()` errors carry the failing phase (`auth-client`, `auth-getUser`, `db-getOrCreateAccount`) so logs show exactly which external service broke.

### Rules

- **Never return `String(err)` in a 5xx response.** Use `apiError()` or return a fixed message.
- **Always log before returning a 4xx validation error** — use `console.warn` so failures are visible in observability tooling.
- **Don't include database column names** in error messages exposed to clients — they're internal.

---

## 5. Validation

Use zod (or stack equivalent) for all input validation:

```typescript
import { z } from "zod";

const CreateFlowSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
  metadata: z.object({
    tags: z.array(z.string()).max(10).default([]),
  }).default({}),
});

export async function POST(req: Request) {
  try {
    const input = CreateFlowSchema.parse(await req.json());
    // … rest
  } catch (err) {
    if (err instanceof z.ZodError) {
      console.warn("[POST /api/flows] validation failed:", err.issues);
      return Response.json(
        { error: "Invalid input", code: "VALIDATION_ERROR", details: err.issues },
        { status: 400 }
      );
    }
    return apiError("POST /api/flows", err);
  }
}
```

Define schemas centrally where possible (`src/lib/api-schemas.ts`) so the same shape can be reused for typed request bodies, OpenAPI-style docs, and tests.

---

## 6. Pagination Contract

For list endpoints:

```
GET /api/flows?cursor=<opaque>&limit=25&sort=updatedAt:desc&filter[status]=active
```

Response:

```json
{
  "items": [ /* up to limit */ ],
  "nextCursor": "<opaque>",
  "totalCount": 142
}
```

Rules:
- **Default `limit` is 25**, max 100.
- **Cursor-based** for stable pagination across writes (offset-based skips/duplicates rows).
- **`nextCursor: null`** means no more pages.
- **`totalCount`** is exact when cheap; estimated (`~142`) when the table is large enough that exact COUNT is expensive. Document which it is per endpoint.
- **Sort is opt-in via query param** — `sort=field:asc` or `sort=field:desc`. Multiple comma-separated.
- **Filter is `filter[key]=value`** — flat, not nested.

For lists with bulk-action UX (data tables), see [`guide-tables.md`](guide-tables.md).

---

## 7. Webhook Endpoints

Webhooks (incoming from third-party services) follow stricter rules:

- **Always validate the signature** using the provider's HMAC verification before any other processing.
- **Idempotency is required** — webhook deliveries are at-least-once. Track delivery IDs and short-circuit on repeats.
- **Respond fast** — < 5 seconds. Defer heavy work to a background job.
- **Path:** `POST /api/webhooks/<provider>` (e.g., `/api/webhooks/github`, `/api/webhooks/stripe`)
- **Auth:** signature only — no session, no user context.

```typescript
export async function POST(req: Request) {
  const signature = req.headers.get("x-hub-signature-256");
  const body = await req.text();
  if (!verifyWebhookSignature(body, signature, secret)) {
    return Response.json({ error: "Invalid signature" }, { status: 401 });
  }
  // … process event
  return Response.json({ ok: true });
}
```

---

## 8. Streaming Responses

For SSE (server-sent events) — used by the proxy layer for LLM streaming:

```
Client → POST /api/proxy/model (fetch with ReadableStream)
       ← SSE: data: {"type":"start","provider":"anthropic","model":"…"}\n\n
       ← SSE: data: {"type":"token","text":"…"}\n\n  (per token)
       ← SSE: data: {"type":"usage","inputTokens":…,"outputTokens":…}\n\n
       ← SSE: data: {"type":"done","status":"success"}\n\n
```

Use `Response` with `TransformStream` — don't pull in Vercel AI SDK or similar abstractions. The event shape is normalized across providers.

For server-side code that needs the full buffered response (e.g., when storing the output rather than streaming to a client), use a buffered helper (`invokeProxyBuffered()`) that runs the same pipeline and returns the assembled text.

---

## 9. CORS

For internal frontend → backend (same-origin) calls, no CORS needed.

For public APIs or cross-origin clients:
- Allow specific origins, never `*` for credentialed requests
- Allow only the methods your endpoint actually serves
- Set `Access-Control-Max-Age` to a sensible value (e.g., 86400) to reduce preflight chatter

---

## 10. Health Check

Every project ships with `/api/healthz` (or stack equivalent):

```typescript
export async function GET() {
  try {
    // Lightweight DB query — proves connection is live
    await db.execute(sql`SELECT 1`);
    return Response.json({ status: "ok" });
  } catch (err) {
    return Response.json({ status: "degraded" }, { status: 503 });
  }
}
```

Used by uptime monitors, load balancer checks, and post-deploy smoke checks. Don't make it public-data-leaking — just status.

---

## Related

- [`guide-arch.md`](guide-arch.md) — proxy egress, ETag/If-Match rationale
- [`guide-db.md`](guide-db.md) — how the version column powers ETags
- [`guide-code.md`](guide-code.md) — `apiError()`, `resolveAccount()`, repository pattern
- [`guide-tables.md`](guide-tables.md) — list endpoint contracts for data tables
