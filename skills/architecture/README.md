# Architecture

Structural patterns for Next.js, Astro, Cloudflare Workers, Supabase, and template development.

## Skills

- [astro-static](astro-static/SKILL.md) — Brochure-friendly Astro conventions — content collections, MDX, no JS.
- [base-development](base-development/SKILL.md) — How to structure, version, and evolve a Bytetalent base app — manifest keys, semver rules, CI smoke test, lessons log, and new-base recipe.
- [cloudflare-worker](cloudflare-worker/SKILL.md) — Form handlers, secrets via wrangler env, edge-ready code.
- [nextjs-app-router](nextjs-app-router/SKILL.md) — Folder structure, server/client split, route handlers — Next.js 16 App Router.
- [proxy-egress](proxy-egress/SKILL.md) — All LLM and tool provider calls route through POST /api/proxy/... for credential resolution, audit logging, and prompt assembly. Direct provider fetch is never allowed outside adapter.test().
- [standardized-access-paths](standardized-access-paths/SKILL.md) — One canonical path per concern — Drizzle for DB+Vault, resolveAccount for auth, apiError for errors, proxy for provider egress. Never add a parallel implementation.
- [supabase-schema](supabase-schema/SKILL.md) — Drizzle conventions, audit fields, version column, soft-delete, partial unique indexes, JSONB typing and when-to-use rules.
- [template-development](template-development/SKILL.md) — How to structure, pair, version, and lint a Bytetalent feature template — anatomy, feature.json fields, bundle file actions, token bindings, and the pairing rule.
