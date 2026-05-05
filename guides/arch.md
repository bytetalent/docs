# Architecture Standards

Cross-cutting architecture decisions that apply to every Bytetalent project. The principle: **one canonical path per concern, no parallel implementations.** Multiple paths for the same concern create drift, security gaps, and incident-shaped surprises.

Stack-specific routing details (Next.js App Router, Astro endpoints, ASP.NET controllers, Expo Router) live in each base or product's `docs/guide-stack.md` overlay.

---

## 1. Standardized Access Paths

Every cross-cutting concern has **one** canonical path. If you think you need a new path for an existing concern, propose it as an architectural change first — don't quietly add a parallel implementation.

**Access paths reference**: see [`skills/architecture/standardized-access-paths`](../skills/architecture/standardized-access-paths/SKILL.md).

### Why one path matters

Real example: when secret reads/writes were routed through both the Vault (via Drizzle direct SQL) **and** PostgREST (via `supabase-js`), the PostgREST path failed under load with a PGRST002 schema-cache miss while the Drizzle path worked. The two paths added no value — same DB, same ACL — but the second path created a real production incident. Single path eliminates this entire class of problem.

---

## 2. Proxy is the Sole Provider Egress

**Proxy egress rules and rationale**: see [`skills/architecture/proxy-egress`](../skills/architecture/proxy-egress/SKILL.md).

---

## 3. ETag / If-Match — Optimistic Concurrency

**ETag contract and implementation**: see [`skills/api/api-etag-optimistic-concurrency`](../skills/api/api-etag-optimistic-concurrency/SKILL.md).

---

## 4. Row Versioning

**Row versioning and query patterns**: see [`skills/db/db-query-patterns`](../skills/db/db-query-patterns/SKILL.md).

---

## 5. Connections Framework

**Connections framework (two-level model, variants, adapter interface, Vault storage)**: see [`skills/connections/connections-framework`](../skills/connections/connections-framework/SKILL.md).

**GitHub App vs. OAuth App decision**: see [`skills/connections/github-app-vs-oauth`](../skills/connections/github-app-vs-oauth/SKILL.md).

---

## 6. Real-time Updates

**Supabase Realtime pattern**: see [`skills/architecture/realtime-supabase`](../skills/architecture/realtime-supabase/SKILL.md).

---

## 7. Infrastructure Stack — Default Choices

**Service default choices and overlap decisions**: see [`skills/infra/infra-service-defaults`](../skills/infra/infra-service-defaults/SKILL.md).

---

## 8. Security Mindset

**Auth guard pattern**: see [`skills/code/server-actions-auth`](../skills/code/server-actions-auth/SKILL.md).

**Security review checklist**: see [`skills/security/security-review`](../skills/security/security-review/SKILL.md).

**Content Security Policy configuration**: see [`skills/security/csp-configuration`](../skills/security/csp-configuration/SKILL.md).

---

## 10. Related Standards

- [`guides/code.md`](code.md) — naming, component standards, security, testing
- [`guides/db.md`](db.md) — schema conventions, audit fields, soft-delete, RLS templates
- [`guides/api.md`](api.md) — REST handlers, ETag/If-Match contract, error response shape
- [`guides/design.md`](design.md) — tokens, typography, theming, stat cards, marketing pages
- [`guides/deploy.md`](deploy.md) — environment principles (commands in stack overlays)
