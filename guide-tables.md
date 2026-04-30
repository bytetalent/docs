# Tables — Standards

Cross-stack standards for data tables in dashboards. Component contracts, tokens, and behavior for filtering, sorting, pagination, bulk actions, and cell content.

Stack-specific component implementations (React + Tailwind via `comp-data-table.tsx`, Astro Islands, ASP.NET Razor partials) live in each base/product's overlay.

---

## 1. Component Hierarchy

```
<FilterActionBar />          ← search, filters, bulk actions, page-size selector
<TableWrapper>               ← rounded-xl border, overflow-hidden
  <TableHeader />            ← sortable column headers
  <TableBody>
    <TableRow />             ← clickable rows, hover, selection
  </TableBody>
</TableWrapper>
<PaginationBar />            ← prev/next, page info, page size selector
```

This hierarchy is the contract. Names may vary slightly per stack, but the four roles (filter bar, header, rows, pagination) are always present.

---

## 2. FilterActionBar

Above the table, full-width.

```tsx
interface FilterActionBarProps {
  totalCount: number;
  selectedCount: number;
  onSelectAll: () => void;
  search?: { value: string; onChange: (v: string) => void; placeholder?: string };
  filters: FilterConfig[];
  bulkActions: BulkAction[];
  pageSize: number;
  onPageSizeChange: (n: number) => void;
}
```

### Layout

```
[ Select all  N selected ]  [ 🔍 Search… ]  [ Filter ▾ ][ Filter ▾ ]  →  [ Bulk Action ▾ ]
```

### Tokens

- Background: `bg-muted/40` (normal) → `bg-accent-blue-bg` when `selectedCount > 0`
- Height: `py-2.5 px-4`
- Filter chips: `rounded-lg border border-border px-2.5 py-1.5 text-xs`
- Bulk action trigger: `rounded-lg border border-border bg-card px-3 py-1.5 text-xs font-medium`
- Search input: `h-8 rounded-lg border border-border pl-8 pr-3 text-xs`
- Select-all checkbox: accent-primary

### Behavior

- Bulk action dropdown only visible when `selectedCount > 0`
- Destructive bulk actions (archive, delete) use `text-status-error-text` and are listed last, separated by a divider

---

## 3. TableHeader (Sortable Column Headers)

```tsx
interface ColumnDef<T> {
  key: keyof T | string;
  label: string;
  width?: string;       // Tailwind w-[Npx] or w-1/5 etc.
  sortable?: boolean;
  align?: "left" | "right" | "center";
}
```

### Tokens

- Row: `border-b border-border bg-accent-neutral-bg`
- Cell: `px-4 py-2.5 text-[10px] font-bold uppercase tracking-wider text-muted-foreground`
- Sort indicator: `ChevronUp` / `ChevronDown` (size 12), right of label
- Active sort column: `text-heading` label + filled chevron icon
- Sorted column header cell: `bg-primary/5`

### Behavior

- Click a sortable column → sort ascending; click again → descending; third click resets to default sort
- Default sort is decided per-page (typically `updatedAt desc`)
- Sort state lives in the URL query (`?sort=name:asc`) so it's bookmark-stable

---

## 4. TableRow

```tsx
interface TableRowProps<T> {
  row: T;
  columns: ColumnDef<T>[];
  selected?: boolean;
  onSelect?: () => void;
  onClick?: () => void;       // navigates to detail page
  actions?: RowAction[];      // renders ⋮ menu on hover
}
```

### Tokens

- Default: `border-b border-border/60`
- Hover: `hover:bg-muted/40`
- Selected: `bg-accent-blue-bg/40`
- Clickable: `cursor-pointer`
- Cell: `px-4 py-3 text-sm text-muted-foreground`
- Primary cell (name/title): `font-medium text-heading`
- Sub-label below primary: `text-[11px] text-muted-foreground truncate`

### Selection checkbox

`h-3.5 w-3.5 accent-primary` in a fixed `w-10` first column.

### Row actions (⋮ menu)

Visible on hover in the last column. Use the project's dropdown-menu primitive. Destructive actions rendered last with `text-status-error-text`.

---

## 5. PaginationBar

```tsx
interface PaginationBarProps {
  page: number;            // 0-indexed
  pageCount: number;
  totalItems: number;
  pageSize: number;
  onPageChange: (page: number) => void;
  onPageSizeChange: (size: number) => void;
}
```

### Layout

```
Showing 1–25 of 142     [ 25 ▾ per page ]     ← Prev  Page 1 of 6  Next →
```

