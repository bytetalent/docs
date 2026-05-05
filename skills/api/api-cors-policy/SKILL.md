---
name: API CORS Policy
description: Never use wildcard origin for credentialed requests; restrict allowed methods to what the endpoint serves; set preflight cache via Access-Control-Max-Age.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

CORS rules for Bytetalent API routes. Source: `bytetalent/docs/guide-api.md` §9.

## When to apply

Only public or cross-origin API endpoints need explicit CORS headers. Internal frontend → same-origin backend calls require no CORS configuration.

## Rules

1. **Never use `*` for credentialed requests.** Enumerate specific allowed origins via env var.
2. **Restrict `Allow-Methods` to what the endpoint actually serves.** Don't blanket-allow all verbs.
3. **Set `Access-Control-Max-Age`** to reduce preflight chatter (e.g., `86400` = 24 hours).

## Pattern

```ts
// Good — specific origin, restricted methods, preflight cache
const corsHeaders = {
  "Access-Control-Allow-Origin": process.env.ALLOWED_ORIGIN!,
  "Access-Control-Allow-Methods": "GET, POST",
  "Access-Control-Allow-Headers": "Content-Type, Authorization",
  "Access-Control-Max-Age": "86400",
};

export async function OPTIONS() {
  return new Response(null, { status: 204, headers: corsHeaders });
}

export async function GET(req: Request) {
  // ... handler logic
  return Response.json(data, { headers: corsHeaders });
}
```

```ts
// Bad — wildcard origin with credentials
"Access-Control-Allow-Origin": "*"  // never for credentialed requests

// Bad — allows every method regardless of what the endpoint handles
"Access-Control-Allow-Methods": "GET, POST, PUT, PATCH, DELETE, OPTIONS"
```

## Anti-patterns

- Setting `ALLOWED_ORIGIN` to a hardcoded string in source code — use env var so staging/prod differ.
- Skipping the `OPTIONS` preflight handler — browsers will block the actual request.
- Omitting `Access-Control-Max-Age` — causes unnecessary preflight on every request.

## See also

- [`skills/api/api-route-conventions`](../api-route-conventions/SKILL.md) — overall auth guard order; CORS lives after input validation
- `bytetalent/docs/guide-api.md` §9 — source section
