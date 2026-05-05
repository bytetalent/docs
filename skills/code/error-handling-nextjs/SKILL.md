---
name: Error Handling (Next.js)
description: Route Handler error patterns, API response shapes, surfacing to client.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Error handling in Route Handlers:
- Validate input with zod; on parse failure, return 400 with a structured
  { error: "validation", issues: [...] }.
- auth() failure returns 401 immediately, before any DB query.
- Resource not found returns 404 with { error: "not_found" }.
- Optimistic concurrency mismatch returns 412 with { error: "version_mismatch", current_version: number }.
- Unexpected errors: log to Sentry; return 500 with { error: "internal" } —
  never leak stack traces or DB errors to the client.
- Hooks consume the same JSON shape consistently (useFetch normalises).
- Do NOT wrap every line in try/catch; let the runtime catch and the route
  handler's catch return the structured response.
