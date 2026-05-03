---
name: API ETag and Optimistic Concurrency
description: Every single-resource GET returns ETag; PUT/PATCH/DELETE require If-Match; version column increments on every write; zero rows = conflict.
category: api
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Optimistic concurrency is enforced end-to-end: schema level (version column), ORM level (WHERE clause), and HTTP level (ETag / If-Match). Per `bytetalent/docs/guide-arch.md` ETags and ORM sections.

## Schema

Every user-owned table carries a `version` integer, starting at 1, incremented on every write:

```ts
version: integer("version").notNull().default(1);
```

## HTTP contract

| Endpoint                    | Rule                                            |
| --------------------------- | ----------------------------------------------- |
| `GET /api/<entity>/:id`     | Must return `ETag: "<version>"` header          |
| `PUT / PATCH / DELETE`      | Must require `If-Match: "<version>"` header     |
| `If-Match` missing          | Return `428 Precondition Required`              |
| `If-Match` version mismatch | Return `412 Precondition Failed`                |
| `If-Match` matches          | Proceed; return `200` with new `ETag` on writes |
| `If-Match: *`               | Not accepted — specific version always required |

List endpoints (`GET /api/flows`) do **not** return per-item ETags. Fetch the individual resource before writing.

## ORM write pattern

Always include the version in `WHERE` and increment it in `SET`. Zero rows returned means a concurrent modification:

```ts
const result = await db
  .update(flows)
  .set({
    ...data,
    version: sql`${flows.version} + 1`,
    updatedAt: new Date(),
    updatedBy: accountId,
  })
  .where(and(eq(flows.id, id), eq(flows.version, expectedVersion)))
  .returning();

if (result.length === 0) {
  return apiError("VERSION_CONFLICT", "Flow was modified by another request");
}

return Response.json(result[0], {
  headers: { ETag: `"${result[0].version}"` },
});
```

## GET with ETag

```ts
export async function GET(req: Request, { params }: { params: { id: string } }) {
  const flow = await db.query.flows.findFirst({
    where: and(eq(flows.id, params.id), eq(flows.accountId, accountId)),
  });
  if (!flow) return apiError("NOT_FOUND", "Flow not found");
  return Response.json(flow, { headers: { ETag: `"${flow.version}"` } });
}
```

## Bad

```ts
// Never skip version in WHERE — no conflict detection
await db.update(flows).set({ name: data.name }).where(eq(flows.id, id));

// Never omit ETag on GET
return Response.json(flow); // missing ETag: "<version>" header

// Never accept If-Match: *
if (ifMatch === "*") proceed(); // specific version must be supplied
```
