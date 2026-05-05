---
name: Table Patterns
description: Server-side pagination/sorting/filtering, table component hierarchy, FilterActionBar, cell templates, and useEntityList hook conventions.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

## Component hierarchy

Every list/table view requires all four roles in this order:

```
FilterActionBar
  └── TableWrapper
        ├── TableHeader  (SortableTableHeader cells)
        └── TableBody    (rows + row-action menus)
PaginationBar
```

Never omit a layer. If there are no filters yet, render `FilterActionBar` with only the search input; if there is only one page of data, render `PaginationBar` in its disabled/hidden state so the layout is stable.

## FilterActionBar

`FilterActionBar` is the top control strip. It contains:

- Select-all checkbox
- Search input
- Filter chips (active filter values shown as dismissible pills)
- Bulk-action dropdown — **only visible when one or more rows are selected**

Destructive bulk actions (archive, delete) are placed **last** in the dropdown, separated by a divider, and colored `text-status-error-text`.

```tsx
<FilterActionBar
  selectedCount={selectedIds.size}
  onSelectAll={handleSelectAll}
  search={search}
  onSearchChange={setSearch}
  filters={activeFilters}
  onFilterRemove={removeFilter}
  bulkActions={[
    { label: "Export", onClick: handleExport },
    { type: "divider" },
    { label: "Delete", onClick: handleBulkDelete, className: "text-status-error-text" },
  ]}
/>
```

## Sortable column headers

Click behavior:

1. First click — sort ascending
2. Second click — sort descending
3. Third click — reset to default sort

Sort state lives in the URL query string (`?sort=name:asc`), not in component state. This makes links bookmarkable and stable for sharing.

Default sort is `updatedAt desc` unless explicitly specified for a given page.

## Row actions

The row-actions menu (⋮) is in the **last column** of each row and is **visible on hover**. Destructive actions are placed last with `text-status-error-text` and separated by a divider — same convention as bulk actions.

## Server-side data operations

Pagination, sorting, and filtering all live on the server. Never fetch all rows into the client and filter/sort/paginate client-side.

```ts
// api route — delegate sort/filter/cursor to DB query
const { sort, filter, cursor, limit } = parseTableParams(request);
const rows = await projectRepo.list({ accountId, sort, filter, cursor, limit });
```

Cursor-based pagination only (no offset). Page sizes: `[10, 25, 50, 100]`; default **25**.

## Horizontal scroll

- Wrap the table in `overflow-x-auto` so it scrolls on narrow screens
- First column `min-w-[180px]` so the primary identifier is always readable
- Never set a fixed height on the table container (that creates double scrollbars)

```tsx
<div className="overflow-x-auto">
  <table className="w-full">
    <thead>
      <tr>
        <th className="min-w-[180px]">Name</th>
        {/* ... */}
      </tr>
    </thead>
    {/* ... */}
  </table>
</div>
```

## Empty state

When the query returns zero rows, render a centered empty-state with:

- Heading: "No {entity} found"
- Contextual hint text (e.g., "Create your first project to get started.")

Never render an empty `<tbody>` without an empty state.

## Cell templates

Use these standard cell renderers consistently:

| Data type          | Renderer                               |
| ------------------ | -------------------------------------- |
| Status value       | `<StatusBadge status={...} />`         |
| Type/category      | `<TypeBadge type={...} />`             |
| Relative timestamp | `relativeTime(date)` utility           |
| Code / slug / ID   | `<code className="font-mono text-xs">` |
| Count / metric     | Plain number, right-aligned            |

Do not build custom inline badge styles per-page — always use `StatusBadge` / `TypeBadge`.

## useEntityList hook

All list pages use the `useEntityList` generic hook. Domain-specific list hooks wrap it:

```ts
// domain hook
export function useProjectList(filters: ProjectFilters) {
  return useEntityList<Project>({
    endpoint: "/api/projects",
    filters,
    defaultSort: "updatedAt:desc",
  });
}
```

The page component calls `useProjectList`, not `useEntityList` directly. This keeps per-entity config (endpoint, default sort, filter shape) out of page files.

## Sort and filter in URL

Both sort and filter parameters must be reflected in the URL query string for stable, bookmarkable links:

```
/space/projects?sort=name:asc&filter[status]=active&filter[type]=web
```

Use flat param syntax: `sort=field:dir`, `filter[key]=value` — no nested objects.
