# Tables — Standards

Cross-stack standards for data tables in dashboards. The canonical component rules live in [`skills/code/table-patterns`](../skills/code/table-patterns/SKILL.md) and [`skills/paths/entity-list-hook`](../skills/paths/entity-list-hook/SKILL.md). This guide covers the structural rationale and when to use each layer.

Stack-specific component implementations (React + Tailwind via `comp-data-table.tsx`, Astro Islands, ASP.NET Razor partials) live in each base/product's overlay.

---

## Why four fixed roles (not ad-hoc tables)

Every list page has the same four responsibilities: filtering/bulk actions, column headers with sort, rows with selection, and pagination. Encoding these as fixed component roles (FilterActionBar, TableHeader, TableRow, PaginationBar) rather than ad-hoc per-page tables gives consistent UX and lets the shared `useEntityList` hook drive all four without re-implementing state per entity.

## Why server-side pagination, sorting, and filtering

All three live on the server, not the client. Fetching the full dataset to the client and sorting/filtering in JavaScript doesn't scale and breaks at 1000+ rows. The client hook maintains sort/filter/page state in the URL query string for stable bookmarkable links; the server applies them in queries.

## Why `useEntityList` (React stacks)

All list pages use the `useEntityList` generic hook. Domain hooks wrap it with endpoint + sort defaults + filter definitions. This way, every list page gets consistent pagination, sort, filter, selection, bulk ops, and fetch-error behavior without re-implementing per entity.

---

## Canonical rules and patterns

**Component hierarchy, token specs, FilterActionBar, TableHeader, TableRow, PaginationBar, empty state, cell content templates, constraints:** see [`skills/code/table-patterns`](../skills/code/table-patterns/SKILL.md).

**`useEntityList` hook contract, domain hook pattern, grouped/stats/bulkOps:** see [`skills/paths/entity-list-hook`](../skills/paths/entity-list-hook/SKILL.md).

**Server-side pagination and filtering API contract:** see [`skills/api/api-pagination-and-filtering`](../skills/api/api-pagination-and-filtering/SKILL.md).

---

## Related

- [`guides/api.md`](api.md) — list endpoint contract (cursor, sort, filter)
- [`guides/design.md`](design.md) — design tokens used in tables
- [`guides/code.md`](code.md) — `useEntityList` and shared list-page patterns
