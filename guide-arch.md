# Architecture Standards

Cross-cutting architecture decisions that apply to every Bytetalent project. The principle: **one canonical path per concern, no parallel implementations.** Multiple paths for the same concern create drift, security gaps, and incident-shaped surprises.

Stack-specific routing details (Next.js App Router, Astro endpoints, ASP.NET controllers, Expo Router) live in each base or product's `docs/guide-stack.md` overlay.

---

## 1. Standardized Access Paths

Every cross-cutting concern has **one** canonical path. If you think you need a new path for an existing concern, propose it as an architectural change first — don't quietly add a parallel implementation.

| Concern | Canonical path | Notes |
|---|---|---|
| Database reads/writes (app schema) | **Drizzle** (or stack-equivalent ORM) at `src/lib/db` | All app tables — accounts, projects, flows, billing, etc. |
| Vault secrets (read/write/delete) | **Drizzle via `src/lib/connections/secrets.ts`** | Calls `vault.create_secret`, `vault.decrypted_secrets`, etc. directly. Never via PostgREST/`supabase-js` for Vault. |
| Auth resolution (current user → app account) | **`resolveAccount()`** in `src/lib/session.ts` | Single entry point. Wraps the auth provider session lookup + `getOrCreateAccount`. |
| Auth gating at the edge | Stack's middleware/proxy primitive (Clerk middleware, Astro middleware, ASP.NET pipeline) | Different concern from session resolution. |
| API error responses | **`apiError()`** in `src/lib/api-error.ts` | Classifies by error type; structured response; safe 5xx messages. |
| Frontend → backend | **`fetch('/api/...')` from hooks** | Never call providers / Supabase REST directly from the browser. |
| Outbound HTTP to LLM/tool providers | **Connection adapters** (`src/lib/connections/providers/<slug>.ts`) for `adapter.test()` only; everything else goes through the **proxy layer** at `POST /api/proxy/...` | See "Proxy is the sole provider egress" below. |
| Service-role / RLS-bypass DB ops | **Drizzle** with documented intent in code comment | Don't add new clients; the legacy split-client pattern is a deprecated path. |
| Supabase Storage operations (file upload, signed URL generation) | **`src/lib/supabase/admin.ts`** via the storage helper layer (`src/lib/storage/`) | Storage isn't expressible through Drizzle. One legitimate caller per project; don't add direct `supabaseAdmin.storage` calls outside the storage helper layer. See app-specific `CLAUDE.md` for the exact caller. |
| ID generation | `crypto.randomUUID()` | Pick one, stick with it. Don't reach for nanoid/cuid2 alongside. |

### Why one path matters

Real example: when secret reads/writes were routed through both the Vault (via Drizzle direct SQL) **and** PostgREST (via `supabase-js`), the PostgREST path failed under load with a PGRST002 schema-cache miss while the Drizzle path worked. The two paths added no value — same DB, same ACL — but the second path created a real production incident. Single path eliminates this entire class of problem.

---

## 2. Proxy is the Sole Provider Egress

Every outbound HTTP call to an LLM or external tool provider — Anthropic, OpenAI, Google, xAI, Slack, GitHub, Linear — routes through `POST /api/proxy/...`.

The proxy enforces:

1. **Credential resolution** — the multi-tier ladder (e.g., client → consultant → platform), single implementation.
2. **Logging + billing** — `keySource`, tokens in/out, latency, project + phase + prompt versions. Per-account charge-back and audit depend on this being captured for every call.
3. **Prompt assembly** — system prompt + skill chunks + project context + user input. Versioned and audited.
4. **Caching** — provider-specific caching only works with consistent system prompts. Centralized assembly → caching just works.
5. **Rate limits + retries** — one place per provider.
6. **Provider swap** — change a config row, not application code.

The proxy is HTTP, not a function call. Server-side code calls it via `fetch('/api/proxy/...')` — same path as a browser request. Cost: one internal HTTP hop. Worth it for the consistency.

**Sole exception:** `adapter.test()` during BYOK (bring-your-own-key) validation. Pre-proxy because there's no project context, no phase, no skills — the call's purpose is _to validate the credential being added_. Lives entirely inside `connections/providers/` adapters and the connection HTTP routes.

---

