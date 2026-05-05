# Database Standards

Cross-stack schema conventions, audit fields, soft-delete, RLS templates, JSONB shape rules. Stack-specific ORM details (Drizzle for TS, EF Core for .NET) live in each base/product's overlay.

---

## 1. ORM Choice

- **TypeScript stacks:** Drizzle ORM. Edge-native, TypeScript-first, thin abstraction with full SQL escape hatches. Selected over Prisma to avoid the Prisma query engine binary at the edge.
- **.NET stacks:** Entity Framework Core, code-first migrations.

The principles in this guide apply regardless of ORM — the SQL shape is the contract.

---

## 2. Schema Conventions

### Required columns on every user-owned table

| Column | Type | Default | Purpose |
|---|---|---|---|
| `id` | uuid (text) | `crypto.randomUUID()` | Primary key |
| `account_id` | text | — | Owning account (RLS boundary) |
| `created_at` | timestamptz | `now()` | Audit |
| `created_by` | text | — | User ID who created |
| `updated_at` | timestamptz | `now()` | Audit |
| `updated_by` | text | nullable | User ID of last writer |
| `version` | integer | `1` | Optimistic concurrency control (ETag value) |
| `deleted_at` | timestamptz | nullable | Soft-delete marker |

**Why every column matters:**
- `account_id` is the RLS pivot. Every read must be scoped by it; every RLS policy filters by `auth.uid() = account_id`.
- `version` increments on every write, used as the `ETag` value in API responses (see [`guide-api.md`](guide-api.md)).
- `deleted_at IS NULL` is the active-row predicate. Partial unique indexes and most reads filter by it.
- `created_by` / `updated_by` carry the auth-provider user ID (e.g., Clerk `userId`) — useful for audit, undo flows, and notification routing.

### Drizzle definition example

```typescript
import { pgTable, text, integer, timestamp, jsonb } from "drizzle-orm/pg-core";

export const projects = pgTable("projects", {
  id: text("id").primaryKey().$defaultFn(() => crypto.randomUUID()),
  accountId: text("account_id").notNull().references(() => accounts.id),
  name: text("name").notNull(),
  slug: text("slug").notNull(),
  metadata: jsonb("metadata").$type<ProjectMetadata>(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  createdBy: text("created_by").notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  updatedBy: text("updated_by"),
  version: integer("version").notNull().default(1),
  deletedAt: timestamp("deleted_at", { withTimezone: true }),
});
```

Types are inferred from the Drizzle schema — don't write separate interface files.

---

## 3. Soft-Delete Convention

- Use `deleted_at timestamptz` — `NULL` means active, non-`NULL` means deleted.
- All reads filter by `WHERE deleted_at IS NULL` (or use a Drizzle helper like `whereActive(table)`).
- Hard delete reserved for: per-row erasure on user account deletion (GDPR), or rows that never had user data (e.g., expired job rows).
- **Partial unique indexes** for slug/name uniqueness scoped to active rows:

```sql
CREATE UNIQUE INDEX projects_account_slug_unique
  ON projects (account_id, slug)
  WHERE deleted_at IS NULL;
```

Lets a deleted project's slug be reused by a new project.

---

## 4. Optimistic Concurrency (Version Column)

Every user-owned entity carries `version integer` for optimistic locking. Increments on every write.

Update pattern:

```typescript
const result = await db
  .update(projects)
  .set({
    name: newName,
    version: sql`${projects.version} + 1`,
    updatedAt: new Date(),
    updatedBy: currentUserId,
  })
  .where(and(
    eq(projects.id, id),
    eq(projects.version, expectedVersion),
    isNull(projects.deletedAt),
  ))
  .returning();

if (result.length === 0) {
  throw new ConflictError("Project was modified by another request");
}
```

The version is exposed via API as the `ETag` header value. Writers must pass `If-Match: "<version>"`. See [`guide-api.md`](guide-api.md).

---

## 5. JSONB Conventions

JSONB is for shapes that are read together but vary per row — flexible config, structured payloads, AI output.

