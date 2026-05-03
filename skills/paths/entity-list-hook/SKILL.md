---
name: Entity List Hook
description: Three-layer generic→domain→page data-access hook for list surfaces — useEntityList base (pure state machine), domain hook owns the fetch, page calls only the domain hook.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 2
composes_with: [fetch-error-retry]
nests: []
conflicts_with: []
---

Every entity list page uses the same three-layer hook stack: a generic base hook (`useEntityList`) that is a **pure state machine** — never owns a `useEffect` or fetch — a domain-specific hook that closes the generics and owns the fetch, and the page that calls only the domain hook.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #3.

## Core rules

- `useEntityList<TItem, TFilters, TStats>` is the **generic base** — it is a pure state machine. It owns selection, filtering, pagination state, and error state. It does **not** own a `useEffect`, does not call `fetchFn`, and has no `isLoading` boolean.
- The domain hook (e.g., `useFlowList`) owns the `useEffect`, performs the fetch, and pushes results into the base hook via `setItems` / `setFetchError`.
- Pages call only the domain hook. They receive typed state + action callbacks; they never manage fetch or filter state themselves.
- URL-driven initial filter: domain hooks read `useSearchParams()` for initial filter values so deep-linked filtered views work.
- Fetch errors surface as `fetchError: Error | null` — the page renders `<FetchErrorBanner>` when set, calls `retry()` on user action. `retry()` increments `retryCount`, which the domain hook uses as a `useEffect` dependency to re-fetch.

## Layer 1 — Generic base hook (pure state machine, never called from pages)

```ts
// src/hooks/use-entity-list.ts
interface UseEntityListReturn<TItem, TFilters, TStats> {
  items: TItem[];
  stats: TStats | null;
  filters: TFilters;
  setFilters: (filters: TFilters) => void;
  setItems: (items: TItem[], stats: TStats) => void;
  setFetchError: (error: Error | null) => void;
  fetchError: Error | null;
  retryCount: number;
  retry: () => void;
  selectedIds: Set<string>;
  toggleSelect: (id: string) => void;
  toggleSelectAll: () => void;
}

export function useEntityList<TItem, TFilters, TStats>(
  defaultFilters: TFilters,
): UseEntityListReturn<TItem, TFilters, TStats> {
  const [items, setItemsState] = useState<TItem[]>([]);
  const [stats, setStatsState] = useState<TStats | null>(null);
  const [filters, setFilters] = useState<TFilters>(defaultFilters);
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
  const [fetchError, setFetchError] = useState<Error | null>(null);
  const [retryCount, setRetryCount] = useState(0);

  // No useEffect here — domain hook owns the fetch.

  const setItems = (newItems: TItem[], newStats: TStats) => {
    setItemsState(newItems);
    setStatsState(newStats);
    setFetchError(null);
  };

  const retry = () => setRetryCount((c) => c + 1);

  const toggleSelect = (id: string) => {
    setSelectedIds((prev) => {
      const next = new Set(prev);
      if (next.has(id)) next.delete(id);
      else next.add(id);
      return next;
    });
  };

  const toggleSelectAll = () => {
    setSelectedIds((prev) =>
      prev.size === items.length
        ? new Set()
        : new Set((items as { id: string }[]).map((i) => i.id)),
    );
  };

  return {
    items,
    stats,
    filters,
    setFilters,
    setItems,
    setFetchError,
    fetchError,
    retryCount,
    retry,
    selectedIds,
    toggleSelect,
    toggleSelectAll,
  };
}
```

**Key invariant:** `useEntityList` never owns a `useEffect` and never calls a fetch function. The `fetchFn` pattern shown in older versions of this skill has been removed.

## Layer 2 — Domain hook (owns the fetch)

```ts
// src/hooks/use-flow-list.ts
import { useSearchParams } from "next/navigation";
import { useEffect } from "react";
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
    status: initialStatus,
    search: "",
  });

  // Domain hook owns the useEffect + fetch.
  useEffect(() => {
    fetchFlows(base.filters)
      .then(({ items, stats }) => base.setItems(items, stats))
      .catch((err) => base.setFetchError(err));
  }, [base.filters, base.retryCount]);

  const bulkAction = async (action: "activate" | "deactivate" | "archive") => {
    const ids = [...base.selectedIds];
    await fetch("/api/flows/bulk", {
      method: "POST",
      body: JSON.stringify({ action, ids }),
    });
    base.retry(); // re-fetch after bulk action
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
    toggleSelectAll,
    bulkAction,
    fetchError,
    retry,
  } = useFlowList();
  // render DataTable, DataToolbar, stat cards
  // No isLoading — loading is not modeled in useEntityList.
  // Show a skeleton only if the domain hook explicitly tracks a loading boolean.
}
```

## What NOT to do

```tsx
// Bad — useEntityList accepting a fetchFn (old architecture, now removed)
const base = useEntityList({
  fetchFn: fetchFlows,       // Wrong — domain hook owns the fetch, not the base
  defaultFilters: { ... },
});

// Bad — using isLoading from useEntityList (field doesn't exist)
const { isLoading } = useEntityList(...); // isLoading doesn't exist on useEntityList

// Bad — calling selectAll (renamed)
base.selectAll(); // Wrong — use base.toggleSelectAll()

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
}

// Bad — calling useEntityList directly from page
export function BadPage() {
  const list = useEntityList({ status: "all", search: "" });
  // should be useFlowList() — domain hook encapsulates entity-specific config
}
```

## Closing generics — type signature for new entity hooks

When adding a new entity:

1. Define `Entity`, `EntityFilters`, `EntityStats` types (infer from Drizzle schema).
2. Create `useEntityList` → domain hook in `src/hooks/use-<entity>-list.ts`.
3. Domain hook owns the `useEffect` + fetch, calls `setItems` / `setFetchError`.
4. Page calls only the domain hook.

## Related skills

- `skills/paths/paginated-data-table` — the UI that consumes this hook
- `skills/paths/fetch-error-retry` — how the page renders `fetchError`
- `skills/data-access-patterns` — DB query layer that the fetch function calls
