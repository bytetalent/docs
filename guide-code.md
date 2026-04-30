# Code Standards

Universal coding standards for Bytetalent projects. Stack-specific patterns (App Router file conventions, Astro Islands, ASP.NET controllers, Expo Router) live in each base or product's own `docs/` overlay.

## 1. Principles

- Optimize for correctness, security, and maintainability over short-term speed.
- Prefer simple, explicit patterns over clever abstractions.
- Default to server-first architecture; use client code only where interaction requires it.
- Keep business logic testable and framework-agnostic where possible.

## 2. Tech Baseline

- TypeScript with `strict: true` is required for all TS/JS Bytetalent projects.
- ESLint + Prettier (or stack equivalent) enforced in CI.
- React Server Components by default in Next.js stacks; Astro Islands by default in Astro stacks; pure C# server-rendered or controller responses in ASP.NET stacks. The principle is the same: **server first, client only where interaction demands it.**
- Per-stack tooling versions, file conventions, and runtime details live in each stack's `docs/guide-stack.md` overlay.

## 3. Folder Structure (Web/Webapp Stacks)

For React/Next.js-based projects, use `src/` as the application root:

```text
src/
  app/                        # routes (Next.js App Router) or pages
  features/                   # feature-first modules (optional — small projects can skip)
    <feature>/
      components/
      hooks/
      lib/
      types/
      actions/
  components/
    ui/                       # shared design-system primitives (Button, Input, Badge)
    shared/                   # composed components used by multiple features
  lib/
    auth/
    db/
    utils/
  hooks/
  types/
public/
  images/
  icons/
```

Rules:
- One feature only → keep local in that feature.
- Used by 2+ unrelated features → promote to `components/shared` or `lib`.
- `components/ui` is for unstyled primitives only (Button, Input, etc.).

For non-React stacks (Astro, ASP.NET, Expo), the equivalent overlay in that stack's docs spells out the structural convention.

## 4. Naming Conventions

- Files/folders: `kebab-case`.
- React component files: prefixed `comp-` and domain-scoped (`comp-flows-overview.tsx`, `comp-billing-summary.tsx`).
- UI primitives in `components/ui/`: no prefix (`button.tsx`, `input.tsx`).
- Component exports: `PascalCase` (`StatusBadge`, `RunStatistics`).
- Variables / functions / hooks: `camelCase` (`useConnectionList`, `getConnectionIcon`).
- Constants / env keys: `UPPER_SNAKE_CASE`.
- Types / interfaces / enums: `PascalCase`.
- Domain entities: use canonical names — pick one per concept and stick with it. No "History" for what's actually "Runs," no "Integrations" for what's actually "Connections."

## 5. Component & Rendering Standards (React Stacks)

- Server Components by default in Next.js App Router stacks.
- Add `"use client"` only when required for:
  - client state/hooks (`useState`, `useEffect`, etc.)
  - browser APIs
  - event handlers
  - third-party client-only libraries
- Keep components presentational when possible.
- Move behavior/state orchestration into hooks (`useX`) and pure functions in `lib/`.
- Don't pass large domain objects through many layers; pass minimal typed props.

## 6. Data, Auth, and Security

### Auth resolution → `resolveAccount()`

Every Bytetalent webapp uses one canonical session-resolution helper named `resolveAccount()` (in `src/lib/session.ts` or its stack equivalent). It wraps the auth provider's session lookup plus account creation/lookup in your DB.

Phase tracking is required — `resolveAccount()` records which phase failed (`auth-client`, `auth-getUser`, `db-getOrCreateAccount`, etc.) so logs show exactly which external call broke when a 500 happens.

Never reach past `resolveAccount()` to call the auth provider directly except for narrowly-scoped operations that don't need an app account lookup (e.g. password verification).

### API errors → `apiError()`

All API route handlers / controllers that return an error must use the canonical `apiError()` helper (in `src/lib/api-error.ts` or its stack equivalent):

```typescript
import { apiError } from "@/lib/api-error";

export async function GET() {
  try {
    const account = await resolveAccount();
    if (!account) return Response.json({ error: "Unauthorized" }, { status: 401 });
    // ... data access ...
  } catch (err) {
    return apiError("GET /api/your-route", err);
  }
}
```

`apiError()` provides:
- **Structured JSON response**: `{ error: "human-readable", code: "AUTH_ERROR" | "DB_ERROR" | "INTERNAL_ERROR" | … }`
- **Safe messages**: 5xx never leaks stack traces, connection strings, or internals.
- **Full server logging**: complete error logged with stack for observability.
- **Error classification**: classifies into codes based on error shape.

