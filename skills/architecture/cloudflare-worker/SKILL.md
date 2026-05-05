---
name: Cloudflare Worker Patterns
description: Form handlers, secrets via wrangler env, edge-ready code.
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [astro-cloudflare]
version: 1
---

Cloudflare Worker conventions for brochure projects:
- Form handlers (contact, newsletter) live as Workers, separate from the
  static site build.
- Secrets (Resend API key, Turnstile secret) bound via wrangler.toml; never
  inlined.
- Always validate input with a small zod-equivalent or hand-rolled schema —
  never trust form payloads.
- Honeypot fields + Turnstile token verification before any outbound call.
- Return JSON with explicit { ok: boolean, error?: string }; never leak
  upstream provider error details to the client.
- Rate limit via Cloudflare's built-in rules or KV-based counter.
