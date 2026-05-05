# API Standards

Cross-stack REST API conventions. Stack-specific routing details (Next.js Route Handlers, Astro endpoints, ASP.NET controllers) live in each base or product's `docs/` overlay.

---

## 1. REST Route Handlers

REST is the chosen API approach — not RPC, not tRPC, not GraphQL.

**Why REST over RPC / tRPC / Server Actions for the canonical API shape:**

1. ETags and `If-Match` are designed into the system — they belong in HTTP headers, not function calls.
2. OAuth callbacks and webhook endpoints require Route Handlers regardless — one consistent pattern.
3. Mobile apps and external integrations are realistic — same API works.

Server Actions are still acceptable for in-page mutations where an HTTP boundary adds no value (e.g., a profile-page form). But the canonical entity API surface is REST.

---

## 2–10. Rules (pointers to skills)

For route conventions, auth guard pattern, and error response shape, see [`skills/api/api-route-conventions`](../skills/api/api-route-conventions/SKILL.md).

For auth resolution (`resolveAccount()`) and the server-action auth guard, see [`skills/code/server-actions-auth`](../skills/code/server-actions-auth/SKILL.md) and [`skills/paths/resolve-account-route-handler`](../skills/paths/resolve-account-route-handler/SKILL.md).

For `apiError()` implementation and error code table, see [`skills/paths/api-error-handler`](../skills/paths/api-error-handler/SKILL.md).

For the full ETag / If-Match contract (read, write requirements, version-mismatch codes, server-side implementation), see [`skills/api/api-etag-optimistic-concurrency`](../skills/api/api-etag-optimistic-concurrency/SKILL.md).

For input validation with zod, see [`skills/api/api-input-validation`](../skills/api/api-input-validation/SKILL.md).

For pagination contract (cursor-based, limit, nextCursor, totalCount, sort, filter), see [`skills/api/api-pagination-and-filtering`](../skills/api/api-pagination-and-filtering/SKILL.md).

For webhook signature validation, idempotency, and fast-response rules, see [`skills/api/api-webhook-security`](../skills/api/api-webhook-security/SKILL.md).

For SSE streaming (TransformStream, normalized event shape, buffered helper), see [`skills/api/api-sse-streaming`](../skills/api/api-sse-streaming/SKILL.md).

---

## Related

- [`guide-arch.md`](arch.md) — proxy egress, ETag/If-Match rationale
- [`guide-db.md`](db.md) — how the version column powers ETags
- [`guide-code.md`](code.md) — `apiError()`, `resolveAccount()`, repository pattern
- [`guide-tables.md`](tables.md) — list endpoint contracts for data tables
