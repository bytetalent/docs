---
name: Fetch Error Retry
description: Every data-fetching page renders a persistent FetchErrorBanner with a retry button when the initial load fails — standard error recovery path.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

When a page's initial data fetch fails, the user should see a clear error message with a retry option — not a blank page or a spinner that never resolves. This pattern standardizes that recovery flow.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #17.

## Core rules

- Every data-fetching page component has a `fetchError: Error | null` state.
- When `fetchError` is non-null, render `<FetchErrorBanner onRetry={retry} />` — either above content (inline) or replacing the content area.
- The `retry` function bumps a counter that triggers a `useEffect` re-run — it does not duplicate fetch logic.
- `FetchErrorBanner` is the shared component from `@/components/shared/comp-fetch-error` — do not inline custom error UI.
- Never swallow fetch errors silently (no empty `catch {}` blocks in data-loading hooks).
- If the entity is not found (404), render `<NotFoundState />` instead of `FetchErrorBanner`.

## Hook-level error surface

Fetch errors bubble up from the hook to the page as `fetchError`:

```ts
// In the data hook
const [fetchError, setFetchError] = useState<Error | null>(null);
const [retryCount, setRetryCount] = useState(0);

useEffect(() => {
  setIsLoading(true);
  fetch(`/api/entities/${entityId}`)
    .then(async (res) => {
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      setEntity(await res.json());
      setFetchError(null);
    })
    .catch((err) => setFetchError(err))
    .finally(() => setIsLoading(false));
}, [entityId, retryCount]);

const retry = () => setRetryCount((c) => c + 1);

return { entity, fetchError, retry, isLoading };
```

## Page-level rendering — inline banner (list pages)

For list pages, the error banner appears above the toolbar/table:

```tsx
"use client";

export function EntitiesPageClient() {
  const { items, fetchError, retry, isLoading } = useEntityList();

  if (fetchError) {
    return (
      <DashboardPageLayout title="Entities">
        <FetchErrorBanner onRetry={retry} />
      </DashboardPageLayout>
    );
  }

  return (
    <DashboardPageLayout title="Entities">
      <DataToolbar /* ... */ />
      <DataTable /* ... */ />
    </DashboardPageLayout>
  );
}
```

## Page-level rendering — full content replacement (editor pages)

For detail/editor pages, the error replaces the entire content area:

```tsx
export function EntityDetailClient({ entityId }: { entityId: string }) {
  const { entity, fetchError, retry, isLoading } = useEntityDetail(entityId);

  if (isLoading) return <LoadingSpinner />;
  if (fetchError)
    return (
      <DashboardPageLayout title="Entity">
        <FetchErrorBanner onRetry={retry} />
      </DashboardPageLayout>
    );
  if (!entity) return <NotFoundState />;

  return (
    <DashboardPageLayout title={entity.name}>
      {/* entity content */}
    </DashboardPageLayout>
  );
}
```

## FetchErrorBanner component shape

```tsx
// comp-fetch-error.tsx
interface FetchErrorBannerProps {
  onRetry: () => void;
  message?: string;
}

export function FetchErrorBanner({
  onRetry,
  message = "Something went wrong loading this page.",
}: FetchErrorBannerProps) {
  return (
    <div className="flex flex-col items-center gap-4 py-12 text-center">
      <AlertCircle className="w-10 h-10 text-destructive" />
      <div>
        <p className="font-medium">Failed to load</p>
        <p className="text-sm text-muted-foreground mt-1">{message}</p>
      </div>
      <Button variant="outline" onClick={onRetry}>
        Try again
      </Button>
    </div>
  );
}
```

## What NOT to do

```tsx
// Bad — swallowing the error, user sees blank page
useEffect(() => {
  fetch(`/api/entities`)
    .then((r) => r.json())
    .then(setEntities)
    .catch(() => {}); // silent failure — user sees nothing
}, []);
```

```tsx
// Bad — showing a toast only, no way to recover
.catch((err) => {
  appToast.error("Failed to load"); // toast disappears, no retry mechanism
});
```

```tsx
// Bad — custom inline error UI instead of shared component
{
  fetchError && (
    <div className="text-red-500">
      Error: {fetchError.message}
      <button onClick={retry}>Retry</button>
    </div>
  );
}
// Use <FetchErrorBanner onRetry={retry} />
```

## Related skills

- `skills/paths/entity-list-hook` — provides `fetchError` + `retry` from `useEntityList`
- `skills/paths/async-toast-mutation` — for mutation errors (not initial load errors)
