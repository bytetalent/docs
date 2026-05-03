---
name: Supabase Schema Patterns
description: Drizzle conventions, audit fields, version column, soft-delete, partial unique indexes, JSONB typing and when-to-use rules.
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 2
---

Drizzle + Supabase schema rules for this codebase. Source: `bytetalent/docs/guide-db.md`.

## Required columns

Every user-owned table must have all eight of these columns:

```ts
// Drizzle schema
export const projects = pgTable("projects", {
  id: text("id").primaryKey(), // app-generated (crypto.randomUUID())
  accountId: text("account_id")
    .notNull()
    .references(() => accounts.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  createdBy: text("created_by").notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  updatedBy: text("updated_by").notNull(),
  version: integer("version").notNull().default(1), // increments on every write
  deletedAt: timestamp("deleted_at", { withTimezone: true }), // nullable; soft-delete
});
```

- `version` increments on every write — used for ETag / If-Match optimistic locking (see `api-etag-optimistic-concurrency` skill)
- IDs are app-generated text (`crypto.randomUUID()`), not `uuid` sequences
- Enums use `pgEnum` and are exported alongside the table
- Relations are declared in `src/lib/db/relations.ts`, not inline in schema files

## Soft delete

Use `deleted_at timestamptz` for deletion. Hard delete is only for GDPR erasure or rows with no user data.

- **All reads must filter `WHERE deleted_at IS NULL`** (Drizzle: `isNull(table.deletedAt)`)
- Uniqueness constraints must be scoped to active rows only (see partial indexes below)

```ts
// Good — soft delete + filter
await db
  .update(projects)
  .set({ deletedAt: new Date(), updatedAt: new Date(), updatedBy: accountId })
  .where(and(eq(projects.id, id), eq(projects.accountId, accountId)));

// Read always excludes deleted
await db.query.projects.findMany({
  where: and(eq(projects.accountId, accountId), isNull(projects.deletedAt)),
});

// Bad — hard delete for user-owned data
await db.delete(projects).where(eq(projects.id, id));
```

## Partial unique indexes

Scope uniqueness constraints to active rows so a deleted name can be reused:

```ts
// Drizzle index definition
export const projectNameAccountIdx = uniqueIndex("projects_name_account_active")
  .on(projects.name, projects.accountId)
  .where(isNull(projects.deletedAt));

// Equivalent SQL in migration
CREATE UNIQUE INDEX projects_name_account_active
  ON projects (account_id, name)
  WHERE deleted_at IS NULL;
```

## JSONB columns

Type every JSONB column with `.$type<T>()` and document the shape in `docs/app-schema.md`.

```ts
// Good
type ProjectConfig = {
  defaultModel?: string;
  featureFlags?: Record<string, boolean>;
};

export const projects = pgTable("projects", {
  // ...
  config: jsonb("config").$type<ProjectConfig>().default({}),
});

// Bad — untyped JSONB
config: jsonb("config"),  // no type; drifts silently
```

### When to use JSONB

Use JSONB when **all** of these are true:

- Shape varies across rows (heterogeneous optional keys)
- Keys evolve over time without requiring a migration
- The column is read together with the parent row 95%+ of the time

### When NOT to use JSONB

Avoid JSONB when:

- Frequent joins or filters by sub-key are needed
- Shape is uniform across all rows (use typed columns instead)
- You need RDBMS constraints (FK, NOT NULL, check constraint) on a sub-key

### JSONB update

Read-modify-write at the application layer instead of `jsonb_set` — preserves type safety:

```ts
// Good — read-modify-write
const project = await db.query.projects.findFirst({ where: eq(projects.id, id) });
const newConfig = { ...project.config, defaultModel: "claude-opus-4-7" };
await db.update(projects).set({ config: newConfig }).where(eq(projects.id, id));

// Bad — jsonb_set bypasses TypeScript type
await db.execute(
  sql`UPDATE projects SET config = jsonb_set(config, '{defaultModel}', '"claude-opus-4-7"') WHERE id = ${id}`
);
```

### JSONB indexing

Do not index JSONB sub-keys unless queries filter by them. Default: no JSONB index.