## 3. ETag / If-Match — Optimistic Concurrency

All API endpoints that return a single resource must include an `ETag` header. The ETag value is the row's `version` integer, stringified.

```
GET /api/flows/123
→ ETag: "42"
```

**Write requests** (`PUT`, `PATCH`, `DELETE`) must include `If-Match`:

```
PUT /api/flows/123
If-Match: "42"
```

| Outcome | HTTP status |
|---|---|
| Version matches — update succeeds | `200` + new `ETag: "43"` |
| Version mismatch — concurrent edit detected | `412 Precondition Failed` |
| `If-Match` header missing | `428 Precondition Required` |

`If-Match: *` is **not** accepted — a specific version must always be supplied.

List endpoints don't return ETags per item. Use the individual resource endpoint to get the current version before writing.

See [`guide-api.md`](guide-api.md) for the full API contract.

---

## 4. Row Versioning

Every user-owned entity carries a `version` integer column for optimistic locking. Starts at `1`, increments on every write.

```typescript
version: integer("version").notNull().default(1);
```

Update pattern — always include the version in the `WHERE` and increment in the `SET`. If zero rows updated, a concurrent modification occurred:

```typescript
const result = await db
  .update(flows)
  .set({ ...data, version: sql`${flows.version} + 1`, updatedAt: new Date() })
  .where(and(eq(flows.id, id), eq(flows.version, expectedVersion)))
  .returning();

if (result.length === 0) {
  throw new ConflictError("Flow was modified by another request");
}
```

See [`guide-db.md`](guide-db.md) for the full schema convention (audit fields, soft-delete, version, JSONB).

---

## 5. Connections Framework

External-service connections (GitHub, Slack, Linear, Anthropic, Resend, Vercel, Supabase, Clerk, etc.) follow a unified strategy.

### Two-level model — credential / config separation

```
accounts ──────────────────────────────────────────────┐
  └── connections (Vault-encrypted credential)          │
        accountId  (always set — whose account)         │
        clientId   (nullable — null = consultant's own) │
        provider   ("github" | "anthropic" | "resend"…) │
        vaultSecretId (encrypted token/key)             │
                                                        │
clients ─────────────────────────────────────────────┐  │
  └── (linked via connections.clientId)              │  │
                                                     │  │
projects ────────────────────────────────────────────┤  │
  └── project_connections                            │  │
        projectId                                   ◄┘  │
        connectionId                                ◄───┘
        config JSONB  ← target, not secret
                        e.g. { repo: "acme/web", branch: "main" }
                             { channel: "#deployments" }
                             { model: "claude-opus-4-7" }
```

Invariants:
- `connections` holds the encrypted auth token/key — written once, reused across many projects.
- `project_connections` holds the target config (repo slug, channel name, model ID) — different per project, no secrets.
- `clientId = null` means a consultant-owned credential.
- `clientId = some-id` means a client-owned credential.
- Multiple projects can share one `connection` row.

### Variants — three kinds, one UX

Every provider falls into exactly one of three variants. The provider adapter declares its variant; UI and proxy branch on that single field, not on provider name.

| Variant | Providers | How it connects | What gets stored |
|---|---|---|---|
| **`app_install`** | GitHub (App), Slack (bot install) | Provider install page → `installationId` returned via webhook → backend mints short-lived tokens on demand from a platform-level private key | `installationId` only — no tokens persisted |
| **`oauth`** | Linear, Notion, Figma, Vercel, Google, Microsoft, Atlassian | OAuth 2.0 authorize + code-exchange; store access + refresh tokens, rotate before expiry | `accessToken`, `refreshToken`, `expiresAt`, granted `scopes` |
| **`api_key`** | Anthropic, OpenAI, Google AI Studio, xAI, Twilio, Telegram, SMTP | Inline form; validated with a test call before saving | `apiKey` (and `apiSecret` / `botToken` where the provider needs two) |

### Provider adapter interface