Rules:
- **Type the JSONB column** with `jsonb().$type<T>()` (Drizzle) or its EF Core equivalent. Never untyped.
- **Document the shape** in the consuming repo's `docs/app-schema.md`. JSONB shapes evolve, and the type alone doesn't capture optional vs. required nuance.
- **Don't index inside JSONB unless you're querying by that key.** Default to no indexes; add `CREATE INDEX … ON table USING gin ((column->'subKey'))` only when a query plan needs it.
- **Don't update sub-keys with raw SQL functions** (`jsonb_set`) when you can read-modify-write at the application layer with the typed shape. Easier to reason about, easier to change.

When to choose JSONB:
- Shape is heterogeneous across rows (different brands have different brand-kit fields)
- Shape evolves alongside the application (new optional keys without migrations)
- Read together with the parent row in 95%+ of queries

When NOT to choose JSONB:
- You'll join or filter by sub-keys frequently → use a child table
- The shape is genuinely uniform → use real columns
- You need RDBMS constraints (unique, FK) on a sub-key → impossible in JSONB

---

## 6. Row Level Security (RLS)

RLS is the **primary** security boundary, not application code. Every user-owned table enforces `account_id = auth.uid()` at the DB level — this cannot be bypassed regardless of application code.

### Required pattern

For each user-owned table:

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY projects_select_own
  ON projects FOR SELECT
  USING (account_id = auth.uid()::text);

CREATE POLICY projects_insert_own
  ON projects FOR INSERT
  WITH CHECK (account_id = auth.uid()::text);

CREATE POLICY projects_update_own
  ON projects FOR UPDATE
  USING (account_id = auth.uid()::text)
  WITH CHECK (account_id = auth.uid()::text);

CREATE POLICY projects_delete_own
  ON projects FOR DELETE
  USING (account_id = auth.uid()::text);
```

### Defense in depth

Drizzle queries must **also** filter by `accountId` from the server-side session — never rely on RLS alone.

```typescript
const account = await resolveAccount();
if (!account) return apiError("Unauthorized");

const projects = await db
  .select()
  .from(projects)
  .where(and(
    eq(projects.accountId, account.id),  // application-layer filter
    isNull(projects.deletedAt),
  ));
// RLS will also reject rows where account_id != auth.uid()
```

If application-layer filtering is forgotten, RLS catches it. If a Drizzle migration accidentally drops a policy, the app-layer filter still works. Both layers must be wrong for a leak to happen.

### Service-role / admin clients

For operations that must bypass RLS (system jobs, billing webhooks, migrations):

- Use Drizzle with documented intent in a code comment: `// service-role bypass: payment webhook applies invoice without user session`
- Don't create a parallel admin client just to bypass RLS — that's the deprecated pattern. Drizzle covers it.
- Storage operations (Supabase Storage signed URLs, file uploads) are the legitimate exception requiring an admin client wrapper, since Storage isn't expressible via Drizzle.

---

## 7. JWT Wiring (Clerk → Supabase)

For projects on Clerk + Supabase:

1. **Clerk dashboard** → JWT Templates → New → Supabase. Name: `supabase`.
2. **Supabase dashboard** → Authentication → Third Party Auth → Add provider → Clerk. Enter Clerk JWKS URL.

After this, Supabase's `auth.uid()` resolves to the Clerk user ID — RLS policies filtering by `auth.uid()` work without any client-side token plumbing beyond passing the Clerk session JWT.

---

## 8. Vault Secrets

Per-user secrets (OAuth tokens, BYOK API keys, bot tokens, SMTP passwords) use **Supabase Vault** (`pgsodium`):

- Encryption keys stay inside the database, not in env vars
- Decryption is a SQL function callable only under RLS — application code never sees raw ciphertext
- Vault columns are never selected by default in Drizzle queries

A dedicated `withSecrets()` helper in `src/lib/connections/secrets.ts` is the **only** path that retrieves decrypted values, and it's only importable from the proxy layer and provider adapters.

```typescript
// Wrong — exposes secret to application code
const conn = await db.select().from(connections).where(eq(connections.id, id));

// Right — use withSecrets only from proxy/adapter layer
const conn = await withSecrets((tx) =>
  tx.select(/* with decrypted secret */).from(connections).where(eq(connections.id, id))
);
```

Platform-level secrets (GitHub App private key, platform Anthropic key, Stripe keys, Clerk secret) stay in the host's env vars — Vault is for per-user/per-account secrets only.

