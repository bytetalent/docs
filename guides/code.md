# Code Standards

Universal coding standards for Bytetalent projects. This guide is a **map** — it points you to the curated set of skills that contain the authoritative rules. Stack-specific patterns (App Router file conventions, Astro Islands, ASP.NET controllers, Expo Router) live in each base or product's own `docs/` overlay.

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

---

## Skills map

### Structure, naming, and components

- **Folder structure, naming conventions, component/rendering standards, page size**: see [`skills/code/nextjs-component-conventions`](../skills/code/nextjs-component-conventions/SKILL.md).

### TypeScript

- **Strict mode, type discipline, import rules**: see [`skills/code/typescript-strict`](../skills/code/typescript-strict/SKILL.md).

### Auth and data access

- **Auth resolution (`resolveAccount()`)**: see [`skills/paths/resolve-account-route-handler`](../skills/paths/resolve-account-route-handler/SKILL.md).
- **API errors (`apiError()`)**: see [`skills/paths/api-error-handler`](../skills/paths/api-error-handler/SKILL.md).
- **Database singleton, query patterns, soft-delete, versioning, JSONB**: see [`skills/db/db-query-patterns`](../skills/db/db-query-patterns/SKILL.md).
- **Row Level Security (RLS)**: see [`skills/db/rls-policies`](../skills/db/rls-policies/SKILL.md).
- **Server actions / mutations auth pattern**: see [`skills/code/server-actions-auth`](../skills/code/server-actions-auth/SKILL.md).
- **Repository / data-access boundary**: see [`skills/code/data-access-patterns`](../skills/code/data-access-patterns/SKILL.md).
- **Data fetching error handling (`FetchErrorBanner`, retry pattern)**: see [`skills/paths/fetch-error-retry`](../skills/paths/fetch-error-retry/SKILL.md).

### Established patterns

- **List pages (`useEntityList` hook)**: see [`skills/paths/entity-list-hook`](../skills/paths/entity-list-hook/SKILL.md).
- **Detail pages (shell, tabs, not-found, error banner)**: see [`skills/paths/detail-page-with-tabs`](../skills/paths/detail-page-with-tabs/SKILL.md).
- **Route centralization and type config registry**: see [`skills/code/route-and-type-patterns`](../skills/code/route-and-type-patterns/SKILL.md).
- **Theme hook (`useTheme`, single source of truth)**: see [`skills/code/theme-hook`](../skills/code/theme-hook/SKILL.md).

### Styling

- **Design tokens, Tailwind theming, CSS variables**: see [`skills/design/design-tokens-and-theming`](../skills/design/design-tokens-and-theming/SKILL.md).

### Security

- **Security headers, CSP, XSS surface**: see [`skills/security/security-review`](../skills/security/security-review/SKILL.md).

### Accessibility

- **Semantic HTML, ARIA, keyboard navigation, color contrast**: see [`skills/design/accessibility-aa`](../skills/design/accessibility-aa/SKILL.md).

### Testing

- **Test pyramid, E2E with Playwright, CI behavior, flakiness discipline**: see [`skills/testing/testing-conventions`](../skills/testing/testing-conventions/SKILL.md).

### Observability and errors

- **Error handling in Next.js (route handlers, error boundaries, logging)**: see [`skills/code/error-handling-nextjs`](../skills/code/error-handling-nextjs/SKILL.md).

### Code review

- **PR checklist (definition of done)**: see [`skills/code/code-review-checklist`](../skills/code/code-review-checklist/SKILL.md).

---

## 18. Quick Decision Rules

- One feature only → keep local in `src/features/<feature>`.
- Reused across features → `components/shared` or `lib`.
- UI primitives only → `components/ui`.
- Mutations with side effects → server action / handler / controller + validation + authz.
- Unsure between server/client → choose server first.
