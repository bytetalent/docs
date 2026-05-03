---
name: Entity List Hook
description: Three-layer generic→domain→page data-access hook for list surfaces — useEntityList base, domain hook, page consumption.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Every entity list page uses the same three-layer hook stack: a generic base hook (`useEntityList`) that is never called directly from pages, a domain-specific hook that closes the generics with entity-specific types, and the page that calls only the domain hook.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #3.

## Core rules

- `useEntityList<TItem, TFilters, TStats>` is the **generic base** — it encapsulates selection, filtering, retry, fetch error, and optimistic mutations. Pages never import or call it directly.
- Each entity domain gets its own hook (e.g. `useFlowList`, `useClientList`) that wraps `useEntityList` with the entity's types, default filters, computed stats, and bulk-action wiring.
- Pages call only the domain hook. They receive typed state + action callbacks; they never manage fetch or filter state themselves.
- URL-driven initial filter: domain hooks read `useSearchParams()` for initial filter values so deep-linked filtered views work out of the box.
- Fetch errors surface as `fetchError: Error | null` — the page renders `<FetchErrorBanner>` when set, calls `retry()` on user action.

## Layer 1 — Generic base hook (not called directly)

```ts
// src/hooks/use-entity-list.ts
interface UseEntityListOptions<TItem, TFilters, TStats> {
  fetchFn: (filters: TFilters) => Promise<{ items: TItem[]; stats: TStats }>;
  defaultFilters: TFilters;
  filterPredicate?: (item: TItem, filters: TFilters) => boolean;
}

export function useEntityList<TItem, TFilters, TStats>(
  options: UseEntityListOptions<TItem, TFilters, TStats>,
) {
  const [items, setItems] = useState<TItem[]>([]);
  const [stats, setStats] = useState<TStats>(/* default */);
  const [filters, setFilters] = useState<TFilters>(options.defaultFilters);
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
  const [fetchError, setFetchError] = useState<Error | null>(null);
  const [retryCount, setRetryCount] = useState(0);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    setIsLoading(true);
    options
      .fetchFn(filters)
      .then(({ items: newItems, stats: newStats }) => {
        setItems(newItems);
        setStats(newStats);
        setFetchError(null);
      })
      .catch(setFetchError)
      .finally(() => setIsLoading(false));
  }, [filters, retryCount]);

  const retry = () => setRetryCount((c) => c + 1);
  const toggleSelect = (id: string) => {
    /* ... */
  };
  const selectAll = () => {
    /* ... */
  };

  return {
    items,
    stats,
    filters,
    setFilters,
    selectedIds,
    toggleSelect,
    selectAll,
    fetchError,
    retry,
    isLoading,
  };
}
```

## Layer 2 — Domain hook

```ts
// src/hooks/use-flow-list.ts
import { useSearchParams } from "next/navigation";
import { useEntityList } from "./use-entity-list";
import type { Flow, FlowFilters, FlowStats } from "@/lib/db/schema";

async function fetchFlows(filters: FlowFilters) {
  const res = await fetch(`/api/flows?status=${filters.status ?? ""}`);
  if (!res.ok) throw new Error("Failed to fetch flows");
  return res.json() as Promise<{ items: Flow[]; stats: FlowStats }>;
}

export function useFlowList() {
  const searchParams = useSearchParams();
  const initialStatus =
    (searchParams.get("status") as FlowFilters["status"]) ?? "all";

  const base = useEntityList<Flow, FlowFilters, FlowStats>({
    fetchFn: fetchFlows,
    defaultFilters: { status: initialStatus, search: "" },
  });

  const bulkAction = async (action: "activate" | "deactivate" | "archive") => {
    const ids = [...base.selectedIds];
    await fetch("/api/flows/bulk", {
      method: "POST",
      body: JSON.stringify({ action, ids }),
    });
    base.retry(); // refetch after bulk action
  };

  return { ...base, bulkAction };
}
```

## Layer 3 — Page consumption

```tsx
// src/app/space/flows/page-client.tsx
"use client";
import { useFlowList } from "@/hooks/use-flow-list";

export function FlowsPageClient() {
  const {
    items,
    stats,
    filters,
    setFilters,
    selectedIds,
    toggleSelect,
    selectAll,
    bulkAction,
    fetchError,
    retry,
    isLoading,
  } = useFlowList();
  // render DataTable, DataToolbar, stat cards
}
```

## What NOT to do

```tsx
// Bad — page manages its own fetch state
"use client";
export function BadFlowsPage() {
  const [flows, setFlows] = useState<Flow[]>([]);
  useEffect(() => {
    fetch("/api/flows")
      .then((r) => r.json())
      .then(setFlows);
    // no error handling, no retry, no filter URL sync, no stats
  }, []);
  return (
    <ul>
      {flows.map((f) => (
        <li key={f.id}>{f.name}</li>
      ))}
    </ul>
  );
}
```

```tsx
// Bad — calling useEntityList directly from page
export function BadPage() {
  const list = useEntityList<Flow, FlowFilters, FlowStats>({
    /* ... */
  });
  // should be useFlowList() — domain hook encapsulates entity-specific config
}
```

## Closing generics — type signature for new entity hooks

When adding a new entity:

1. Define `Entity`, `EntityFilters`, `EntityStats` types (infer from Drizzle schema).
2. Create `useEntityList` → domain hook in `src/hooks/use-<entity>-list.ts`.
3. Page calls only the domain hook.

## Related skills

- `skills/paths/paginated-data-table` — the UI that consumes this hook
- `skills/paths/fetch-error-retry` — how the page renders `fetchError`
- `skills/data-access-patterns` — DB query layer that the fetch function calls
