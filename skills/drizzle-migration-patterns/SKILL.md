---
name: Drizzle Migration Patterns
description: One migration file per schema change; never edit applied migrations; include both schema + RLS in the same migration; apply dev → staging → prod in sequence.
category: db
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Drizzle migration discipline for this codebase. Source: `bytetalent/docs/guide-db.md` and `docs/guide-setup.md` (local migration workflow).

## Core rules

- Migrations are committed source files in `src/lib/db/migrations/`
- One migration per atomic schema change
- Never manually edit an applied migration file — write a new one to fix it
- Every migration that adds a table must also include the matching RLS policies
- Apply in sequence: dev → staging → prod

## Workflow

```bash
# 1. Edit schema in src/lib/db/schema.ts
# 2. Generate the migration file
npm run db:generate

# 3. Review the generated SQL before applying
# 4. Apply to local dev DB
npm run db:migrate

# 5. Commit the generated migration file alongside the schema change
git add src/lib/db/migrations/ src/lib/db/schema.ts
```

## What goes in a migration

A migration for a new table should contain all three parts in the same file:

```sql
-- Part 1: Schema
CREATE TABLE projects (
  id           TEXT PRIMARY KEY,
  account_id   TEXT NOT NULL REFERENCES accounts(id),
  name         TEXT NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_by   TEXT NOT NULL,
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_by   TEXT NOT NULL,
  version      INTEGER NOT NULL DEFAULT 1,
  deleted_at   TIMESTAMPTZ
);

-- Part 2: Index (soft-delete scope)
CREATE UNIQUE INDEX projects_name_account_active
  ON projects (account_id, name)
  WHERE deleted_at IS NULL;

-- Part 3: RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users access their own projects"
  ON projects
  USING (account_id::text = auth.jwt()->>'sub');
```

## Adding columns to an existing table

When adding a nullable column to an existing table, always use `DEFAULT` or `NULL` — never add a `NOT NULL` column without a default to a populated table.

```sql
-- Good — nullable with a backfill default
ALTER TABLE projects ADD COLUMN archived_at TIMESTAMPTZ;

-- Good — NOT NULL with a safe default
ALTER TABLE projects ADD COLUMN tier TEXT NOT NULL DEFAULT 'free';

-- Bad — NOT NULL with no default on a table that has rows
ALTER TABLE projects ADD COLUMN tier TEXT NOT NULL; -- fails on populated tables
```

## Never edit applied migrations

If a migration was already applied (on any environment), do not edit it. The migration journal tracks applied files by content hash. Edits cause journal drift and break future `db:migrate` runs.

```bash
# Wrong — editing a previously applied file
vim src/lib/db/migrations/0012_add_projects.sql  # do NOT do this

# Right — create a follow-up migration
npm run db:generate  # generates 0013_fix_projects_index.sql
```

## Promotion sequence

Apply migrations on each environment before deploying the code that depends on them:

```
1. npm run db:migrate  (dev / local)
2. Deploy to staging → migrations run as part of deploy (or run manually)
3. Deploy to prod → migrations run as part of deploy (or run manually)
```

Never run schema-changing SQL by hand on production. RLS policy changes count as migrations too.