```typescript
export interface ProviderAdapter {
  // Identity
  id: string;
  product?: string;
  category: "tool" | "channel" | "model";
  label: string;
  icon: string;

  // Variant + config
  variant: ConnectionVariant;
  requiredScopes?: string[];
  configFields?: ConfigField[];

  // Lifecycle
  startConnect(ctx: ConnectCtx): Promise<{ redirectUrl: string } | { inlineForm: true }>;
  handleCallback?(code: string, state: string): Promise<ConnectionRow>;
  saveApiKey?(fields: Record<string, string>): Promise<ConnectionRow>;
  refresh?(row: ConnectionRow): Promise<ConnectionRow>;
  revoke?(row: ConnectionRow): Promise<void>;
  test(row: ConnectionRow): Promise<{ ok: boolean; detail?: string }>;
  mintOutboundToken(row: ConnectionRow): Promise<string>;
}
```

### Unified status vocabulary

| Status | Meaning | User action |
|---|---|---|
| `connected` | Healthy; last test or use succeeded | — |
| `expired` | Token expired and auto-refresh failed | Reconnect |
| `revoked` | Upstream uninstall/revoke detected via webhook or 401 | Reconnect |
| `error` | Test connection or last outbound call errored | Inspect + retry |
| `disconnected` | User paused the connection | Reconnect |

### Secret storage — Supabase Vault (pgsodium)

All user-supplied secrets use Supabase Vault. Encryption keys stay inside the database, not in env vars. Decryption is a SQL function callable only under RLS — application code never sees raw ciphertext.

Vault columns are never selected by default. A dedicated `withSecrets()` helper in `src/lib/connections/secrets.ts` is the only path that retrieves decrypted values. List pages, detail reads, and audit logs get the row with secret columns omitted — defense in depth against accidental logging.

Platform-level secrets (GitHub App private key, platform Anthropic key, Stripe keys, auth-provider secret) remain in the host's env vars. Vault is only for per-user/per-account secrets.

### GitHub as a GitHub App, not OAuth App

For repo access, use the **GitHub App** model (not an OAuth App). Different concern from social login (which is OAuth via the auth provider).

Rationale:
- Bot identity for PR comments
- Per-repo scoping at install time
- Short-lived (1-hour) tokens minted on demand
- Org-wide adoption from one install
- Native webhook lifecycle (`installation.created/deleted`)
- Marketplace path (apps only, not OAuth)

`GITHUB_APP_PRIVATE_KEY` is a platform secret in Vault, not per-user.

---

## 6. Real-time Updates

For projects that need live status updates (run/job execution, notifications), use **Supabase Realtime** Postgres change listeners — no polling, no third-party WebSocket service.

Pattern:
```
Run starts → run.status = 'running' written to DB
           → Supabase Realtime broadcasts INSERT/UPDATE on runs table
           → client subscription receives change event
           → UI updates without refetch
```

Don't add a separate WebSocket service (Pusher, Ably, etc.) when Realtime covers the use case.

---

## 7. Infrastructure Stack — Default Choices

Where there's overlap between Vercel, Upstash, and Supabase, the decision favours the service already in the stack with the deepest native support for that concern. Avoid adding a fourth service when one of the three already covers it.

| Concern | Default | Rationale |
|---|---|---|
| App hosting (web) | Vercel | Primary host for Next.js + Astro deploy targets vary |
| Database | Supabase PostgreSQL | Primary DB with Drizzle ORM |
| Auth | Clerk | User identity, session management |
| Secrets / env vars | Host's env vars (Vercel, Cloudflare, Azure) | Native to host, no extra service |
| File / asset storage | Supabase Storage | Already in stack — avoid Vercel Blob duplication |
| Real-time updates | Supabase Realtime | Already in stack — Postgres change listeners |
| Caching | Upstash Redis | More control than Vercel KV (which is Upstash under a wrapper) |
| Rate limiting | Upstash Ratelimit | Built on Upstash Redis — zero extra setup |
| Job queue | Upstash QStash | HTTP-based, works with serverless, supports retries and delays |
| Scheduled triggers | Upstash QStash | Unified with the job queue |
| Simple periodic tasks | Vercel Cron | Built-in to Vercel, no overhead for lightweight schedules |
| Payments | Stripe | Standard for the billing data model |
| Transactional email | Resend | + React Email for templates |
| Error tracking | Sentry | Standard |
| Observability (logs/metrics/traces) | Grafana Cloud | Loki + Prometheus + Tempo in one platform |
| Product analytics | PostHog | More powerful than basic page-view counters |

