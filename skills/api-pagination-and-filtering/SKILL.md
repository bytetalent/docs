---
name: API Pagination and Filtering
description: Cursor-based pagination (not offset), default limit 25 / max 100, nextCursor null when exhausted, flat sort/filter query params.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Pagination and filter/sort conventions for list endpoints. Source: `bytetalent/docs/guide-api.md` and `bytetalent/docs/guide-tables.md`.

## Cursor-based pagination

Use cursor-based pagination (not offset/page-number). Offset pagination produces duplicate or skipped rows when data is inserted between pages.

- Default limit: **25**
- Maximum limit: **100** (reject or clamp requests above this)
- Response field: `nextCursor: string | null` — `null` when no more pages exist
- Cursor value: opaque to the client; typically a base64-encoded `{ id, updatedAt }` pair

```ts
// Response shape
{
  "items": [...],
  "nextCursor": "eyJpZCI6IjEyMyIsInVwZGF0ZWRBdCI6IjIwMjYtMDEtMDFUMDA6MDA6MDBaIn0=",
  "total": 150   // optional — omit if expensive
}

// Last page
{
  "items": [...],
  "nextCursor": null
}
```

```ts
// Route Handler
export async function GET(req: Request) {
  const url = new URL(req.url);
  const rawLimit = Number(url.searchParams.get("limit") ?? "25");
  const limit = Math.min(Math.max(rawLimit, 1), 100); // clamp 1–100
  const cursor = url.searchParams.get("cursor") ?? null;

  const { accountId } = await resolveAccount();

  const where = cursor
    ? and(eq(flows.accountId, accountId), isNull(flows.deletedAt), lt(flows.updatedAt, decodeCursor(cursor)))
    : and(eq(flows.accountId, accountId), isNull(flows.deletedAt));

  const items = await db.query.flows.findMany({
    where,
    limit: limit + 1, // fetch one extra to detect next page
    orderBy: [desc(flows.updatedAt)],
  });

  const hasMore = items.length > limit;
  const page = hasMore ? items.slice(0, limit) : items;
  const nextCursor = hasMore ? encodeCursor(page[page.length - 1]) : null;

  return Response.json({ items: page, nextCursor });
}
```

## Sort parameters

Flat query param: `sort=field:asc` or `sort=field:desc`. No nested objects.

```
GET /api/flows?sort=updatedAt:desc
GET /api/flows?sort=name:asc
```

Default sort is `updatedAt desc` unless explicitly overridden per-resource.

## Filter parameters

Flat query params: `filter[key]=value`. No nested JSON in query string.

```
GET /api/flows?filter[type]=automation
GET /api/flows?filter[status]=active&sort=name:asc
```

## Server-side only

Pagination, sorting, and filtering all run on the server. Never fetch all rows then filter or sort client-side.

## Bad

```ts
// Never use offset — skips rows on concurrent inserts
const items = await db.query.flows.findMany({
  offset: page * limit, // offset pagination
  limit,
});

// Never filter client-side
const allFlows = await fetchAllFlows();
const filtered = allFlows.filter((f) => f.type === type); // fetches everything
```