Rules:
- **Never return `String(err)` in a 5xx response** — it leaks internal details.
- **Always log before returning a 4xx validation error** — use `console.warn` so failures appear in observability tooling.

### Database connections — singleton only

Every Bytetalent project uses a **single shared DB client** at module scope (e.g. `db` from `src/lib/db/index.ts`). Never call the connection constructor (`postgres()`, `drizzle()`, `new Pool()`, etc.) inside a function body — each call leaks a connection.

```typescript
// ✅ Correct — uses shared singleton
import { db } from "@/lib/db";
const rows = await db.select().from(accounts).where(eq(accounts.id, id));

// ❌ Wrong — leaks a connection on every call
function getDb() {
  return drizzle(postgres(process.env.DATABASE_URL!));
}
```

Standalone scripts in a `scripts/` folder are the sole exception — they manage their own connection lifecycle.

### RLS-first security

For projects with a database holding user data, Row Level Security is the **primary** security boundary, not application code. All user-owned tables must enforce `account_id = auth.uid()` (or stack-equivalent identity check) at the DB level. Application-layer filtering by `accountId` is required as defense in depth, but RLS is what cannot be bypassed.

See [`guide-db.md`](guide-db.md) for the RLS template + Drizzle conventions.

### Server actions / mutations

- Use route handlers / server actions / controllers for mutations — never direct browser-to-DB.
- Validate all input with `zod` (or stack-equivalent schema validator) before touching data.
- Perform authz checks inside the action/handler (not only in UI).
- Return typed success/error shapes; don't leak raw DB errors.

### Repository / data-access boundary

- All app data access goes through repositories in `src/lib/repositories/` (or stack equivalent).
- UI layers (`src/app`, `src/components`, `src/hooks`) must not import from `src/data/*` directly.
- UI layers must not import DB clients directly.
- Repository implementations are the only place where raw data sources are read.
- When a feature needs new data: add or extend a repository method first, then consume from hooks/components/pages.

### Data fetching error handling

**Detail-page client-side fetches:**

```typescript
const [loaded, setLoaded] = useState(false);
const [fetchError, setFetchError] = useState<string | null>(null);

const loadData = () => {
  setLoaded(false);
  setFetchError(null);
  Promise.all([
    fetch(`/api/entity/${id}`).then((r) => (r.ok ? r.json() : undefined)),
    fetch("/api/related")
      .then((r) => (r.ok ? r.json() : { items: [] }))
      .then((d) => d.items ?? []),
  ])
    .then(([entity, related]) => {
      setEntity(entity);
      setRelated(related);
      setLoaded(true);
    })
    .catch((err) => {
      console.error("[PageName] fetch failed:", err);
      setFetchError(err instanceof Error ? err.message : "Failed to load");
      setLoaded(true);
    });
};

useEffect(loadData, [id]);

if (!loaded) return null;
if (fetchError) return <FetchErrorBanner message={fetchError} onRetry={loadData} />;
if (!entity) return <NotFoundState entity="Entity" />;
```

Rules:
- **Every `Promise.all` / fetch chain must have a `.catch()`.** Without it, a rejected promise leaves `loaded` permanently false and the page renders a permanent white screen.
- **Secondary fetches must guard `r.ok`** — return safe defaults instead of calling `.json()` on error responses.
- **Always set `loaded = true` in both success and error paths.**
- **Use `FetchErrorBanner` with `onRetry`.** The `loadData` function is extracted so it can be passed as the retry handler.

**List-page fetches:** use a `useEntityList`-style domain hook that already handles `fetchError` state internally.

## 7. TypeScript Standards

- `strict: true` is required.
- No `any` (use `unknown`, generics, or proper models).
- `type` and `interface` are both fine pragmatically; pick what's clearer per case.
- Prefer discriminated unions for variant/state modeling.
- Export shared domain models from feature `types/` or `src/types/`.

## 8. Styling and Design System (React/Tailwind Stacks)

- Tailwind CSS v4+ with `@utility` for custom utilities.
- Use `cn()` for class composition.
- Use `cva` for variant-heavy components.
- Design tokens are CSS custom properties defined in `globals.css` (colors, spacing, typography).
- Reusable text utilities defined as `@utility` (e.g., `text-field-label`, `text-section-heading`).
- Charts and dynamic inline styles must use CSS variables only — no hex fallbacks. Use `var(--color-primary)`, not `var(--color-primary, #96c237)`.
- Use the shared stat-cards-grid component (`comp-stat-cards-grid.tsx`) for consistent stat card rendering across list pages.

