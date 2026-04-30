# Testing Standards

Cross-stack testing principles. Stack-specific test runner config (Vitest setup, Playwright config paths, ASP.NET test projects) lives in each base or product's `docs/` overlay.

---

## 1. Test Pyramid

| Type | What | Where |
|---|---|---|
| **Unit** | Pure logic — utilities, validators, mappers, route builders | Co-located: `*.test.ts` next to the source, or `__tests__/` folders |
| **Integration** | Data access + server actions + API handlers, with a real DB | `tests/integration/` or `__tests__/integration/` |
| **E2E** | Critical user flows in a real browser against a real dev server | `e2e/` directory |

Unit tests catch logic regressions cheap. Integration tests catch ORM/RLS/migration mistakes. E2E catches the cracks between layers — auth flows, redirect behavior, multi-step UX.

## 2. What Must Be Tested

**Unit-test coverage required for:**
- Pure logic modules — utilities, validators, route builders, type config registries
- Mappers / serializers
- Calculation logic (billing, contrast ratios, sort comparators, etc.)

**Integration tests required for:**
- Repository methods that hit a real DB
- Server actions that perform writes
- RLS policy enforcement (verify a different account can't read another's data)

**E2E tests required for:**
- Sign-in / sign-up / sign-out
- Forgot password / reset
- Protected route access (authenticated vs. unauthenticated)
- Core CRUD paths for primary entities

**No test required for:**
- Pure presentational components (Button, Badge, etc.) — unless they own non-trivial logic
- Static marketing pages (covered by E2E happy-path)

## 3. E2E with Playwright

E2E tests use Playwright. Tests live in `e2e/` and run against a live dev server (port 3000 by default for Next.js, varies for Astro).

### Test project structure

Playwright is typically configured with multiple projects, each covering a different auth state:

```text
e2e/
  auth/
    auth.setup.ts           ← creates one shared test account (runs once)
    change-password.spec.ts ← uses shared auth session
    profile.spec.ts         ← uses shared auth session
    forgot-password.spec.ts ← no auth needed
    guards.spec.ts          ← no auth needed (tests redirect behavior)
    signin.spec.ts          ← no auth needed
    signup.spec.ts          ← no auth needed
  full-flow.spec.ts         ← creates its own test user; tests all dashboard pages
  helpers/                  ← shared auth helpers (signUp, signIn, fillOtp, etc.)
  prompts/
    full-flow.md            ← manual checklist + AI test-writing prompt
  test.config.ts            ← centralized constants (OTP, password, email)
```

| Project | Matches | Auth state | Dependencies |
|---|---|---|---|
| `setup` | `auth.setup.ts` | Creates shared account → saves to `e2e/.auth-state.json` | None |
| `no-auth` | signup/signin/forgot-password/guards specs | No auth | None |
| `full-flow` | `full-flow.spec.ts` | Creates its own test user per run | None |
| `authenticated` | change-password/profile specs | Loads shared session from `e2e/.auth-state.json` | `setup` |

### Clerk test mode + OTP

When using Clerk: test mode accepts OTP `424242` for any email ending in `+clerk_test@<domain>`. No Clerk dashboard configuration needed — automatic in development mode.

```text
test+<unique-id>+clerk_test@<consultancy-domain>
```

Helper `testEmail()` generates a unique email per call.

**OTP value:** `424242` (constant `TEST_OTP`).
**Password:** any strong test value (constant `TEST_PASSWORD`).

For other auth providers, use the equivalent test-mode mechanism.

### Test isolation

- Tests in `full-flow.spec.ts` create their own user via `testEmail()` — no shared state between tests.
- `authenticated` project tests share the user from `auth.setup.ts` — don't mutate irreversible account state in these tests.

### Helpers

```typescript
import { signUp, signIn, fillOtp } from "../helpers/auth";
import { testEmail, TEST_PASSWORD } from "../test.config";

test("example", async ({ page }) => {
  const email = testEmail();
  await signUp(page, email);
  await page.goto("/space/flows");
  await expect(page.getByRole("heading", { name: "Flows" })).toBeVisible();
});
```

## 4. CI Behavior

- E2E tests are **not run on every PR** by default — CI runs lint, typecheck, format check, unit tests, and build only. E2E runs on a schedule or pre-release.
- When E2E is in CI:
  - `retries: 2` is automatically enabled when `process.env.CI` is set
  - Screenshots and traces captured on failure (`screenshot: "only-on-failure"`, `trace: "on-first-retry"`)
  - Workers forced to 1 (`fullyParallel: false`) to avoid auth-provider rate limiting on test account creation

## 5. Writing New Tests

### Reference the manual checklist

`e2e/prompts/full-flow.md` is a comprehensive manual test checklist covering every page and user flow. Use it as reference when writing new specs or as a prompt for AI-assisted test generation.

### Naming

Spec files: `<feature>.spec.ts`. Test names use plain English describing user-visible behavior, not implementation:

```typescript
// ✅ Good — describes user behavior
test("user can sign up with email and verify via OTP", async ({ page }) => { ... });

// ❌ Bad — describes implementation
test("renders SignupForm with submit button", async ({ page }) => { ... });
```

### Selector discipline

Prefer accessible role-based selectors over implementation-coupled selectors:

```typescript
// ✅ Good — what the user sees
await page.getByRole("button", { name: "Sign in" }).click();
await page.getByLabel("Email").fill("test@example.com");

// ❌ Bad — coupled to implementation
await page.locator(".btn-primary").click();
await page.locator("#email-input").fill("test@example.com");
```

## 6. Troubleshooting

- **`net::ERR_CONNECTION_REFUSED`** → Dev server isn't running. Start it first or let Playwright start via `webServer` config.
- **OTP step times out** → Verify auth provider is in development/test mode.
- **`storageState: e2e/.auth-state.json` not found** → Run the `setup` project first.
- **Tests fail with provider 429 rate limit** → Too many sign-up attempts. Wait or reduce parallel workers.
- **`forbidOnly` error in CI** → A `test.only()` was committed. Remove all `.only` calls before merging.

## 7. Flakiness Discipline

- **Fix flaky tests before merge.** A flaky test that "usually passes" is worse than no test — it trains everyone to ignore failures.
- If a flake is environmental (timing, network), fix the test (better wait, idempotent setup) — don't add retries.
- If a flake is real (race condition, test pollution), fix the underlying bug.
