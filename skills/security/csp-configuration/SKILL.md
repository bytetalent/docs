---
name: CSP Configuration
description: Content Security Policy update discipline — where CSP lives in each stack, which directives to touch when adding a new external service, and what not to loosen.
category: security
applicable_phases: [code_gen, arch_review]
applicable_stacks: [universal]
version: 1
---

CSP is the browser-enforced allow-list that restricts what origins scripts, styles, images, and connections can load from. When adding any new external service (analytics, CDN, OAuth provider, AI endpoint), updating CSP is a required step — not optional cleanup. Source: `bytetalent/docs/guides/arch.md` §8 and `bytetalent/docs/guides/code.md` §10.

## Where CSP lives per stack

| Stack | CSP location |
|---|---|
| Next.js (App Router) | `headers()` in `next.config.mjs` |
| Astro | Middleware (`src/middleware.ts`) |
| ASP.NET | `UseContentSecurityPolicy` middleware or response headers |

CSP is never set only in `<meta http-equiv>` tags — that form cannot cover all directive types and is bypassed by some injection vectors.

## How to apply when adding a new external service

1. Identify which directives the new service requires. Common mappings:

   | Service type | Directive(s) to update |
   |---|---|
   | API / data endpoint (fetch, XHR) | `connect-src` |
   | Third-party script (analytics, widget) | `script-src` |
   | Embedded iframe (OAuth popup, embed) | `frame-src` |
   | Font CDN | `font-src` |
   | Image CDN or avatar provider | `img-src` |
   | CSS CDN | `style-src` |

2. Add the minimum origin needed (scheme + host only — no path). Use exact domain, not a wildcard parent.

   ```
   # Good — minimum-scope allow
   connect-src 'self' https://api.example.com

   # Bad — wildcard that grants too much
   connect-src 'self' https://*.example.com
   ```

3. Test in the browser with DevTools open — CSP violations appear as console errors with the blocked URL. Fix until no new violations appear.

4. Update CSP and the service integration in the same PR so reviewers can see both changes together.

## Baseline directive shape

Every Bytetalent app starts from this conservative baseline and extends it with project-specific origins:

```
default-src 'self';
script-src 'self' <auth-provider-domain>;
connect-src 'self' <auth-provider-domain>;
style-src 'self' 'unsafe-inline';
img-src 'self' data: blob:;
font-src 'self';
frame-src 'none';
object-src 'none';
base-uri 'self';
form-action 'self';
```

`'unsafe-inline'` on `style-src` is a pragmatic concession for CSS-in-JS and Tailwind purge artifacts. It must never appear on `script-src`.

## Anti-patterns

- **`script-src 'unsafe-eval'`** — never allow `eval()` in production. If a dependency requires it, find an alternative.
- **`script-src 'unsafe-inline'`** — blocks XSS protection at the most critical directive. Use nonces or hashes if inline scripts are genuinely required.
- **`default-src *`** — a no-op CSP; provides no protection.
- **Adding a service's CDN domain to `script-src` without a Subresource Integrity (SRI) hash** — allows any script served from that CDN, not just the one you intend. Either use SRI or self-host the script.
- **Delaying CSP updates to a follow-up PR** — CSP gaps opened during a service integration window are the highest-risk period; close them in the same PR.

## Refs

- `bytetalent/docs/guides/arch.md` §8 — Security Mindset, Content Security Policy sub-section
- `bytetalent/docs/guides/code.md` §10 — Security Headers (full header set beyond CSP)
- `skills/security/security-review/SKILL.md` — cross-cutting security checklist that CSP compliance feeds into
