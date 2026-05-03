---
name: Optimistic CRUD With ETag
description: Optimistic concurrency via integer version + If-Match header — API returns ETag, client sends If-Match on PUT, handles 412 Precondition Failed.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Every entity that can be edited from multiple tabs (or concurrent users) uses integer version columns + ETag headers to prevent lost updates. This pattern is the standard update guard.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #10.

Related skill: `skills/api-etag-optimistic-concurrency` (contract specification).

## Core rules

- Every mutable entity table has a `version integer NOT NULL DEFAULT 1` column.
- Every GET/PUT/POST response that returns an entity sets `ETag: "<version>"` in the response headers.
- Every PUT that modifies an entity requires an `If-Match: "<version>"` request header.
- On version mismatch: return `412 Precondition Failed` with `{ error: "Version mismatch", code: "PRECONDITION_FAILED" }`.
- On missing entity: return `404 Not Found` — distinguish clearly from 412.
- The PUT handler increments `version` on every successful write.
- The client captures the ETag from the GET response and sends it on the PUT.
- The client shows a "version mismatch" toast on 412 and re-fetches to get the latest version.

## API route — GET (returns ETag)

```ts
// src/app/api/entities/[id]/route.ts
export async function GET(
  _req: Request,
  { params }: { params: { id: string } },
) {
  const account = await resolveAccount();
  if (!account)
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  try {
    const entity = await db.query.entities.findFirst({
      where: (t, { and, eq, isNull }) =>
        and(
          eq(t.id, params.id),
          eq(t.accountId, account.accountId),
          isNull(t.deletedAt),
        ),
    });
    if (!entity)
      return NextResponse.json({ error: "Not found" }, { status: 404 });

    return NextResponse.json(entity, {
      headers: { ETag: `"${entity.version}"` },
    });
  } catch (err) {
    return apiError("GET /api/entities/[id]", err);
  }
}
```

## API route — PUT (requires If-Match, returns 412 on mismatch)

```ts
export async function PUT(
  req: Request,
  { params }: { params: { id: string } },
) {
  const account = await resolveAccount();
  if (!account)
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const ifMatch = req.headers.get("If-Match");
  if (!ifMatch) {
    return NextResponse.json(
      { error: "If-Match header required" },
      { status: 428 },
    );
  }
  const clientVersion = Number(ifMatch.replace(/"/g, ""));

  const body = await req.json();
  const parsed = UpdateEntitySchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json({ error: "Invalid input" }, { status: 400 });
  }

  try {
    const existing = await db.query.entities.findFirst({
      where: (t, { and, eq, isNull }) =>
        and(
          eq(t.id, params.id),
          eq(t.accountId, account.accountId),
          isNull(t.deletedAt),
        ),
    });
    if (!existing)
      return NextResponse.json({ error: "Not found" }, { status: 404 });

    if (existing.version !== clientVersion) {
      return NextResponse.json(
        { error: "Version mismatch", code: "PRECONDITION_FAILED" },
        { status: 412 },
      );
    }

    const [updated] = await db
      .update(entities)
      .set({
        ...parsed.data,
        version: existing.version + 1,
        updatedAt: new Date(),
      })
      .where(eq(entities.id, params.id))
      .returning();

    return NextResponse.json(updated, {
      headers: { ETag: `"${updated.version}"` },
    });
  } catch (err) {
    return apiError("PUT /api/entities/[id]", err);
  }
}
```

## Client-side ETag plumbing

```tsx
"use client";

export function EntityDetailPage({ entityId }: { entityId: string }) {
  const [entity, setEntity] = useState<Entity | null>(null);
  const etagRef = useRef<string | null>(null);
  const [saving, setSaving] = useState(false);

  // On mount: fetch entity + capture ETag
  useEffect(() => {
    fetch(`/api/entities/${entityId}`).then(async (res) => {
      if (!res.ok) throw new Error("Failed to fetch");
      etagRef.current = res.headers.get("ETag");
      setEntity(await res.json());
    });
  }, [entityId]);

  const handleSave = async () => {
    if (!entity || !etagRef.current) return;
    setSaving(true);
    try {
      const res = await fetch(`/api/entities/${entityId}`, {
        method: "PUT",
        headers: {
          "Content-Type": "application/json",
          "If-Match": etagRef.current,
        },
        body: JSON.stringify({ name: entity.name /* ...edited fields */ }),
      });

      if (res.status === 412) {
        appToast.error("This record was updated elsewhere. Reloading…");
        // Re-fetch to get current version
        const refreshRes = await fetch(`/api/entities/${entityId}`);
        etagRef.current = refreshRes.headers.get("ETag");
        setEntity(await refreshRes.json());
        return;
      }
      if (!res.ok) throw new Error("Save failed");

      etagRef.current = res.headers.get("ETag");
      const updated = await res.json();
      setEntity(updated);
      appToast.success("Saved");
    } catch {
      appToast.error("Failed to save");
    } finally {
      setSaving(false);
    }
  };
}
```

## What NOT to do

```ts
// Bad — PUT without If-Match check (lost update possible)
export async function PUT(req: Request, { params }) {
  const body = await req.json();
  await db.update(entities).set(body).where(eq(entities.id, params.id));
  return NextResponse.json({ ok: true });
}
```

```tsx
// Bad — client ignores 412 and treats it as a generic error
if (!res.ok) appToast.error("Error"); // 412 needs a specific "reload" flow, not a generic error
```

## Related skills

- `skills/api-etag-optimistic-concurrency` — ETag/If-Match contract specification
- `skills/paths/soft-delete-api-resource` — soft delete pairs with this pattern
- `skills/paths/async-toast-mutation` — the save handler frame
