---
name: Soft Delete API Resource
description: Soft-delete pattern — DELETE sets deletedAt or isArchived, never hard-deletes; notDeleted() helper filters all SELECTs.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
composes_with: [db-query-patterns, api-error-handler, resolve-account-route-handler]
nests: []
conflicts_with: []
---

No resource in the system is hard-deleted from the database. DELETE routes set a `deletedAt` timestamp or `isArchived` boolean. Every SELECT query filters out soft-deleted rows using the `notDeleted()` helper.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #11.

## Core rules

- Every deletable table has either `deleted_at TIMESTAMPTZ` (preferred) or `is_archived BOOLEAN NOT NULL DEFAULT FALSE`.
- The DELETE route handler sets `deletedAt = new Date()` (or `isArchived = true`) and bumps `version`. Returns `204 No Content`.
- Every SELECT query that should exclude deleted rows composes `notDeleted(table)` into the `where` clause.
- Do **not** call `db.delete()` in route handlers — the only delete is a soft update.
- UI labels: use "Archive" for consultant-facing resources. Use "Delete" only if the UX copy explicitly calls it permanent (and even then, it's still a soft delete at the DB layer).
- The `notDeleted()` helper is imported from `@/lib/db/soft-delete` — do not inline `isNull(t.deletedAt)` across query files.

## DB schema

```ts
// Preferred — timestamp-based
export const entities = pgTable("entities", {
  id: uuid("id").primaryKey().defaultRandom(),
  accountId: uuid("account_id").notNull(),
  name: text("name").notNull(),
  version: integer("version").notNull().default(1),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
  deletedAt: timestamp("deleted_at"), // nullable; null = not deleted
});

// Alternative — boolean-based (brand kits, batches)
export const batches = pgTable("batches", {
  id: uuid("id").primaryKey().defaultRandom(),
  isArchived: boolean("is_archived").notNull().default(false),
  // ...
});
```

## The `notDeleted()` helper

```ts
// src/lib/db/soft-delete.ts
import { isNull, eq } from "drizzle-orm";
import type { PgTableWithColumns } from "drizzle-orm/pg-core";

export function notDeleted<T extends { deletedAt: any }>(table: T) {
  return isNull(table.deletedAt);
}

export function notArchived<T extends { isArchived: any }>(table: T) {
  return eq(table.isArchived, false);
}
```

## DELETE route handler

```ts
// src/app/api/entities/[id]/route.ts
export async function DELETE(
  req: Request,
  { params }: { params: { id: string } },
) {
  const account = await resolveAccount();
  if (!account)
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  try {
    const existing = await db.query.entities.findFirst({
      where: (t, { and, eq }) =>
        and(
          eq(t.id, params.id),
          eq(t.accountId, account.accountId),
          notDeleted(t),
        ),
    });
    if (!existing)
      return NextResponse.json({ error: "Not found" }, { status: 404 });

    await db
      .update(entities)
      .set({
        deletedAt: new Date(),
        version: existing.version + 1,
        updatedAt: new Date(),
      })
      .where(eq(entities.id, params.id));

    return new NextResponse(null, { status: 204 });
  } catch (err) {
    return apiError("DELETE /api/entities/[id]", err);
  }
}
```

## SELECT — always compose notDeleted()

```ts
// Good — soft-delete filter in every query
const entity = await db.query.entities.findFirst({
  where: (t, { and, eq }) =>
    and(eq(t.id, id), eq(t.accountId, accountId), notDeleted(t)),
});

// Good — list query
const items = await db
  .select()
  .from(entities)
  .where(and(eq(entities.accountId, accountId), notDeleted(entities)));
```

```ts
// Bad — missing notDeleted, will return deleted rows
const entity = await db.query.entities.findFirst({
  where: (t, { and, eq }) => and(eq(t.id, id), eq(t.accountId, accountId)),
  // deletedAt check missing — returns soft-deleted rows
});
```

## Hard-delete is never used

```ts
// Bad — never call db.delete() in route handlers
await db.delete(entities).where(eq(entities.id, id)); // Wrong — use soft delete
```

## RLS consideration

When RLS policies filter by `deleted_at IS NULL`, the soft-delete is enforced at the DB level even if code misses the filter. Add the `deleted_at IS NULL` predicate to every SELECT RLS policy:

```sql
CREATE POLICY "entities_select" ON entities
  FOR SELECT USING (account_id = auth.uid() AND deleted_at IS NULL);
```

## Related skills

- `skills/paths/optimistic-crud-with-etag` — version bump pairs with soft delete
- `skills/rls-policies` — RLS policy templates that include deleted_at filters
- `skills/supabase-schema` — audit fields + soft-delete conventions