See [`guide-design.md`](guide-design.md) for token taxonomy, theming sources of truth, stat-card anatomy, marketing spacing, and CTA discipline.

## 9. Established Patterns (React/Tailwind Stacks)

### List pages — `useEntityList`

All list pages use the `useEntityList` generic hook (`src/hooks/use-entity-list.ts`). Provides filtering, grouping, stats, selection, and optimistic bulk operations with rollback. Domain hooks (e.g., `useConnectionList`, `useFlowList`) wrap this with domain-specific config.

### Detail pages — shared shells

- `DetailPageShell` for standard detail-page layouts (stat cards + config fields + connected entities).
- `NotFoundState` for missing-entity guards.
- `FetchErrorBanner` with `onRetry` for data loading failures (see §6).
- `EmptyState` for empty filtered results.
- `ErrorBoundary` to wrap dashboard children at the layout level.

### Routes

Centralize all route paths in `src/lib/routes.ts`. Use `detailRoute(base, id)` for building detail URLs with proper encoding. Never hardcode route strings outside `routes.ts`.

### Type config registry

Use `createTypeRegistry<T>()` (from `src/lib/type-config.ts`) when defining type/status/category configurations. Provides consistent `getConfig(key)` and `getAll()` interface across all domain type files.

### Theme

Single source of truth: `useTheme` hook (`src/hooks/use-theme.ts`). Syncs between `localStorage`, the auth provider's user metadata, and system preference. Both top-bar and profile-page consume this hook; never create separate theme state.

### Page size

Keep page components under ~300 lines. When a page has tabs or large sections, extract them into sub-components in the domain's component folder (e.g., `comp-flows-overview-tab.tsx`).

### Environment

Required env vars validated at startup via `src/lib/env.ts` (imported in root layout). Add new required vars to the `required` array; access via the exported `env` object.

## 10. Security Headers

CSP, HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy are configured at the framework's headers config (e.g., `next.config.mjs` `headers()` for Next.js, Astro middleware for Astro, ASP.NET middleware for .NET).

When adding a new external service (analytics, CDN, OAuth provider), update CSP `connect-src` and any other relevant directives.

## 11. Accessibility

- Use semantic HTML before ARIA.
- Every form control has an accessible name.
- Visible `<label>` by default; `aria-label` only when no visible label is possible.
- Keyboard navigation + visible focus styles.
- Icon-only buttons require accessible labels.

## 12. Route & UX Reliability

- Provide route-level states: `loading.tsx`, `error.tsx`, `not-found.tsx` (or stack equivalent).
- Handle empty states explicitly.
- Optimistic UI only when rollback/error behavior is defined.

## 13. Performance

- Prefer Server Components / Astro Islands / server-rendered output where practical.
- Minimize client component boundaries and bundle size.
- Use `next/image` (Next.js) or `<Image>` (Astro) for images where appropriate.
- Cache reads intentionally; revalidate explicitly after mutations.

## 14. Testing

- Unit tests for pure logic (`lib`, mappers, validators).
- Integration tests for data access and server actions.
- E2E tests for critical flows: sign-in/out, protected route access, core CRUD paths.
- Fix flaky tests before merge.

See [`guide-testing.md`](guide-testing.md).

## 15. Observability & Errors

- Use `apiError()` in all route handler / controller catch blocks (§6).
- `resolveAccount()` includes phase tracking so logs show exactly which external service failed.
- Log actionable errors on the server with context (request/user/feature).
- Don't log secrets, tokens, or sensitive PII.
- Standardize user-facing error messages.

## 16. Dependency & Import Rules

- Prefer absolute imports with `@/` (or stack-equivalent path alias).
- Keep imports ordered; remove unused.
- New dependencies require justification: problem solved, size/runtime impact, maintenance status.

## 17. Definition of Done — PR Checklist

- Types pass (`tsc --noEmit` or stack equivalent).
- Lint passes.
- Tests pass for changed behavior.
- Authz and RLS implications reviewed.
- Loading/error/empty states handled.
- Accessibility checks completed for new UI.
- No secrets or sensitive data exposed.
- Repository boundary respected (no direct `@/data/*` or DB-client imports in app/components/hooks).
- New tables and marketing pages follow [`guide-design.md`](guide-design.md).

## 18. Quick Decision Rules

- One feature only → keep local in `src/features/<feature>`.
- Reused across features → `components/shared` or `lib`.
- UI primitives only → `components/ui`.
- Mutations with side effects → server action / handler / controller + validation + authz.
- Unsure between server/client → choose server first.
