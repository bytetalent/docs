---
name: DB Query Patterns
description: Every read includes accountId + deletedAt filters; every write includes version match in WHERE + increment in SET + updatedAt + updatedBy; use Drizzle transactions for multi-write atomicity.
category: db
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Standard Drizzle query patterns for reads, writes, and transactions. Source: `bytetalent/docs/guide-db.md`.

## Read pattern

Every read query must include:

- `accountId` filter (app-layer defense in depth; RLS also enforces this)
- `deletedAt IS NULL` filter (soft-delete; must never return deleted rows)

```ts
// Good — both filters always present
const flow = await db.query.flows.findFirst({
  where: and(eq(flows.id, id), eq(flows.accountId, accountId), isNull(flows.deletedAt)),
});

// Good — list read
const items = await db.query.flows.findMany({
  where: and(eq(flows.accountId, accountId), isNull(flows.deletedAt)),
  orderBy: [desc(flows.updatedAt)],
});

// Bad — missing accountId filter
const flow = await db.query.flows.findFirst({
  where: eq(flows.id, id), // any account can read any row
});

// Bad — missing deletedAt filter
const items = await db.query.flows.findMany({
  where: eq(flows.accountId, accountId), // returns soft-deleted rows
});
```

## Write pattern

Every update must include:

- `version` match in `WHERE` (optimistic concurrency — see `api-etag-optimistic-concurrency` skill)
- `version + 1` in `SET`
- `updatedAt: new Date()` in `SET`
- `updatedBy: accountId` in `SET`
- Zero rows returned = conflict → return `VERSION_CONFLICT`

```ts
// Good — full write pattern
const result = await db
  .update(flows)
  .set({
    ...data,
    version:   sql`${flows.version} + 1`,
    updatedAt: new Date(),
    updatedBy: accountId,
  })
  .where(
    and(
      eq(flows.id, id),
      eq(flows.accountId, accountId),
      eq(flows.version, expectedVersion), // version guard
    )
  )
  .returning();

if (result.length === 0) {
  return apiError("VERSION_CONFLICT", "Flow was modified by another request");
}

// Bad — no version guard
await db.update(flows).set({ name: data.name }).where(eq(flows.id, id));
// Bad — no updatedAt/updatedBy
await db.update(flows).set({ name: data.name, version: sql`${flows.version} + 1` }).where(...);
```

## Soft-delete write

```ts
// Good — soft delete preserves the row
await db
  .update(flows)
  .set({
    deletedAt: new Date(),
    updatedAt: new Date(),
    updatedBy: accountId,
    version: sql`${flows.version} + 1`,
  })
  .where(and(eq(flows.id, id), eq(flows.accountId, accountId)));

// Bad — hard delete removes audit trail
await db.delete(flows).where(eq(flows.id, id));
```

## Transactions

Use Drizzle transactions for atomic multi-row writes. Two rules:

1. No network calls inside a transaction — transactions hold DB connections; slow network calls starve the pool
2. Keep transactions short — only the minimal set of writes that must be atomic

```ts
// Good — atomic multi-write with no network calls inside
await db.transaction(async (tx) => {
  await tx.insert(projects).values(projectData);
  await tx.insert(projectPhases).values(phasesData);
  await tx.insert(projectConnections).values(connectionsData);
});

// Bad — network call inside transaction
await db.transaction(async (tx) => {
  await tx.insert(projects).values(projectData);
  await fetch("https://api.github.com/..."); // holds the transaction open during network I/O
  await tx.insert(projectConnections).values(connectionsData);
});
```
