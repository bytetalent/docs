# API

Standards for REST routes, input validation, pagination, streaming, webhooks, and concurrency patterns.

## Skills

- [api-etag-optimistic-concurrency](api-etag-optimistic-concurrency/SKILL.md) — Every single-resource GET returns ETag; PUT/PATCH/DELETE require If-Match; version column increments on every write; zero rows = conflict.
- [api-input-validation](api-input-validation/SKILL.md) — Validate all inputs with zod; centralize schemas in api-schemas.ts; use the canonical error code set; console.warn before 4xx; never leak DB column names in 5xx.
- [api-pagination-and-filtering](api-pagination-and-filtering/SKILL.md) — Cursor-based pagination (not offset), default limit 25 / max 100, nextCursor null when exhausted, flat sort/filter query params.
- [api-route-conventions](api-route-conventions/SKILL.md) — REST-only routes, file layout for list vs. detail, auth guard order (validate → auth → authz → logic), CORS rules, and health-check endpoint.
- [api-sse-streaming](api-sse-streaming/SKILL.md) — Use Response + TransformStream for SSE; do not pull in Vercel AI SDK; normalize event shape across providers.
- [api-webhook-security](api-webhook-security/SKILL.md) — Validate provider HMAC signature first on every inbound webhook; require idempotency via delivery ID; respond within 5 seconds.
