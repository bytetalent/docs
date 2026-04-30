# Connections — What Each One Does, How to Revoke

Plain-language reference for every provider connection the Bytetalent pipeline asks for. Written for consultants explaining things to clients, and clients deciding what access to grant.

The pipeline app renders this doc inline during the onboarding wizard, so the same content lives in two places: in front of you here, and in front of the user when they're connecting a provider.

---

## How connections work

Bytetalent stores credentials at two levels:

- **Account-level** (consultancy-owned) — the consultant's own keys. Used for the consultancy's own projects, or as a BYOK (bring-your-own-key) for a client engagement.
- **Client-level** — the client's own keys. Used when the project should bill against the client's own infrastructure (their Vercel team, their Supabase org, their Stripe account).

Per-user secrets live encrypted in Supabase Vault (`pgsodium`). Decryption is a SQL function callable only under RLS — application code never sees raw ciphertext.

Platform-level secrets (the Bytetalent GitHub App private key, the platform fallback Anthropic key, Stripe webhook signing secret) live in Vercel environment variables and don't appear in the Vault.

See [`guide-arch.md`](guide-arch.md) §5 for the full connections-framework architecture.

---

## Connection inventory

| Provider | Level | Required for | Permissions/scopes |
|---|---|---|---|
| [Anthropic](#anthropic) | Account | Every AI call (or platform fallback) | API access |
| [GitHub App](#github-app) | Account or Client | Code-gen — creating client repos + opening PRs | `contents:write`, `pull_requests:write`, `metadata:read` |
| [Vercel](#vercel) | Account or Client | Provisioning Next.js webapp deploys | `read:project`, `write:project`, `read:env`, `write:env`, `read:domain`, `write:domain` |
| [Cloudflare](#cloudflare) | Account or Client | Provisioning Astro marketing-site deploys | Pages: edit + read; DNS: edit (if domain on Cloudflare) |
| [Supabase](#supabase) | Account or Client | Database + storage + auth backbone | Project create + manage; secret reads |
| [Clerk](#clerk) | Client (per project) | User authentication for the generated client app | Application admin (config + webhooks) |
| [Stripe](#stripe) | Client (per project, if billing is enabled) | Subscription billing in the generated client app | Read/write products + prices, webhooks |
| [Resend](#resend) | Account or Client | Transactional email | Domain management, API key issuance |
| [Upstash Redis](#upstash-redis) | Client (per project) | Rate limiting + caching in the generated client app | Database create + manage |
| [Sentry](#sentry) | Client (per project) | Error tracking in the generated client app | Project create + DSN read |

---

## Anthropic

**What it is:** the LLM provider Bytetalent uses for every AI call — PRD generation, brand kit assembly, design prompts, architecture review, code generation.

**What we use it for:**

- All AI-generated deliverables in your project's pipeline phases
- The agent that writes the code modules in your generated client repo

**Permissions/scopes:** the API key gives Bytetalent the ability to spend Anthropic credits on calls made on behalf of your project.

**How to set it up:**

1. Sign in at [console.anthropic.com](https://console.anthropic.com)
2. **API Keys** → **Create Key** — name it `bytetalent-pipeline-{your-name}`
3. Copy the key (`sk-ant-...`)
4. Paste into the pipeline's connection form

**How to revoke:** at [console.anthropic.com](https://console.anthropic.com) → API Keys → click the key → **Delete**. Takes effect immediately.

**What breaks if you revoke:**

- All AI phases will fail until you connect a new key (or the platform falls back to Bytetalent's key, depending on configuration)
- In-flight generations will error and surface a retry notification

**BYOK vs. platform key:** Bytetalent provides a platform Anthropic key as fallback so projects can run without BYOK during evaluation. Production engagements should connect their own — costs are visible in the Anthropic dashboard rather than rolled into Bytetalent's invoice.

---

## GitHub App

**What it is:** the **Bytetalent GitHub App** (different from a personal access token, different from an OAuth app). Installs onto your GitHub organization to scope access per-repo.

**What we use it for:**

- Creating the generated client repo (one per project)
- Pushing initial scaffold + opening per-module PRs
- Posting PR comments + status checks

**Permissions/scopes** (declared by the app):

- `contents:write` — read/write repo files
- `pull_requests:write` — open/comment/merge PRs
- `metadata:read` — basic repo info

**Repository selection:** at install time, you choose **All repositories** or **Only select repositories**. Pick the latter and grant access only to repos under generation. Bytetalent never asks for blanket access.

**How to set it up:**

1. From the pipeline's onboarding wizard, click **Install Bytetalent GitHub App**
2. GitHub redirects you to the app's install page
3. Choose your organization (or personal account)
4. Select repos: **All repositories** or **Only select repositories**
5. Approve

**How to revoke:** at [github.com/settings/installations](https://github.com/settings/installations) (or your org's `/settings/installations`) → find **Bytetalent** → **Configure** → **Uninstall**. Takes effect immediately; in-flight code-gen will fail with a clear error.

**What breaks if you revoke:**

- Code-gen for new and in-flight projects fails until you reinstall
- Existing repos generated previously stay where they are — they're owned by you, not Bytetalent
- PR comments from the bot stop being possible (the bot loses write access)

**Why GitHub App, not OAuth:** apps post comments under the Bytetalent identity (not your user account), use short-lived 1-hour tokens (not persistent ones), and revoke cleanly via uninstall. See [`guide-arch.md`](guide-arch.md) §5.

---

## Vercel

**What it is:** the host for Next.js projects. Bytetalent provisions a Vercel project per pipeline project, links it to the GitHub repo, sets env vars, and triggers the first deploy.

**What we use it for:**

- Creating the Vercel project
- Linking it to the generated GitHub repo
- Setting environment variables (Clerk + Supabase + Stripe + Sentry + Upstash keys)
- Attaching a custom domain
- Triggering deploys

**Permissions/scopes** (Vercel API token):

- `read:project`, `write:project`
- `read:env`, `write:env`
- `read:domain`, `write:domain`

**How to set it up:**

1. Sign in at [vercel.com](https://vercel.com)
2. **Account Settings** → **Tokens** → **Create Token**
3. Scope: full account access (Vercel doesn't yet support fine-grained API tokens — track [their issue tracker](https://vercel.com/changelog) for updates)
4. Name: `bytetalent-{project-name}`; expiration: as needed
5. Paste into the pipeline's connection form

**How to revoke:** at [vercel.com](https://vercel.com) → Account Settings → Tokens → click the token → **Delete**. Takes effect immediately.

**What breaks if you revoke:**

- Provisioning of new Vercel projects from the pipeline fails
- Existing deploys continue running (they don't depend on the API token)
- Bytetalent can no longer push env-var updates or trigger deploys via the API; you'd push them manually via the dashboard

---

## Cloudflare

**What it is:** the host for Astro marketing sites. Bytetalent provisions a Cloudflare Pages project per marketing engagement and configures DNS where applicable.

**What we use it for:**

- Creating the Cloudflare Pages project
- Setting environment variables + secrets for the Resend contact form
- Attaching a custom domain
- (Optional) Configuring DNS records if your domain is on Cloudflare

**Permissions/scopes** (API token):

- **Pages**: Edit + Read
- **DNS** (only if your domain is on Cloudflare): Edit

**How to set it up:**

1. Sign in at [dash.cloudflare.com](https://dash.cloudflare.com)
2. **My Profile** → **API Tokens** → **Create Token**
3. Use the **Custom token** template
4. Set permissions: `Account → Pages: Edit`, optionally `Zone → DNS: Edit`
5. Account resources: include all (or scope to a specific account)
6. Zone resources (if including DNS): scope to specific zones you manage
7. Paste into the pipeline's connection form

**How to revoke:** [dash.cloudflare.com](https://dash.cloudflare.com) → My Profile → API Tokens → click the token → **Roll** (rotate) or **Delete**. Takes effect immediately.

**What breaks if you revoke:**

- Provisioning of new Pages projects fails
- Existing deploys continue running
- DNS updates would need manual dashboard work

---

## Supabase

**What it is:** the database + storage + realtime backbone for every Bytetalent webapp.

**What we use it for:**

- Creating the per-project Supabase project
- Applying Drizzle migrations + RLS policies
- Creating storage buckets
- Pulling project keys (URL, anon, service-role) into Vercel env vars

**Permissions/scopes** (Personal Access Token):

- Full account access (Supabase doesn't yet support fine-grained tokens for the Management API — track [their changelog](https://supabase.com/changelog))

**How to set it up:**

1. Sign in at [supabase.com](https://supabase.com)
2. **Account Settings** → **Access Tokens** → **Generate New Token**
3. Name: `bytetalent-{project-name}`
4. Paste into the pipeline's connection form

**How to revoke:** [supabase.com](https://supabase.com) → Account Settings → Access Tokens → click the token → **Delete**. Takes effect immediately.

**What breaks if you revoke:**

- Provisioning of new Supabase projects fails
- Migrations + RLS policy updates would need manual application via the SQL editor or CLI
- Existing project data stays put; the deployed app continues running with its anon + service-role keys (those are project-scoped, not token-scoped)

---

## Clerk

**What it is:** the auth provider for generated client apps. Handles sign-in, sign-up, password reset, OTP, OAuth providers, and session management.

**What we use it for:**

- Creating the per-project Clerk application (manual step — Clerk doesn't auto-create apps via API)
- Configuring webhooks (so user events flow into your client app's database)
- Configuring OAuth providers (Google, GitHub) when you opt in
- Pulling Clerk publishable + secret keys into Vercel env vars

**Permissions/scopes:** Clerk Backend API key with full app admin access. Scoped to a single Clerk application — not your entire Clerk account.

**How to set it up:**

1. Sign in at [clerk.com](https://clerk.com)
2. **Create Application** — manual; Clerk doesn't expose application creation via API
3. In the new app: **API Keys** → copy the publishable + secret keys
4. Paste both into the pipeline's connection form
5. The pipeline configures webhooks + OAuth providers via the Backend API

**How to revoke:**

- For one project: in the Clerk dashboard for that application → **API Keys** → rotate the secret key. The pipeline can no longer manage webhooks/config until you provide the new key.
- For all projects: deactivate the application entirely.

**What breaks if you revoke:**

- The pipeline can't update the application's webhook config or OAuth providers
- The deployed client app continues serving auth — it uses the publishable + secret keys directly from Vercel env vars; rotating the API key in Clerk also requires updating the env var in Vercel for the auth to keep working
- User sessions in the deployed app are unaffected by API key rotation

---

## Stripe

**What it is:** the payment processor used by the optional `stripe-subscriptions` template.

**What we use it for:**

- Creating products + prices that match your pricing tiers
- Configuring the webhook endpoint (so subscription events flow into your client app's database)
- Configuring the customer portal (where users manage their own subscription)

**Permissions/scopes** (restricted API key):

- Read/write: Products, Prices, Webhook Endpoints, Customer Portal Configuration
- Write: Customers (read-only is insufficient — the app creates customer records on signup)

**How to set it up:**

1. Sign in at [dashboard.stripe.com](https://dashboard.stripe.com)
2. **Developers** → **API keys** → **Restricted keys** → **+ Create restricted key**
3. Name: `bytetalent-{project-name}`
4. Set the permissions listed above
5. Paste into the pipeline's connection form

**How to revoke:** [dashboard.stripe.com](https://dashboard.stripe.com) → Developers → API keys → click the key → **Revoke**. Takes effect immediately.

**What breaks if you revoke:**

- Provisioning of new products + webhooks fails
- The deployed client app continues to function for users with active subscriptions (it uses its own publishable + secret keys from Vercel env vars; rotating those requires updating the env var)
- New subscription creation in the app stops working until you provide a new key

**Test mode vs. live mode:** Bytetalent's pipeline supports both. Use test-mode keys (`sk_test_...`) for staging projects; live-mode keys (`sk_live_...`) only for production engagements.

---

## Resend

**What it is:** the transactional email provider. Used by the Astro base's contact form, the optional `notifications-center` template, and any future email-driven feature.

**What we use it for:**

- Creating an API key (account-level)
- Registering + verifying sending domains (per project)
- Setting the API key as a Vercel/Cloudflare env var so the deployed app can send mail

**Permissions/scopes** (API key):

- `domains:write` (create + verify sending domains)
- `emails:send` (send transactional mail)
- `audiences:write` (only if using Resend's audience feature; optional)

**How to set it up:**

1. Sign in at [resend.com](https://resend.com)
2. **API Keys** → **Create API Key**
3. Permission: **Sending access** (covers domains + emails:send)
4. Domain: scope to the specific domain you'll send from
5. Paste into the pipeline's connection form

**How to revoke:** [resend.com](https://resend.com) → API Keys → click the key → **Delete**. Takes effect immediately.

**What breaks if you revoke:**

- Domain verification + new API key issuance via the pipeline fails
- The deployed client app's contact form / transactional emails fail until you rotate the env var on Vercel/Cloudflare with a new key

---

## Upstash Redis

**What it is:** the Redis store used for rate limiting + caching in the generated client app. Per [`guide-arch.md`](guide-arch.md) §7, Bytetalent picks Upstash directly over Vercel KV (which wraps Upstash anyway).

**What we use it for:**

- Creating the per-project Redis database
- Pulling REST URL + token into Vercel env vars

**Permissions/scopes** (Upstash API key):

- Full account access (Upstash doesn't yet support fine-grained API tokens — track their roadmap)

**How to set it up:**

1. Sign in at [console.upstash.com](https://console.upstash.com)
2. **Account** → **Management API** → **Create API Key**
3. Name: `bytetalent-{project-name}`
4. Paste into the pipeline's connection form

**How to revoke:** [console.upstash.com](https://console.upstash.com) → Management API → click the key → **Delete**. Takes effect immediately.

**What breaks if you revoke:**

- Provisioning of new Redis databases fails
- Existing databases continue serving rate-limit + cache reads to your deployed apps (they use REST URL + token from Vercel env vars; rotating those at Upstash requires updating env vars)

---

## Sentry

**What it is:** error tracking + performance monitoring for the deployed client app.

**What we use it for:**

- Creating the per-project Sentry project
- Pulling DSN into Vercel env vars
- Uploading source maps at build time (via `SENTRY_AUTH_TOKEN`)

**Permissions/scopes** (Internal Integration token):

- `project:write` (create projects, manage releases)
- `team:read` (resolve team scoping)

**How to set it up:**

1. Sign in at [sentry.io](https://sentry.io)
2. **Settings** → **Custom Integrations** → **Create New Integration**
3. Name: `Bytetalent Pipeline`
4. Permissions: as listed above
5. Save → copy the auth token
6. Paste into the pipeline's connection form

**How to revoke:** [sentry.io](https://sentry.io) → Settings → Custom Integrations → click the integration → **Disable**. Takes effect immediately.

**What breaks if you revoke:**

- Provisioning of new Sentry projects fails
- Existing projects continue receiving events (the DSN is project-scoped, not auth-token-scoped)
- New release tracking + source-map upload at build time fails (deploys still succeed; you just lose unminified stack traces in Sentry)

---

## What if I want to revoke everything?

In order, from least disruptive to most:

1. **Disable webhooks first** — at each provider's dashboard, disable any webhook endpoints pointing at Bytetalent. Stops the pipeline from receiving lifecycle events.
2. **Rotate or delete API keys/tokens** — for each provider above. Stops the pipeline from making outbound calls on your behalf.
3. **Uninstall the Bytetalent GitHub App** — at github.com → Settings → Installations. Removes repo access entirely.
4. **Notify your consultant** — they'll see Connection-revoked badges in the pipeline app for all the providers you've disconnected.

Generated client repos and deployed apps are unaffected by step 1–3 — they own their own keys and run on their own infrastructure. The pipeline just loses the ability to make further updates on your behalf.

---

## Where credentials are stored after you connect

- **Per-user credentials** (your Anthropic key, your Vercel token, etc.) → encrypted at rest in Supabase Vault. Decryption only happens server-side, only via `withSecrets()`, only during a pipeline action that needs the credential.
- **Platform credentials** (Bytetalent's GitHub App private key, the platform fallback Anthropic key) → Vercel environment variables. Not in Vault. Bytetalent rotates these periodically.

See [`guide-arch.md`](guide-arch.md) §5 for the full secret-storage architecture.

---

## Related

- [`guide-arch.md`](guide-arch.md) — connections framework, vault, three-tier credential resolution
- [`guide-deploy.md`](guide-deploy.md) — environment-variable management on deploy targets
- [`guide-pipeline.md`](guide-pipeline.md) — end-to-end picture of how connections feed into provisioning + code-gen