### Overlap decisions to remember

- **Vercel KV vs. Upstash Redis** → Upstash directly. KV is Upstash with a wrapper.
- **Vercel Blob vs. Supabase Storage** → Supabase Storage. Already in stack.
- **Vercel Cron vs. QStash** → both, for different purposes. QStash for the queue + delays + fan-out; Vercel Cron for simple periodic tasks (monthly invoices, cleanup).
- **Supabase Realtime vs. Pusher / Ably** → Realtime. Already in stack.

---

## 8. Security Mindset

### Auth guard pattern

Every server-side handler follows:

1. Schema validation on input (zod or stack equivalent)
2. Auth check — reject unauthenticated requests
3. Authz check — verify the caller has access to the resource
4. Business logic

All route handlers and server actions follow this pattern. Never put authz checks only in the UI.

### Content Security Policy

CSP is configured at the framework's headers config. Restricts scripts to `self` + auth-provider domains. When adding a new external service, update CSP `connect-src` and other relevant directives.

---

## 9. Org Model

The pipeline app (`bt-ai-web`) uses a three-account-type model that governs what users can do and what roles they may hold. This section documents the model for reference when building features that are account-type-aware.

### Three account types

| accountType | Who | Purpose |
|---|---|---|
| `agency` | Consultants and small agencies | Primary pipeline users. Create projects, run phases, manage clients, own credentials. |
| `client` | Client companies | Invited by an agency. Access limited to their own projects and deliverables via the client portal. |
| `platform` | Bytetalent | The super-admin org. Exactly one instance in production. |

The `accountType` value is stored on `accounts.accountType`. Every account is exactly one type.

### Member role taxonomy

Each account type has its own valid set of membership roles. Roles are stored in `account_memberships.role` (a single enum). The application layer validates the combination with `isValidRoleForAccountType(role, accountType)` before any write.

| accountType | Role | Description |
|---|---|---|
| agency | `owner` | Account creator; full admin and billing authority. |
| agency | `consultant` | Pipeline operator; creates and manages projects. Default invited role. |
| agency | `viewer` | Read-only observer (partner, auditor). |
| client | `owner` | Account creator; primary contact for the agency. |
| client | `product-owner` | Decides scope; approves deliverables. |
| client | `reviewer` | Approves specific phases. |
| client | `payor` | Billing relationship (may be the same person as owner). |
| client | `stakeholder` | Read-only access to deliverables. |
| client | `viewer` | Read-only; broadest access within client type. |
| platform | `owner` | Bytetalent founder or top admin. |
| platform | `staff` | Bytetalent employees with platform-wide visibility. |
| platform | `viewer` | Read-only platform observers. |

Legacy roles `admin` and `member` remain in the enum during the transition window. All new writes should use the new vocabulary. The helper `isAdminEquivalent(role)` accepts both `"owner"` and the legacy `"admin"` during the window.

### Application-layer validator

Use `isValidRoleForAccountType(role, accountType)` (in `src/lib/auth/roles.ts`) before inserting or updating any `account_memberships` row. This is the single enforcement point. Never write a role directly without running this check.

```typescript
// Example usage before creating a membership row
if (!isValidRoleForAccountType(role, account.accountType)) {
  return apiError(400, "invalid_role", `Role '${role}' is not valid for ${account.accountType} accounts`);
}
```

### Super-admin vs. platform accountType

These are two independent concepts that can both be true for the same user:

- **`accountType = "platform"`** — a DB label on Bytetalent's account row. Used for data-model logic (e.g., exclude platform from agency-facing queries).
- **`isSuperAdmin()`** — a Clerk-org membership check (`CLERK_ADMIN_ORG_ID`). Controls access to `/admin/*` routes. Works independently of `accountType`.

---

## 10. Related Standards

- [`guide-code.md`](guide-code.md) — naming, component standards, security, testing
- [`guide-db.md`](guide-db.md) — schema conventions, audit fields, soft-delete, RLS templates
- [`guide-api.md`](guide-api.md) — REST handlers, ETag/If-Match contract, error response shape
- [`guide-design.md`](guide-design.md) — tokens, typography, theming, stat cards, marketing pages
- [`guide-deploy.md`](guide-deploy.md) — environment principles (commands in stack overlays)
