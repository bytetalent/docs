---
name: Route and Type Patterns
description: routes.ts centralization, type config registry, useTheme singleton, env.ts validation.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Conventions for route paths, type registries, theme, and environment variables.
Source: guide-code.md.

## Route centralization (source: guide-code.md §9)

All route paths live in `src/lib/routes.ts`. Never hardcode route strings
outside of `routes.ts`.

Use `detailRoute(base, id)` for building detail page URLs with correct encoding.

```typescript
// Good
import { ROUTES, detailRoute } from "@/lib/routes";
<Link href={detailRoute(ROUTES.FLOWS, flow.id)}>View flow</Link>

// Bad — hardcoded route string in component
<Link href={`/space/flows/${flow.id}`}>View flow</Link>
```

## Type config registry (source: guide-code.md §9)

Use `createTypeRegistry<T>()` from `src/lib/type-config.ts` for type/status/category
configurations. This provides a consistent `getConfig(key)` and `getAll()` interface.

All domain type files follow this pattern: `flow-types.ts`, `connection-types.ts`,
`model-types.ts`, etc.

```typescript
// Good
import { createTypeRegistry } from "@/lib/type-config";

const flowTypeRegistry = createTypeRegistry<FlowTypeConfig>([
  { key: "automation", label: "Automation", icon: Zap, color: "blue" },
  { key: "pipeline",   label: "Pipeline",   icon: GitBranch, color: "purple" },
]);

export const { getConfig: getFlowTypeConfig, getAll: getAllFlowTypes } = flowTypeRegistry;

// Bad — one-off object lookup per domain
const FLOW_TYPES: Record<string, { label: string }> = { ... };
function getFlowTypeLabel(key: string) { return FLOW_TYPES[key]?.label ?? key; }
```

## Theme singleton (source: guide-code.md §9)

Theme state is managed by the `useTheme` hook (`src/hooks/use-theme.ts`) — single
source of truth. It syncs between `localStorage`, Clerk metadata, and system
preference. Both TopBar and Profile page consume this hook; do not create separate
theme state in other components.

```typescript
// Good
import { useTheme } from "@/hooks/use-theme";
const { theme, setTheme } = useTheme();

// Bad — local useState + localStorage.setItem duplicating theme logic
const [theme, setTheme] = useState("light");
useEffect(() => {
  localStorage.setItem("theme", theme);
}, [theme]);
```

## Env validation (source: guide-code.md §9)

Required environment variables are validated at startup via `src/lib/env.ts`
(imported in root layout). Apps fail fast on missing config.

- Add new required vars to the `required` array in `env.ts`.
- Always access env vars via the exported `env` object, not `process.env.*` directly.

```typescript
// Good
import { env } from "@/lib/env";
const apiKey = env.STRIPE_SECRET_KEY;

// Bad — process.env accessed directly; no startup validation
const apiKey = process.env.STRIPE_SECRET_KEY!;
```
