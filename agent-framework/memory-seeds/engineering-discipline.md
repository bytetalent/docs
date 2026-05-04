# Engineering discipline

Quality and safety standards. The rules in this doc share a common thread: shortcuts cost more than they save, and the cheapest moment to do something correctly is the first one.

## Read docs end-to-end before integrating an SDK

When integrating an external SDK or service, the workflow MUST be:

1. **Find the official "manual setup" / "installation" / "getting started" page for the EXACT version we're using.** Not v9 docs when we're on v10. Not generic "JavaScript guide" when there's a Next.js-specific one.
2. **Read it end-to-end before writing any code.** Note every required file, every recommended option, every gotcha mentioned for our framework version.
3. **List the deviations between the canonical setup and what's about to be written.** If deviating, justify it explicitly — otherwise match the docs.
4. **THEN** write the code.
5. If the integration doesn't work, **the first hypothesis is "I deviated from the docs," not "there's an SDK bug."** Re-read the docs before debugging.

**Why:** Anthony 2026-04-28 after the Sentry saga: "we just spent a lot of time and tokens, chasing our tail with this one. it sure would be nice to read the docs and do it right rather than firing in every direction."

The Sentry integration cost ~5 PRs and many tokens because:
- Used `sentry.client.config.ts` (older filename) instead of `instrumentation-client.ts` (current).
- Set `integrations: []` (disabling all default integrations including the uncaught-exception handlers — the actual root cause of the silent-drop).
- Missed `onRouterTransitionStart`, `widenClientFileUpload`, `tunnelRoute`, `sendDefaultPii`, `global-error.tsx`.
- Jumped to writing diagnostic probes when stuck instead of doing the doc audit first.

A 30-minute doc audit at the start would have prevented all of it.

**How to apply:**
- Sentry, Stripe, Clerk, Supabase, Anthropic SDK, Vercel SDK, Drizzle, Next.js — all integrations get the doc-read-first treatment.
- For breaking-change-prone vendors (Sentry, Next.js), check changelogs for the version we're on, not just the latest docs.
- When fixing integration bugs, the diff between "what we have" and "what the docs show" is always step 1.
- Use WebFetch to pull the actual current docs page when in doubt — don't rely on training data for vendor specifics.
- If a probe / diagnostic seems necessary, ask: "would re-reading the docs answer this?" first.

## No shortcuts on infrastructure setup or testing

When a PR description lists post-merge manual setup (e.g. "configure Clerk webhook", "add env var", "verify Stripe keys", "register OAuth callback URL"), do those steps **as part of landing the PR**, not later.

Skipping them means the feature is half-built. The gap surfaces weeks later during E2E testing, often blocking the very thing the test is trying to validate. Re-discovering and re-fixing is more expensive than just doing it right when the context is fresh.

**Why:** Anthony 2026-05-04 — testing testy_admin onboarding revealed that PR #445 (R2-5a multi-account memberships) listed "Add Clerk webhook endpoint for /api/webhooks/clerk/organization" as a manual post-merge step. That step never happened. Result: org membership webhooks never fire, account_memberships rows never materialize on Clerk org changes, and testy_admin's super-admin path was silently broken until E2E testing surfaced it.

Anthony's principle: "Path A always, no shortcuts. It's only time and tokens for the knowledge that we always do things right."

**How to apply (infrastructure):**
- When a PR description says "After merge: do X" → don't merge until X is also done OR file a tracked follow-up ticket and complete it before declaring the feature ready.
- Auth + identity infrastructure (Clerk config, org slug, admin user setup, webhook endpoints, env vars) is foundational; it should be set up EARLY and COMPLETELY. Half-configured auth blocks every persona walkthrough downstream.
- For every cross-environment infrastructure change, ask: does this need to be re-applied on staging? prod? — those are usually missed too.
- When dispatching a worker on a feature that needs external setup (Clerk dashboard, third-party admin panels), include explicit instructions in the brief: "after PR lands, document the manual steps OR open a follow-up `[infra]` ticket."
- Maintain a living "deploy/setup runbook" doc capturing all the manual steps that aren't automated (env vars, dashboard configs, webhook registrations, OAuth callbacks). Verify it during every staging promotion.

**This principle applies to TESTING / VALIDATION too.** When walking E2E scenarios, exercise the production-target flow — the realistic path real users will take. Anthony 2026-05-04: "no quick paths ever claude come on, we are not hacks vibe coding our way, we are professional and polished engineers, trying to properly verify every product feature to its completion. enough with the vibe hacker suggestions."

**How to apply (testing/validation):**
- **Don't propose platform-fallback / env-var hacks** to bypass per-account setup ("just set ANTHROPIC_API_KEY at the platform level"). The realistic flow IS the per-account path; that's what real customers do; that's what we test.
- **Don't frame shortcuts as "the quick path"** vs the "realistic path" — that label leads toward the shortcut even when the user has already shown they prefer correctness. If the user is choosing between two real paths, present both neutrally without pushing the easier one.
- **Don't offer to bypass a discovered gap** "for now" — the gap IS the test. Surface it, file the ticket, walk the proper flow even if it costs more time.
- The instinct to suggest "quick path / hack to keep moving" is the wrong instinct when working with someone building toward a production product. Resist it.

## Never read whole .env or secret-bearing files

**Rule:** Do not use the `Read` tool on `.env`, `.env.local`, `.env.prod`, `.env.staging`, or any `.env.*` file. Never display their contents. Same rule applies to any file likely to contain secrets — `*credentials*`, `*secret*`, `*.pem`, `*.key`, `*.p12`, kubeconfigs, AWS credential files, service-account JSONs.

**Why:** On 2026-05-01, reading `.env.local` to debug a multi-line PEM parsing problem put the whole file into the transcript, exposing 8 secrets (GitHub App private key + webhook secret, Clerk test key + webhook secret, Supabase service role key + DB password, Upstash REST token, Sentry auth token). The user was rightfully angry — none of that needed to happen. A check script that printed stats only would have diagnosed the same issue without exposing any value.

**How to apply:**

When debugging env-var issues, write a Node/PowerShell script that:
- Reads the specific variable(s) by name (parse the file manually if dotenv chokes — multi-line-aware reference parser scripts exist, e.g. `scripts/check-github-app-env.mjs` in app-flow-web).
- Logs structure only: length, line count, presence of expected markers (e.g. `BEGIN`/`END` for PEMs), boolean checks like "is numeric," "starts with `pk_test_`," etc.
- Never logs the value itself, never logs the raw file contents.
- Can optionally try to instantiate a real client (Octokit, Clerk SDK, etc.) to confirm runtime correctness — that's a stronger signal than parsing alone.

If a config file might contain secrets and the user asks me to inspect it, propose the check-script approach FIRST. Only `Read` after the user confirms the file is non-sensitive.