See [`guide-arch.md`](guide-arch.md) §5.

---

## 9. Migrations

- Migrations are **committed source files** in a tracked folder (`drizzle/`, `migrations/`, etc.)
- One migration per schema change — never bundle unrelated changes
- Migration files include both schema changes AND RLS policy changes — they're part of the same atomic unit
- Apply order: dev → staging → prod, never skip
- Don't manually edit applied migrations — write a new one to fix

Migration commands are stack-specific and live in each project's `docs/guide-deploy.md`.

---

## 10. Migration journal — `when` semantics

Drizzle's migrator (used via `drizzle-orm/node-postgres/migrator`) decides whether to apply or skip each journal entry using this comparison:

```
apply if: lastApplied.created_at < migration.when
```

`when` is a Unix millisecond timestamp set at migration-generation time. `created_at` is the timestamp written into the `drizzle_migrations` table when the migration was actually applied on a given environment.

### Why this matters

If a newly authored migration entry has a `when` value **earlier than or equal to** the most-recently applied entry's `created_at`, Drizzle silently skips it. No error, no warning — the migration just never runs. The schema is missing the change, but the app deploys anyway.

This failure mode is silent and environment-specific: an entry may apply fine on a freshly wiped local DB (where nothing has been applied yet) but be silently skipped on staging or prod where an older migration with a higher `created_at` was applied first.

### Today's bug (2026-05-03)

Parallel workers generated journal entries 0040–0048 with `when` values in the 2025 range (~1.75e12 ms). The preceding entry 0039 had `when = 1777695000000` (2026). Any environment where 0039 was applied would have a `created_at` in the 2026 range — larger than the 2025 `when` values on 0040–0048. Drizzle skipped all nine entries silently.

Fix: PR #463 assigned each entry a `when` value that is strictly greater than its predecessor, restoring monotonicity. The entries were not yet applied on production, so no rollback was required.

### Correct authoring

Always set `when` to `Date.now()` **at the time you generate the migration**, not a hardcoded or copied value. `drizzle-kit generate` does this automatically — the silent-skip risk only arises when workers or humans manually construct or copy journal entries without updating `when`.

```
// Wrong — copied from a previous entry or hardcoded to a past date
{ "when": 1748000000000, ... }  // 2025 value copied by mistake

// Right — current millisecond timestamp at time of authoring
{ "when": 1777700000000, ... }  // generated by drizzle-kit or Date.now()
```

### Monotonicity check (CI)

`bt-ai-web` ships `scripts/check-journal-monotonic.mjs` which reads `drizzle/meta/_journal.json` and asserts strict monotonicity (`entry[i].when > entry[i-1].when`). It runs as the first step in `.github/workflows/ci.yml` and exits 1 with a named error if violated.

Any other Bytetalent project using Drizzle should copy this check.

### Related

- `bytetalent/docs/skills/drizzle-migration-patterns/SKILL.md` — pitfall callout with a pointer back here
- `bt-ai-web` PR #463 — the journal timestamp fix
- `bt-ai-web` PR that ships `check-journal-monotonic.mjs` — mechanical enforcement

---

## 11. Query Patterns

### Reads

- Prefer Drizzle's relational API (`db.query.*`) for joins and nested selects
- Use raw select builder (`db.select(...).from(...)`) when you need fine-grained control
- Always include `accountId` filter and `deletedAt IS NULL` filter

### Writes

- Use `db.insert/update/delete`
- Updates must always include version match in WHERE + version increment in SET
- Updates must always set `updatedAt: new Date()` and `updatedBy: currentUserId`
- Soft-delete sets `deletedAt: new Date()`, doesn't actually delete

### Transactions

- Use Drizzle transactions when multiple writes must be atomic
- Don't put network calls inside a transaction — keep them short
- Don't put long-running operations inside a transaction — they hold connections

### Connection pooling

See [`guide-code.md`](guide-code.md) §6 — single shared `db` singleton at module scope, never instantiated per call.

---

## Related

- [`guide-arch.md`](guide-arch.md) — connections framework, vault, ETag/If-Match
- [`guide-api.md`](guide-api.md) — how schemas surface in REST handlers
- [`guide-code.md`](guide-code.md) — repository pattern, data access boundary
