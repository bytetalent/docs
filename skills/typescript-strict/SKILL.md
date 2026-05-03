---
name: TypeScript Strict Mode
description: Strict TS — no any, no implicit returns, exhaustive switch, Drizzle-inferred types only.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 2
---

TypeScript strict-mode rules:

## Core rules

- `tsconfig "strict": true` (all strict flags); no escape hatches.
- Never use `any`. If a value is truly unknown, use `unknown` and narrow.
- Narrow union types with discriminator fields, then exhaustive switch with
  a never-typed default for compile-time exhaustiveness checking.
- Do not use non-null assertion (`!`) outside of provably safe boundaries
  (e.g. JSX refs known to be attached). Prefer null guards.
- Function return types are inferred where simple; explicit when public API.

## DB type inference (source: guide-code.md)

- Infer types from Drizzle schema via `$inferSelect` / `$inferInsert`; do not
  write separate interface files that mirror table shapes.

```typescript
// Good
type Flow = typeof flows.$inferSelect;
type NewFlow = typeof flows.$inferInsert;

// Bad — duplicates Drizzle schema; drifts silently
interface Flow {
  id: string;
  accountId: string;
  name: string;
  // ...
}
```

## Domain entity naming (source: guide-code.md)

- Pick one canonical name per concept across types, routes, components, and copy.
- Do not use synonyms: `History` vs `Runs`, `Integrations` vs `Connections`,
  `Automations` vs `Flows`.
- Canonical set: **Flows, Runs, Connections, Assistants, Models, Notifications**.

## Import hygiene (source: guide-code.md)

- Prefer absolute imports with `@/`.
- Keep imports ordered and remove unused imports.
- New dependencies require explicit justification: problem solved, size/runtime
  impact, maintenance status.

```typescript
// Good
import { db } from "@/lib/db";
import { flows } from "@/lib/db/schema";

// Bad — relative path three levels deep
import { db } from "../../../lib/db";
```
