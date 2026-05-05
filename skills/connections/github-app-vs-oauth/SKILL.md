---
name: GitHub App vs OAuth App
description: Use GitHub App (not OAuth App) for repo access — bot identity, per-repo scoping, short-lived tokens, org-wide install, and Marketplace eligibility. OAuth App is for Clerk social login only.
category: connections
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Decision rationale for GitHub integration model. Source: `bytetalent/docs/guide-arch.md` §5.

## The decision

Use the **GitHub App** model for repo access. Do not use an OAuth App for repository operations.

OAuth App registered for Clerk social login (sign-in with GitHub) is a separate, unrelated concern — that one stays as OAuth.

## Why GitHub App

| Capability | GitHub App | OAuth App |
|---|---|---|
| Bot identity for PR/commit comments | Yes — `github-actions[bot]`-style identity | No — acts as the user who authorized |
| Per-repo scoping at install time | Yes — installer picks exact repos | No — all-or-nothing per user |
| Token lifetime | Short-lived (1 hour), minted on demand from platform private key | Long-lived user token |
| Org-wide adoption | One install covers all org repos | Each user must authorize individually |
| Webhook lifecycle | Native `installation.created/deleted` events | Manual subscription management |
| GitHub Marketplace | Apps only | Not eligible |

## Implementation details

- `GITHUB_APP_PRIVATE_KEY` is a **platform secret** stored in Vault, not per-user.
- Short-lived installation tokens are minted on demand via `POST /app/installations/{id}/access_tokens` using the JWT-signed private key.
- The `connections` row stores `installationId` only — no tokens persisted (variant: `app_install`).
- Token minting happens inside `mintOutboundToken()` in the GitHub provider adapter.

```ts
// Good — variant is app_install; token minted on demand
const adapter: ProviderAdapter = {
  id: "github",
  variant: "app_install",
  async mintOutboundToken(row) {
    const jwt = signAppJwt(process.env.GITHUB_APP_PRIVATE_KEY!);
    const { token } = await fetchInstallationToken(row.installationId, jwt);
    return token; // 1-hour token; never persisted
  },
};

// Bad — storing a long-lived OAuth token for repo access
const adapter: ProviderAdapter = {
  id: "github",
  variant: "oauth",  // wrong for repo operations
  // ...
};
```

## Anti-patterns

- Registering a GitHub OAuth App for repository reads/writes — loses bot identity, per-repo scoping, and Marketplace path.
- Persisting the installation token in Vault — it expires in 1 hour anyway; always mint fresh.
- Using the same GitHub App registration for both social login and repo access — keep them separate.

## See also

- [`skills/connections/connections-framework`](../connections-framework/SKILL.md) — `app_install` variant, `withSecrets()` boundary, unified status vocabulary
- `bytetalent/docs/guide-arch.md` §5 — source section
