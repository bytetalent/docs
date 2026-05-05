---
name: Data Access Patterns
description: DB singleton, repository boundary, detail-page fetch with Promise.all/.catch, useEntityList, DetailPageShell.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Standardized data access patterns for server and client code. Source: guide-code.md.

## DB singleton (source: guide-code.md §6)

One shared `db` at module scope. Never call `drizzle()` / `postgres()` / `new Pool()`
inside a function body — each call opens a connection that is never released.

```typescript
// Good — module-level singleton
import { db } from "@/lib/db";
const rows = await db.select().from(flows).where(eq(flows.accountId, accountId));

// Bad — leaks a connection on every call
function getData() {
  const db = drizzle(postgres(process.env.DATABASE_URL!));
  return db.select().from(flows);
}
```

Scripts in `scripts/` are the sole exception (standalone CLI processes with their
own connection lifecycle).

## Repository boundary (source: guide-code.md §6)

- All data access goes through repositories in `src/lib/repositories/`.
- `src/app`, `src/components`, and `src/hooks` must **not** import from `src/data/*`
  or Supabase clients directly.
- When a feature needs new data, add/extend a repository method first; then
  consume it in hooks/components/pages.

```typescript
// Good — goes through repository
const flow = await flowsRepository.getById(id, accountId);

// Bad — UI layer imports raw data source
import { sampleFlows } from "@/data/sample-flows";
```

## List pages — useEntityList (source: guide-code.md §9)

All list pages use the `useEntityList` generic hook (`src/hooks/use-entity-list.ts`).
Domain hooks wrap it with entity-specific config.

```typescript
// Good — domain hook wraps useEntityList
const { items, fetchError, reload, filters, stats } = useFlowList();
if (fetchError) return <FetchErrorBanner message={fetchError.message} onRetry={reload} />;

// Bad — one-off fetch effect per page
const [flows, setFlows] = useState([]);
useEffect(() => { fetch("/api/flows").then(...) }, []);
```

## Detail pages — shell + error + not-found (source: guide-code.md §9)

- Use `DetailPageShell` for detail pages with the standard layout
  (stat cards + config fields + connected flows/runs).
- Guard missing entities with `NotFoundState`.
- Surface load failures via `FetchErrorBanner` with `onRetry`.

## Detail-page fetch pattern (source: guide-code.md §6)

Every detail page must follow this exact pattern:

```typescript
const [loaded, setLoaded] = useState(false);
const [fetchError, setFetchError] = useState<string | null>(null);

const loadData = () => {
  setLoaded(false);
  setFetchError(null);
  Promise.all([
    fetch(`/api/flows/${id}`).then((r) => (r.ok ? r.json() : undefined)),
    fetch("/api/runs")
      .then((r) => (r.ok ? r.json() : { items: [] }))
      .then((d) => d.items ?? []),
  ])
    .then(([flow, runs]) => {
      setFlow(flow);
      setRuns(runs);
      setLoaded(true);
    })
    .catch((err) => {
      console.error("[FlowPage] fetch failed:", err);
      setFetchError(err instanceof Error ? err.message : "Failed to load data");
      setLoaded(true); // must always be set true
    });
};

useEffect(loadData, [id]);

if (!loaded) return null;
if (fetchError) return <FetchErrorBanner message={fetchError} onRetry={loadData} />;
if (!flow) return <NotFoundState entity="Flow" />;
```

Rules:

- Every `Promise.all` / fetch chain must have a `.catch()` — without it a rejected
  promise leaves `loaded` as `false` and renders a permanent white screen.
- Secondary fetches must guard `r.ok` — return safe defaults instead of calling
  `r.json()` on an error response.
- Always set `loaded = true` in both success **and** error paths.
- Extract `loadData` (not inline in `useEffect`) so it can be passed as `onRetry`.
