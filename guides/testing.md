# Testing Standards

Cross-stack testing principles. The canonical rules live in [`skills/testing/testing-conventions`](../skills/testing/testing-conventions/SKILL.md). This guide covers the rationale behind the pyramid and when each layer applies.

Stack-specific test runner config (Vitest setup, Playwright config paths, ASP.NET test projects) lives in each base or product's `docs/` overlay. Clerk test-mode OTP details belong in the nextjs-clerk-supabase base overlay.

---

## Why three layers (not just E2E)

Unit tests catch logic regressions cheap, with no infrastructure. Integration tests catch ORM/RLS/migration mistakes that unit tests miss — they hit a real DB so schema changes and RLS policies are exercised. E2E catches the cracks between layers: auth flows, redirect behavior, multi-step UX. Running all three means failures are fast to diagnose — a unit failure isolates the function, an integration failure isolates the data path, an E2E failure isolates the user-visible behavior.

## Why E2E is not on every PR by default

E2E runs against a live dev server and a real auth provider. They're slow, prone to rate-limit failures on test account creation, and require the server to be up. Running them on a schedule or pre-release rather than every PR keeps CI fast without sacrificing coverage at the critical moments.

## Why role-based selectors over CSS/ID selectors

Coupling tests to CSS classes or internal IDs ties them to implementation rather than behavior — a refactor that keeps the UX identical breaks the tests. Role-based selectors (`getByRole`, `getByLabel`) survive component rewrites and match what the user actually sees.

## Why flaky tests are worse than no tests

A flaky test that "usually passes" trains everyone to ignore failures. When it eventually catches a real regression, nobody believes it. Fix flaky tests before merge; if a flake is environmental, fix the test (better wait, idempotent setup) — don't add retries as a workaround.

---

## Canonical rules and patterns

**Test pyramid coverage requirements, E2E project structure, test isolation, helper patterns, CI behavior, naming and selector discipline, troubleshooting, flakiness discipline:** see [`skills/testing/testing-conventions`](../skills/testing/testing-conventions/SKILL.md).
