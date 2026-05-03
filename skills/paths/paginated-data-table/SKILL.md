---
name: Paginated Data Table
description: List-of-entities page with stat cards, toolbar (filters, bulk actions), DataTable rows, and empty state. Stat cards live inside a CollapsiblePanel when the page uses collapsible-section-stack.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 2
composes_with: [entity-list-hook, async-toast-mutation, fetch-error-retry, server-client-page-split, collapsible-section-stack]
nests: [modal-dialog-action]
conflicts_with: []
---

Every entity list page follows the same structural shape: stat summary cards → toolbar (count + select-all + filters + bulk actions) → data table rows → empty state. This pattern standardizes that shape.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #2.

## Core rules

- Use `DataTable` + `DataToolbar` components for full list surfaces (multi-select, bulk actions, filters).
- Use inline `<table>` + local search only for small, non-paginated admin lists.
- Stat cards row appears above the toolbar when the entity has meaningful aggregate counts.
- Row click navigates to the entity detail page — do not use action menus for primary navigation.
- Empty state must be present for both zero-records and zero-results-after-filter conditions.
- Bulk actions (activate / deactivate / archive) appear in the toolbar only when rows are selected.
- All filter state is URL-driven via `useSearchParams` so links are shareable.

## Page structure

```tsx
"use client";

export function EntityListPage() {
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
  } = useEntityList();

  if (fetchError) return <FetchErrorBanner onRetry={retry} />;

  return (
    <DashboardPageLayout
      title="Entities"
      action={<Button>+ New Entity</Button>}
    >
      {/* 1. Stat cards */}
      <div className="grid grid-cols-3 gap-4 mb-6">
        <StatCard label="Total" value={stats.total} />
        <StatCard label="Active" value={stats.active} />
        <StatCard label="Archived" value={stats.archived} />
      </div>

      {/* 2. Toolbar */}
      <DataToolbar
        count={items.length}
        selectedCount={selectedIds.size}
        filters={filters}
        onFiltersChange={setFilters}
        onSelectAll={selectAll}
        bulkActions={
          selectedIds.size > 0 ? (
            <BulkActionButtons
              onActivate={() => bulkAction("activate")}
              onArchive={() => bulkAction("archive")}
            />
          ) : null
        }
      />

      {/* 3. Table */}
      <DataTable
        rows={items}
        columns={ENTITY_COLUMNS}
        selectedIds={selectedIds}
        onToggleSelect={toggleSelect}
        onRowClick={(row) => router.push(`/space/entities/${row.id}`)}
        emptyState={<EmptyState message="No entities yet" />}
        isLoading={isLoading}
      />
    </DashboardPageLayout>
  );
}
```

## Column definition shape

```tsx
const ENTITY_COLUMNS: ColumnDef<Entity>[] = [
  {
    id: "select",
    cell: ({ row }) => <Checkbox checked={selectedIds.has(row.id)} />,
  },
  { accessorKey: "name", header: "Name" },
  {
    accessorKey: "status",
    header: "Status",
    cell: ({ row }) => <StatusBadge value={row.status} />,
  },
  {
    accessorKey: "createdAt",
    header: "Created",
    cell: ({ row }) => <DateCell value={row.createdAt} />,
  },
];
```

## What NOT to do

```tsx
// Bad — inline fetch in component, no toolbar, no empty state
export function BadEntityList() {
  const [items, setItems] = useState([]);
  useEffect(() => {
    fetch("/api/entities")
      .then((r) => r.json())
      .then(setItems);
  }, []);
  return (
    <ul>
      {items.map((i) => (
        <li key={i.id}>{i.name}</li>
      ))}
    </ul>
  ); // missing: error handling, empty state, bulk actions, URL-driven filters
}
```

## Inline table variant (small admin lists)

For lists under ~50 items with no bulk actions or complex filters, use a simple inline table:

```tsx
<table className="w-full text-sm">
  <thead>
    <tr className="border-b">
      <th className="text-left py-2">Name</th>
      <th className="text-left py-2">Status</th>
    </tr>
  </thead>
  <tbody>
    {items.map((item) => (
      <tr
        key={item.id}
        className="border-b hover:bg-muted/50 cursor-pointer"
        onClick={() => router.push(`/admin/entities/${item.id}`)}
      >
        <td className="py-2">{item.name}</td>
        <td className="py-2">
          <StatusBadge value={item.status} />
        </td>
      </tr>
    ))}
  </tbody>
</table>
```

## Stat-card placement — stat-card-in-panel pattern

When the list page is organized with `collapsible-section-stack`, stat cards go **inside a `<CollapsiblePanel>`** — not in a plain `<div>`. This keeps them collapsible when the user wants to focus on the table.

```tsx
// Good — stat cards wrapped in CollapsiblePanel
import { CollapsiblePanel } from "@/components/shared/comp-collapsible";
import { BarChart3 } from "lucide-react";

<CollapsiblePanel
  title="Statistics"
  icon={<BarChart3 className="w-4 h-4" />}
  defaultOpen={true}
>
  <div className="grid grid-cols-3 gap-4 py-2">
    <StatCard label="Total" value={stats.total} />
    <StatCard label="Active" value={stats.active} />
    <StatCard label="Archived" value={stats.archived} />
  </div>
</CollapsiblePanel>

{/* toolbar + table follow outside the panel */}
<DataToolbar ... />
<DataTable ... />
```

```tsx
// Bad — plain div for stat cards (use when there is no collapsible-section-stack context)
<div className="grid grid-cols-3 gap-4 mb-6">
  <StatCard label="Total" value={stats.total} />
  <StatCard label="Active" value={stats.active} />
  <StatCard label="Archived" value={stats.archived} />
</div>
// Only acceptable if the page does NOT use CollapsiblePanel for other sections.
// Once any section uses CollapsiblePanel, stat cards must also be in a panel.
```

The hierarchy when using `collapsible-section-stack`:

```
DashboardPageLayout
  └─ div.space-y-4
       ├─ CollapsiblePanel title="Statistics" icon={<BarChart3 />}  ← stat cards
       │    └─ div.grid.grid-cols-3
       │         ├─ StatCard "Total"
       │         ├─ StatCard "Active"
       │         └─ StatCard "Archived"
       ├─ DataToolbar  ← toolbar (outside panel)
       └─ DataTable    ← table rows (outside panel)
```

## Related skills

- `skills/paths/entity-list-hook` — the data layer that backs this component
- `skills/table-patterns` — server-side pagination, sorting, filtering
- `skills/paths/fetch-error-retry` — error banner + retry mechanism
- `skills/paths/collapsible-section-stack` — when stat cards live inside a CollapsiblePanel (stat-card-in-panel pattern)
