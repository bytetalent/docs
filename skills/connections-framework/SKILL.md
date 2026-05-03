---
name: Connections Framework
description: Two-level credential model (connections + project_connections), three provider variants (app_install/oauth/api_key), Vault-only secret storage, withSecrets() boundary, unified status vocabulary.
category: connections
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

External-service connections are the most sensitive integration surface. This skill covers the unified credential/config model, provider variants, Vault storage rules, and status vocabulary. Per `bytetalent/docs/guide-arch.md` Connections section and `bytetalent/docs/guide-connections.md`.

## Two-level model: credential vs. config

Secrets are stored once at the account level; project-specific targets are stored separately with no secrets:

```
connections           ← Vault-encrypted credential (stored once, reused across projects)
  accountId           ← always set (whose account)
  clientId            ← nullable: null = consultant-owned, non-null = client-owned
  provider            ← "github" | "anthropic" | "slack" | "resend" | ...
  vaultSecretId       ← pgsodium-encrypted token/key

project_connections   ← target config per project (no secrets)
  projectId
  connectionId        ← FK → connections
  config JSONB        ← { repo: "acme/web" } | { channel: "#deploys" } | { model: "claude-opus-4-7" }
```

**Key invariants:**

- `clientId = null` → consultant-owned credential (e.g. consultant's Anthropic key, GitHub App install)
- `clientId = <id>` → client-owned credential (e.g. client's GitHub org, client's Slack workspace)
- Multiple projects share one `connection` row — the client authorises GitHub once; all their projects use it
- Platform-level secrets (GitHub App private key, platform Anthropic key, Stripe, Clerk) stay in Vercel env vars — Vault is for per-user/per-account secrets only

## Three provider variants

Every provider is exactly one variant. The UI, proxy, and status poller branch on this field — never on provider name.

| Variant       | Providers                                                   | Stored                                               |
| ------------- | ----------------------------------------------------------- | ---------------------------------------------------- |
| `app_install` | GitHub (repos/PRs), Slack (bot)                             | `installationId` only — tokens minted on demand      |
| `oauth`       | Linear, Notion, Figma, Vercel, Google, Microsoft, Atlassian | `accessToken`, `refreshToken`, `expiresAt`, `scopes` |
| `api_key`     | Anthropic, OpenAI, xAI, Twilio, SMTP, Telegram              | `apiKey` (+ `apiSecret` / `botToken` where needed)   |

**GitHub as GitHub App (not OAuth App):** Bot identity, per-repo scoping, 1-hour short-lived tokens minted from a platform private key, org-wide install, native webhook lifecycle, Marketplace-ready. OAuth App registered for Clerk social login is a separate concern.

## Vault / withSecrets() boundary

Vault columns are **never** selected by default in Drizzle queries. The only path that retrieves decrypted values is `withSecrets()` in `src/lib/connections/secrets.ts`, importable only from the proxy layer and provider adapters.

- List pages, detail reads, audit logs → get the row with secret columns omitted
- `withSecrets()` → decrypted value, available only in proxy + adapter context
- Revocation order: disable webhooks first → rotate/delete API keys → uninstall GitHub App

```ts
// Good: proxy layer decrypts via withSecrets()
import { withSecrets } from "@/lib/connections/secrets";
const { apiKey } = await withSecrets(connectionId);

// Bad: never select Vault columns outside secrets.ts
const conn = await db.query.connections.findFirst({
  where: eq(connections.id, id),
  columns: { vaultSecretId: true }, // never expose raw vault id in a list route
});
```

## Unified status vocabulary

Use exactly these five values — no synonyms:

| Status         | Meaning                            | Badge |
| -------------- | ---------------------------------- | ----- |
| `connected`    | Healthy; last test/use succeeded   | green |
| `expired`      | Token expired; auto-refresh failed | amber |
| `revoked`      | Upstream uninstall/revoke detected | red   |
| `error`        | Test or last outbound call failed  | red   |
| `disconnected` | User paused                        | grey  |

The execution engine reads status before every step — a non-`connected` connection fails fast with `reconnect_required` and a deep-link back to the detail page.

## Provider adapter interface

Every provider exports a uniform `ProviderAdapter` from `src/lib/connections/providers/<slug>.ts`. The UI, proxy, and status poller all consume this interface — no per-provider branching at call sites.

Required methods: `startConnect`, `test`, `mintOutboundToken`. Optional per variant: `handleCallback` (oauth/app_install), `saveApiKey` (api_key), `refresh` (oauth), `revoke` (all).

## Stripe credential handling

- Use `sk_test_...` for staging and `sk_live_...` for production only
- Client-level connections are scoped per Clerk application (not per Clerk account)
