# Database Standards

Cross-stack schema conventions — audit fields, soft-delete, optimistic concurrency, JSONB shape rules, RLS, Vault secrets, and migrations. Rule details live in the skills; this guide covers the ORM choice rationale and points to each skill.

---

## 1. ORM Choice

- **TypeScript stacks:** Drizzle ORM. Edge-native, TypeScript-first, thin abstraction with full SQL escape hatches. Selected over Prisma to avoid the Prisma query engine binary at the edge.
- **.NET stacks:** Entity Framework Core, code-first migrations.

The principles in this guide apply regardless of ORM — the SQL shape is the contract.

---

## 2–6. Schema conventions, soft-delete, optimistic concurrency, JSONB, RLS

For required columns on every user-owned table (id, account_id, created_at, created_by, updated_at, updated_by, version, deleted_at), the soft-delete convention, the version-column update pattern, JSONB rules, and the RLS required pattern, see [`skills/db/db-query-patterns`](../skills/db/db-query-patterns/SKILL.md).

For RLS required policies and the defense-in-depth rule (always filter by accountId at both the application layer and DB layer), see [`skills/db/rls-policies`](../skills/db/rls-policies/SKILL.md).

---

## 7. JWT Wiring (Clerk → Supabase)

Stack-specific to the `nextjs-clerk-supabase` base. See §7 of [`bytetalent-nextjs-clerk-supabase-base/docs/guide-stack.md`](https://github.com/bytetalent/bytetalent-nextjs-clerk-supabase-base/blob/main/docs/guide-stack.md).

---

## 8. Vault Secrets

Per-user secrets (OAuth tokens, BYOK API keys) use Supabase Vault. The `withSecrets()` helper in `src/lib/connections/secrets.ts` is the only path that retrieves decrypted values. Platform-level secrets stay in host env vars.

See [`skills/connections/connections-framework`](../skills/connections/connections-framework/SKILL.md) for the full Vault storage rules.

---

## 9–10. Migrations and migration journal `when` semantics

For migration authoring principles (one migration per change, apply order, don't edit applied migrations), the journal `when` monotonicity requirement, and the CI monotonicity check, see [`skills/db/drizzle-migration-patterns`](../skills/db/drizzle-migration-patterns/SKILL.md).

The journal `when` incident (2026-05-03, PR #463) is documented in `bt-ai-web`'s commit history. The monotonicity check script (`scripts/check-journal-monotonic.mjs`) originated in `bt-ai-web` — any Drizzle project should copy it.

---

## 11. Query Patterns

See [`skills/db/db-query-patterns`](../skills/db/db-query-patterns/SKILL.md) for read, write, transaction, and connection-pooling conventions.

---

## Related

- [`guide-arch.md`](arch.md) — connections framework, vault, ETag/If-Match
- [`guide-api.md`](api.md) — how schemas surface in REST handlers
- [`guide-code.md`](code.md) — repository pattern, data access boundary
