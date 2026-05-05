# Connections

How external-service connections work in Bytetalent projects — the two-level model, Vault storage, three-tier key resolution, and provider adapter interface.

For the full framework, see [`skills/connections/connections-framework`](../skills/connections/connections-framework/SKILL.md).

---

## How connections work

Credentials are stored at two levels:

- **Account-level** (consultancy-owned) — the consultant's own keys. Used for the consultancy's own projects, or as BYOK for a client engagement.
- **Client-level** — the client's own keys. Used when the project should bill against the client's own infrastructure.

Per-user secrets are encrypted in Supabase Vault (`pgsodium`). Platform-level secrets (GitHub App private key, platform fallback Anthropic key) live in host env vars and don't appear in the Vault.

The full two-level schema, three-tier credential resolution ladder, provider adapter interface, and secret-storage rules are in the `connections/connections-framework` skill above.

---

## Provider setup and revocation (app-flow.ai)

The concrete setup steps, required permissions, and revocation instructions for each provider (Anthropic, GitHub App, Vercel, Cloudflare, Supabase, Clerk, Stripe, Resend, Upstash Redis, Sentry) are product-specific to the app-flow.ai pipeline.

See `bt-ai-web/docs/ops/connections-reference.md` in the [bytetalent/bt-ai-web](https://github.com/bytetalent/bt-ai-web) repo.

---

## Related

- [`skills/connections/connections-framework`](../skills/connections/connections-framework/SKILL.md) — two-level model, adapter interface, Vault storage, status vocabulary
- [`guide-arch.md`](arch.md) — standardized access paths, proxy egress
- [`guide-deploy.md`](deploy.md) — environment-variable management on deploy targets
- [`guide-pipeline.md`](pipeline.md) — end-to-end picture of how connections feed into provisioning + code-gen