### Tokens

- Container: `flex items-center justify-between border-t border-border px-4 py-2.5`
- Info text: `text-xs text-muted-foreground`
- Page size select: `rounded-md border border-border px-2 py-1 text-xs`
- Prev/Next buttons: `rounded-md border border-border px-2.5 py-1 text-xs font-medium text-heading disabled:opacity-40`

---

## 6. Standard Cell Content Templates

| Content type | Implementation |
|---|---|
| Name + sub-label | `<div className="font-medium text-heading">{name}</div><div className="text-[11px] text-muted-foreground truncate max-w-[240px]">{sub}</div>` |
| Status badge | `<StatusBadge status={status} />` from `comp-status-badge.tsx` |
| Type badge | `<TypeBadge type={type} />` from `comp-type-badge.tsx` |
| Pill (success) | `rounded-full bg-status-success-bg px-2.5 py-0.5 text-[11px] font-bold text-status-success-text` |
| Pill (error) | `rounded-full bg-status-error-bg px-2.5 py-0.5 text-[11px] font-bold text-status-error-text` |
| Pill (neutral) | `rounded-full bg-muted px-2.5 py-0.5 text-[11px] font-semibold text-muted-foreground` |
| Pill (brand) | `rounded-full bg-primary/10 px-2.5 py-0.5 text-[11px] font-bold text-primary` |
| Percent bar | `SuccessBadge` component |
| Timestamp | `text-xs text-muted-foreground` with relative time via `relativeTime()` helper |
| Monospace code | `font-mono text-xs font-semibold text-heading` |
| Tags / chip list | flex wrap, gap-1, each: `rounded-md bg-muted px-1.5 py-0.5 text-[10px] font-medium` |

---

## 7. Empty State

When there are no rows:

```tsx
<tr>
  <td colSpan={N} className="px-4 py-12 text-center">
    <p className="text-heading text-sm font-medium">No {entity} found</p>
    <p className="text-muted-foreground mt-1 text-xs">
      Try adjusting your filters or add a new {entity}.
    </p>
  </td>
</tr>
```

---

## 8. Constraints

- Tables must be horizontally scrollable on mobile: wrap in `overflow-x-auto`
- First column (name/title) should have `min-w-[180px]` to prevent excessive wrapping
- Never use fixed heights on table containers — let content determine height
- Page sizes: `[10, 25, 50, 100]` — default varies by page density (typically 25)

---

## 9. Server-side Pagination, Sorting, Filtering

All three live on the server, not the client:

- **Pagination** — cursor-based for stability across writes (see [`guide-api.md`](guide-api.md) §6)
- **Sorting** — sort param sent to API, applied in the query, response includes the rows for the requested page only
- **Filtering** — filter params sent to API, applied via `WHERE` clauses, response excludes filtered-out rows entirely

Don't fetch the entire dataset to the client and filter/sort/paginate in JavaScript — it doesn't scale and breaks at 1000+ rows.

The client-side hook (`useEntityList`) maintains the current sort/filter/page state and reflects it in the URL query string for stable links.

---

## 10. The `useEntityList` Hook (React Stacks)

All list pages use the `useEntityList` generic hook (`src/hooks/use-entity-list.ts`). Domain hooks wrap it:

```typescript
// src/hooks/use-flow-list.ts
export function useFlowList() {
  return useEntityList<Flow>({
    endpoint: "/api/flows",
    sortDefault: "updatedAt:desc",
    filterDefs: [
      { key: "status", label: "Status", values: ["active", "paused", "archived"] },
      { key: "type", label: "Type", values: [...] },
    ],
  });
}
```

`useEntityList` provides:

- `items` — the current page of rows
- `grouped` — grouped by a configurable key
- `stats` — summary counts (total, by status, etc.)
- `filters` / `setFilters`
- `sort` / `setSort`
- `selectedIds` / `toggleSelect` / `selectAll` / `clearSelection`
- `bulkOps` — bulk action handlers with optimistic UI + rollback on error
- `fetchError` — surface via `<FetchErrorBanner onRetry={reload} />`
- `pageSize` / `setPageSize`
- `page` / `setPage`
- `reload` — manual refetch

This way, every list page has consistent UX without re-implementing pagination/sort/filter/select state per entity.

---

## Related

- [`guide-api.md`](guide-api.md) — list endpoint contract (cursor, sort, filter)
- [`guide-design.md`](guide-design.md) — design tokens used in tables
- [`guide-code.md`](guide-code.md) — `useEntityList` and shared list-page patterns
