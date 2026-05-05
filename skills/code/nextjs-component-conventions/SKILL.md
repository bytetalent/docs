---
name: Next.js Component Conventions
description: Server-first rendering, comp- naming, kebab-case files, folder structure, component size limits.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Component authoring conventions for Next.js App Router. Source: guide-code.md.

## Server-first rendering

- Server Components by default. Add `"use client"` only when required for:
  - `useState`, `useEffect`, or other React hooks
  - Browser APIs (`window`, `document`, `localStorage`)
  - Event handlers attached directly to the component
  - Third-party client-only libraries
- Keep components presentational where possible. Move behavior into hooks
  (`useX`) and pure functions in `lib/`.

```tsx
// Good — server component, no "use client"
export default async function FlowsPage() {
  const flows = await getFlows();
  return <FlowList flows={flows} />;
}

// Bad — entire page is client just for one button handler
"use client";
export default function FlowsPage() { ... }
```

## Folder structure (source: guide-code.md §3)

```
src/
  app/                       # routes, layouts, error/loading/not-found states
  features/<feature>/        # feature-first modules
    components/
    hooks/
    lib/
    types/
    actions/
  components/
    ui/                      # shared primitives (Button, Input, Badge) — no prefix
    shared/                  # composed components used by 2+ unrelated features
  lib/                       # utilities, db client, services
```

Promotion rule:

- Used by **one** feature → keep in `src/features/<feature>/`.
- Used by **2+ unrelated features** → promote to `components/shared` or `lib/`.
- UI primitives only → `components/ui`.

## File and export naming (source: guide-code.md §4)

- Files and folders: `kebab-case`.
- Component files: prefix `comp-` and domain-scope.
  - `comp-dash-sidebar.tsx`, `comp-flows-overview-tab.tsx`
- UI primitives in `components/ui/`: no prefix (`button.tsx`, `input.tsx`).
- React component exports: `PascalCase` (`StatusBadge`, `RunStatistics`).
- Variables / functions / hooks: `camelCase` (`useConnectionList`).
- Constants / env keys: `UPPER_SNAKE_CASE`.
- Types / interfaces / enums: `PascalCase`.

## Component size (source: guide-code.md §9)

- Keep page components under ~300 lines.
- Keep individual component files under ~150 lines (flagged in code review).
- When a page has tabs or large sections, extract them into sub-components in
  the domain's `components/` folder.

```tsx
// Good — tab extracted
// src/features/flows/components/comp-flows-overview-tab.tsx
export function FlowsOverviewTab({ flow }: { flow: Flow }) { ... }

// Bad — 500-line page component with everything inlined
export default function FlowPage() {
  // 500 lines of JSX...
}
```

## Accessibility (source: guide-code.md §11)

- Use semantic HTML before ARIA attributes.
- Every form control must have an accessible name (visible `<label>` preferred;
  `aria-label` only when no visible label is possible).
- Ensure keyboard navigation and visible focus styles.
- Icon-only buttons require accessible labels (`aria-label` or `<span className="sr-only">`).

```tsx
// Good
<button aria-label="Delete flow">
  <Trash2Icon className="h-4 w-4" />
</button>

// Bad — screen readers announce nothing
<button>
  <Trash2Icon className="h-4 w-4" />
</button>
```

## Route-level states (source: guide-code.md §12)

Every route directory must carry:

- `loading.tsx`
- `error.tsx`
- `not-found.tsx`
