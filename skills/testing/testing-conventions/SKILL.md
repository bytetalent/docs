---
name: Testing Conventions
description: Unit/integration/E2E pyramid, Playwright project structure, Clerk test OTP, isolation rules, CI cadence, selector discipline, flakiness policy.
category: testing
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Conventions for the three-layer test pyramid and Playwright E2E structure. Per `bytetalent/docs/guide-testing.md`.

## Test pyramid

| Layer       | What to test                                                                                             | Co-located?                           |
| ----------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Unit        | Pure logic: utilities, validators, route builders, type config registries, mappers, billing calculations | Yes — `*.test.ts` next to source file |
| Integration | Repository methods hitting a real DB; server actions performing writes; RLS policy enforcement           | Yes or `src/__tests__/`               |
| E2E         | Sign-in/sign-up/sign-out; forgot-password/reset; protected route access; core CRUD for primary entities  | `e2e/` at project root                |

Never write unit tests for pure presentational components (Button, Badge) unless they own non-trivial logic. Never write tests for static marketing pages.

## Playwright project structure

```
e2e/
  auth/            # auth setup (Clerk test account creation)
  helpers/         # shared page-object helpers, testEmail(), etc.
  prompts/         # prompt fixtures for AI-call stubs
  test.config.ts   # projects: setup, no-auth, full-flow, authenticated
```

Four test projects in `playwright.config.ts`:

| Project         | Purpose                                                   |
| --------------- | --------------------------------------------------------- |
| `setup`         | Create the shared Clerk test account once                 |
| `no-auth`       | Routes that must return 401/redirect when unauthenticated |
| `full-flow`     | Creates its own user per test; self-contained flows       |
| `authenticated` | Reuses setup account for faster single-feature tests      |

## Clerk test mode

- Test mode accepts OTP `424242` for any email ending in `+clerk_test@<domain>`.
- Generate unique addresses with the `testEmail()` helper in `e2e/helpers/`.
- `full-flow` tests create their own user; `authenticated` tests reuse the setup account.
- Do **not** mutate irreversible state (deletes, billing changes) from the `authenticated` project — those go in `full-flow`.

```typescript
// Good — unique address per test run
const email = testEmail("signup"); // e.g., signup+clerk_test@example.com

// Bad — shared address causes test collisions
const email = "test@example.com";
```

## Selector discipline

- Prefer role-based selectors: `getByRole`, `getByLabel`, `getByText` for user-visible strings.
- Avoid CSS class selectors (`locator('.comp-flow-card')`) — they break on refactors and don't reflect user-visible behavior.
- Test names use plain English describing user-visible behavior, not implementation.

```typescript
// Good
await page.getByRole("button", { name: "Create Flow" }).click();

// Bad — brittle CSS class selector
await page.locator(".comp-flow-create-btn").click();
```

## CI cadence

The CI pipeline runs these steps on **every PR**:

1. Lint (`eslint`)
2. Type-check (`tsc --noEmit`)
3. Format check (`prettier --check`)
4. Unit tests (`vitest run`)
5. Build (`next build`)

E2E tests run on a **separate cadence** — a schedule or immediately before a release. They are not required to pass on every PR. This is intentional: E2E suites are slower and environment-dependent.

## Flakiness policy

- Timing-based flakes: fix with better waits (`waitForResponse`, `waitForURL`, `expect.poll`).
- Race-condition flakes: fix the underlying bug causing the race.
- Never add `retries` in `playwright.config.ts` to mask a flaky test — retries hide real bugs.
- A flaky test must be fixed before the PR is merged. Mark it `test.skip` with a linked issue if a fix requires more investigation.

```typescript
// Good — wait for navigation, not an arbitrary timeout
await page.getByRole("button", { name: "Save" }).click();
await page.waitForURL(/\/flows\/[a-z0-9-]+/);

// Bad — arbitrary sleep masks a real timing issue
await page.click("button");
await page.waitForTimeout(2000);
```
