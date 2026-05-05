# Code Review Checklist

Cross-stack code review categories and severity rubric. When performing a review, work through these systematically.

For each finding, report: **severity** (Critical / High / Medium / Low), **file path with line number**, **what's wrong**, and **how to fix it**.

---

## 1. Security

- **Server actions & API routes**: Are all user inputs validated (e.g., Zod schemas) before processing? Are there any raw string interpolations in SQL, shell commands, or redirects that could enable injection?
- **Authentication & authorization**: Are all protected routes covered by middleware? Are there client-side guards that duplicate or replace server-side checks (redundant SessionGuard patterns)?
- **Secrets & credentials**: Are `.env` files, API keys, tokens, or connection strings committed or hardcoded? Are sensitive values exposed to the client bundle via missing `NEXT_PUBLIC_` (or stack-equivalent) discipline?
- **XSS surface**: Is `dangerouslySetInnerHTML` used anywhere? Are user-supplied values rendered without sanitization? Are href attributes built from user input without validation?
- **CSRF & headers**: Are server actions and API routes protected against CSRF? Are security headers configured (CSP, X-Frame-Options, etc.)?
- **Dependencies**: Are there outdated packages with known CVEs? Are lock files committed and consistent?

## 2. Maintainability

- **File size**: Flag any page or component exceeding ~150 lines (or ~300 for complex page-clients with extracted sub-components). Identify extractable sub-components, custom hooks, or utility functions that would bring it under threshold.
- **Naming conventions**: Verify all files follow project conventions (e.g., `comp-` prefix for React components, kebab-case for hooks, PascalCase for exported components). Flag any deviations.
- **Route centralization**: Are route strings hardcoded across the codebase, or centralized in a `ROUTES` constant object / `routes.ts`? Flag every instance of a hardcoded route path.
- **Dead code**: Identify unused imports, unexported functions, components with zero consumers, files that are never imported.
- **Hook hygiene**: Are custom hooks following the `use-` prefix convention? Are there hooks with misleading names (e.g., legacy naming that doesn't match current domain terminology)?
- **Type safety**: Are there `any` types, type assertions (`as`), or `@ts-ignore` / `@ts-expect-error` comments? Are shared interfaces/types co-located or centralized appropriately?
- **Prop drilling depth**: Flag components receiving >5 props that could be restructured with composition, context, or extracted sub-components.
- **Component consolidation**: Identify components with overlapping responsibility, near-identical structure, or shared state logic that would be better as a single parameterized component. For each candidate group, suggest: which component to keep as the base, what props or slots to add, and which files to delete. Examples: two detail shells with the same header/edit-mode pattern, multiple stat cards with different labels but identical layout, list pages that share filter/pagination/bulk-action behavior.
- **Error boundaries**: Are there `error.tsx` and `loading.tsx` files at appropriate route segments (or stack-equivalent)? Is there a root `not-found.tsx`?

## 3. Performance

- **Client vs. server components**: Are `"use client"` directives applied only where necessary (event handlers, hooks, browser APIs)? Flag components marked client-side that use no client features.
- **Bundle size**: Are heavy libraries imported where lighter alternatives exist? Are barrel imports pulling in unused modules?
- **Re-render risk**: Are inline object/array/function literals passed as props (causing unnecessary re-renders)? Are expensive computations memoized where appropriate?
- **Image optimization**: Are images using the framework's `<Image>` (Next.js, Astro) with proper `width`/`height` or `fill`? Are there raw `<img>` tags?
- **Data fetching**: Are there client-side fetches that could be server components or server actions? Are there waterfall request patterns that could be parallelized?
- **Lazy loading**: Are large, below-the-fold components using `dynamic()` / `React.lazy()`?

## 4. Design Consistency

- **Hardcoded colors**: Flag every hex color (`#xxx`), `rgb()`/`rgba()` value, or hardcoded color string. These should use CSS variables (e.g., `var(--color-primary)`) or Tailwind theme tokens (e.g., `text-foreground`, `bg-muted`).
- **Gradient & opacity patterns**: For semi-transparent brand colors, verify use of `color-mix(in srgb, var(--color-primary) N%, transparent)` instead of hardcoded rgba.
- **Spacing & sizing**: Are there inconsistent spacing values (arbitrary `px-[13px]` vs. standard scale `px-3`)? Flag non-standard magic numbers.
- **Typography**: Are text styles using utility classes (e.g., `text-section-heading`, `text-table-header`) vs. ad-hoc combinations?
- **Dark mode**: Does every custom style have a `dark:` variant? Are there components that would break visually in dark mode?
- **Accessibility**: Do images have meaningful `alt` text (or `role="presentation"` for decorative)? Do interactive elements have proper `aria-*` labels? Is keyboard navigation supported? Is color contrast sufficient (see [`guide-design.md`](guide-design.md))?
- **Component reuse & consolidation**: Are there near-duplicate UI patterns that should use a shared component (e.g., status badges, empty states, table skeletons)? Beyond visual duplication, look for components with identical layout but different data — flag these as consolidation candidates and suggest the unified API (props, slots, or render props) that would replace them.

## 5. Code Minimization

- **Repeated logic**: Identify duplicated conditional chains, status-to-style mappings, or config objects that should be extracted into utility functions or lookup tables (e.g., `createTypeRegistry<T>()` pattern).
- **Verbose patterns**: Flag ternary chains >3 branches (replace with lookup objects), repeated className construction (extract to utilities), and copy-pasted JSX blocks.
- **Abstraction opportunities**: Are there generic patterns (list page with filters/pagination/bulk actions) that could use a shared hook (e.g., `useEntityList`) to eliminate boilerplate?
- **Import hygiene**: Are imports organized and minimal? Are there default imports where named imports would be clearer, or vice versa?
- **Prop interfaces**: Are component props minimal? Flag props that pass entire objects when only 1-2 fields are used (prefer individual props for clarity and narrower coupling).

## 6. CI/CD & Tooling

- **Pipeline coverage**: Is there a CI pipeline that runs lint, typecheck, format check, tests, and build on every PR?
- **Formatting**: Is Prettier configured with consistent settings? Is `prettier-plugin-tailwindcss` included for class sorting?
- **Test coverage**: Are pure logic modules (utilities, validators, route builders) covered by unit tests? Are there test files for the most critical business logic?
- **TypeScript strictness**: Is `strict: true` enabled in `tsconfig`? Are there loose compiler options that should be tightened?
- **Version currency audit (required)**: Check all frameworks, supporting software, and package versions against currently available stable releases (Next.js, React, TypeScript, Node.js runtime, Clerk, Tailwind, test tooling, key transitive security-sensitive deps). Report:
  - what's current in the repo
  - latest available stable version
  - upgrade priority (`now`, `next sprint`, `defer`) with reason (security, compatibility, breaking-change risk)
  - concrete upgrade path (direct bump vs. staged migration)

---

## Output Format

Organize findings into a prioritized table grouped by severity. For each finding include: **ID, severity, category, file:line, description, recommended fix**. After the table, provide a suggested implementation order that addresses critical/high items first.

---

## Cadence

- Reviews land as dated files: `docs/app-review-YYYY-MM-DD.md` in the consuming repo.
- Open action items from the latest review get tracked in that repo's plan file.
