# Pipeline — End-to-End

How the four kinds of Bytetalent repos fit together to turn a consultant's idea into a deployable client product. This is the **big-picture map** — companion to [`guide-base-development.md`](guide-base-development.md) and [`guide-template-development.md`](guide-template-development.md), which cover the inner workings of each piece.

---

## 1. The four kinds of repos

Every Bytetalent engagement involves repos in (up to) four roles:

| Role | Repo example | Lifecycle |
|---|---|---|
| **Pipeline app** | `bytetalent/bt-ai-web` | One; long-lived; product itself |
| **Universal docs** | `bytetalent/docs` | One; long-lived; cross-stack standards |
| **Base** | `bytetalent/astro-cloudflare-base`, `bytetalent/nextjs-clerk-supabase-base`, future `expo-…` | One per stack; long-lived; dogfooded at stable URLs |
| **Templates** | `bytetalent/templates` | One; long-lived; sparse matrix of feature × stack-bundle |
| **Generated client repo** | `{agency-org}/{project}` or `{client-org}/{project}` | One per pipeline project; lifetime of the engagement |

The first four are **Bytetalent-owned and reused**. The fifth is **per-engagement and never owned by Bytetalent**.

---

## 2. The big picture

```
                ┌──────────────────────────────────────┐
                │        bytetalent/bt-ai-web          │
                │        (the pipeline app)            │
                │                                      │
                │  setup wizard, phase pipeline,       │
                │  code-gen agent, project tracking    │
                └──────────────┬───────────────────────┘
                               │
            ┌──────────────────┼──────────────────────────┐
            │                  │                          │
            ▼                  ▼                          ▼
  ┌───────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
  │  bytetalent/docs  │  │ bytetalent/bases │  │ bytetalent/templates │
  │                   │  │                  │  │                      │
  │ universal cross-  │  │ astro-cloudflare │  │ features × stack-    │
  │ stack standards;  │  │ nextjs-clerk-    │  │ bundle matrix; opt-  │
  │ guide-arch, code, │  │   supabase       │  │ in additions like    │
  │ db, api, design,  │  │ expo-clerk-…     │  │ auth-magic-link,     │
  │ etc.              │  │ aspnet-azure-…   │  │ stripe-subs, etc.    │
  └─────────┬─────────┘  └────────┬─────────┘  └──────────┬───────────┘
            │                     │                       │
            │ referenced by       │ scaffolded from       │ layered on top
            │ all bases'          │ at code-gen time      │ at code-gen time
            │ docs/ overlays      │                       │
            │                     ▼                       ▼
            │           ╔═══════════════════════════════════════╗
            └─────────► ║   {org}/{project} — generated client  ║
                        ║                                       ║
                        ║   - base files at pinned version      ║
                        ║   - template files at pinned versions ║
                        ║   - novel modules per PRD             ║
                        ║   - bytetalent-base.json (audit trail)║
                        ╚═══════════════════════════════════════╝
                                          │
                                          ▼
                                deploy to Vercel / Cloudflare /
                                Azure / EAS — client owns the repo,
                                client's infrastructure runs it
```

---

## 3. End-to-end flow

A consultant creates a new project. Here's what happens, step by step.

### Phase 0 — Setup

Consultant opens the pipeline app's setup wizard. Picks:

- **Project type** — webapp, marketing site, mobile, etc.
- **Stack bundle** — `nextjs-clerk-supabase`, `astro-cloudflare`, `aspnet-azure-sql`, `expo-clerk-supabase`, etc.
- **Client** — existing or new
- **Brand kit** — pick from client's existing kits (R2.6) or create new
- **Repo topology** — where the generated repo lives: agency's GitHub org, client's, or transfer-on-completion (decision #2)
- **Included features** — checkboxes per feature listed in `bytetalent/templates`'s feature catalog, filtered to those whose `supportedBundles` includes the chosen bundle

The pipeline records all this as a `projects` row in its DB. No code is written yet.

### Phases 1-5 — AI-driven design phases

Sequential, each producing an approved deliverable that feeds the next:

1. **Idea capture + AI vision review** — consultant writes the idea, AI suggests refinements
2. **PRD** — generated from idea + audience, reviewed, approved
3. **Brand kit** — pick existing client kit OR generate new (R2.6's three-paths model)
4. **Design** — Pencil-paired baseline rendered with this project's brand kit (via token bindings); manual export today, R4 worker future
5. **Architecture review** — Opus/Sonnet generates 8-section architecture doc, validated against PRD

Each phase output is stored as a `project_deliverables` snapshot — frozen at approval time so later edits to the source kit/PRD don't change historical deliverables.

### Phase 5.5 — Provisioning (R1-3.6, parallel track)

Consultant authorizes the pipeline to provision per-project infrastructure:

- Vercel project (or Cloudflare Pages, Azure, EAS, depending on stack)
- Supabase project + migrations + RLS policies + storage buckets
- Clerk app config (webhooks, OAuth providers)
- Stripe products + prices + webhook endpoint (if billing)
- Resend domain + API key
- DNS records

This runs **in parallel with code-gen** — they only converge at first deploy. The consultant's *active interaction time* is short; wall-clock includes manual walls (KYC, DNS verification) the pipeline guides through.

### Phase 6 — Code generation (R1-4)

This is where bases + templates + the agent come together. Step by step:

#### 6a. Create the empty client repo

The pipeline calls GitHub via the Bytetalent App install token (R1-3) to create a new empty repo at `{org}/{project-slug}`. The org is whichever the consultant picked: agency, client, or agency-then-transfer.

#### 6b. Pull base files (scaffold)

The pipeline reads `bytetalent-base.json` from the chosen base repo at its current pinned version. Pulls all source files (`git clone --depth 1 --branch v1.2.0` or tarball-via-GitHub-API). Copies into the new repo with brand-kit substitutions applied to:

- `globals.css` `@theme` block (CSS custom properties)
- Project name + URL placeholders in config files
- Env-var references throughout

Commits as the new repo's initial commit.

#### 6c. Resolve and apply templates

For each feature the consultant checked in setup:

1. Find `features/<featureKey>/bundles/<bundleId>/` in the templates repo at its current pinned version
2. Apply each file in `template.json`'s `files[]` array per its action:
   - `add` — create new file
   - `replace` — overwrite existing base file
   - `merge-json` — deep-merge JSON keys (e.g., into `package.json`)
3. Run any `postInstall` steps (migrations, commands)
4. Substitute brand-kit values into Pencil variables per `tokenBindings.json` (drives the per-project email + UI design)

Commits as the second commit on the new repo.

#### 6d. Generate novel feature modules

For each PRD must-have feature **not covered** by templates, the agent generates a complete module following the standard sequence (per [`guide-arch.md`](guide-arch.md) §1):

1. Drizzle schema → 2. migration SQL → 3. RLS policies → 4. API route handlers → 5. hooks → 6. page (`page.tsx` + `page-client.tsx`) → 7. components

Each module = one PR opened against the new client repo. PR description references the PRD section + design page that drove the module. Consultant reviews + merges in GitHub.

#### 6e. Audit trail

Pipeline writes `bytetalent-base.json` to the new repo's root, capturing:

```json
{
  "base": "bytetalent-nextjs-clerk-supabase-base",
  "baseVersion": "1.2.0",
  "baseSha": "abc123",
  "templates": [
    { "featureKey": "auth-magic-link", "version": "1.0.0", "sha": "def456" },
    { "featureKey": "stripe-subscriptions", "version": "0.3.0", "sha": "ghi789" }
  ],
  "generatedAt": "2026-04-30T14:23:00Z",
  "generatedBy": "consultant-id"
}
```

This is the **audit trail** for the rest of the engagement. Future rebuilds pin to these exact versions even if base/templates evolve (Strategy A snapshot, decision #7).

### Phase 7 — Bundle assembly + deploy (R1-5)

When the consultant marks the project complete:

1. Pipeline generates a bundle index — client repo URL + Pencil files + brand kit + PRD + architecture PDFs + dashboard URL + deployed URL
2. Email index to the client
3. Auto-revoke agency GitHub access if topology was "agency-then-transfer"
4. R1-6 coherence checks run:
   - Brand kit colors actually appear in the design's exported tokens
   - PRD entities exist in the Drizzle schema
   - All must-have features have at least one matching design page AND code module

---

## 4. What goes where — detailed map

| What | Where | Why there |
|---|---|---|
| Universal coding standards | `bytetalent/docs` | Same across stacks; one source of truth |
| Stack-specific patterns | `{base}/docs/guide-stack.md` overlay | Differ per framework; can't be universal |
| Pre-wired infrastructure (apiError, resolveAccount, ETag helpers) | `{base}/src/lib/` | Real running code; the base IS the infrastructure |
| Section components, dashboard shell | `{base}/src/components/` | The base ships these |
| Optional features (auth-magic-link, stripe-subs, blog-mdx) | `bytetalent/templates/features/<key>/bundles/<id>/files/` | Stack-specific implementations of optional features |
| Per-project content (Hero copy, FAQ items, pricing tiers) | Generated `client-repo/src/content/site-content.ts` | Per-engagement; the pipeline produces this from PRD + brand voice |
| Per-project schema (Drizzle tables for the actual product) | Generated `client-repo/src/lib/db/schema/<entity>.ts` | Per-engagement; the agent generates from PRD must-haves |
| Per-project pages, hooks, components for novel features | Generated `client-repo/src/{app,hooks,components}/` | Per-engagement; the agent generates from PRD |
| Universal access-path discipline + ETag/RLS rules | Inherited from `bytetalent/docs` | The base + templates + agent all follow these |

---

## 5. Versioning + how updates flow

Three pin-points exist across the pipeline:

### Base version

- Recorded in `client-repo/bytetalent-base.json` as `baseVersion + baseSha`
- Frozen at code-gen time
- **Doesn't auto-update** when the base evolves (Strategy A)
- Updates require manual back-port today; Strategy B diff tooling deferred until R1-1 evidence justifies building it

### Template versions

- Recorded per template in the same `bytetalent-base.json`
- Frozen at code-gen time
- Same Strategy A behavior — doesn't auto-update

### Documentation references

- `bytetalent/docs` is referenced via raw-Github-URL links in each base's `docs/` overlay
- These point at `main` — updates flow immediately
- Acceptable because docs are non-load-bearing in the runtime sense; reading the latest version is preferable to reading a frozen one

### When something does need to flow

If a critical security fix lands in the base, the consultant needs to:

1. Manually back-port the fix to each affected client repo, OR
2. Use future Strategy B tooling once it exists, OR
3. Open a PR from the consultant + agent collaborating in the client repo (current default)

The pipeline's R1-1 staging projects will surface whether back-port pain is acute enough to justify Strategy B work.

---

## 6. R1-1's role — why bases come first

The pipeline plan's R1-0 (bases) explicitly blocks R1-1 (the 6-10 staging projects), which itself blocks much of R1-2 (templates) and R4 (Pencil worker reliability gate).

R1-1 generates the evidence that drives:

- **Base improvements** — what consistently gets generated from scratch that *should* have been in the base
- **Template needs** — what features keep appearing across staging projects but aren't in the base
- **Pencil capacity ceiling** — at what page count / app complexity does design quality drop?
- **Convention violations** — does the agent generate code that fights base conventions? (skill-prompt bugs or doc-gap signals)
- **Upgrade-pain signal** — if a base evolves between projects 3 and 6, how painful is bringing 3-5 forward? (decides whether Strategy B becomes a real tier)

Without R1-1, template implementations are speculative. The plan's R1-2 explicitly says "Initial template set (informed by R1-1 findings)" — building before that is the wrong sequence.

---

## 7. The dogfood promise

Strategic decision #3 from the plan: **bases are real running apps, deployed at stable URLs.** Not code skeletons.

Consequences:

- `bytetalent-astro-cloudflare-base` is deployed at `marketing.bytetalent.com`
- `bytetalent-nextjs-clerk-supabase-base` is deployed at `base.bytetalent.com`
- Every base change ships to the dogfood URL before generated clients see it
- Regressions show up at the dogfood URL first
- The base is a working reference for "how do we do X in stack Y?" — not aspirational

This means:

- A base PR that breaks the build doesn't merge (CI catches it)
- A base change that passes CI but breaks the dogfood deploy is rolled back (rollback in Vercel/Cloudflare dashboard)
- Updates to the dogfood URL are visible to the team — accountability without ceremony

---

## 8. Common confusions, cleared up

### "Why doesn't the base just include all the templates?"

If 80%+ of projects need a feature, it goes in the base. If it's optional (some projects do, some don't), it's a template. Putting Stripe Subscriptions in the base means every dogfood deploy includes it, every generated repo includes it, even projects that don't bill anything. That bloats the base and complicates the scaffold.

### "Why don't all bases share the same components?"

The Astro base's components are `.astro` files; the Next.js base's are React `.tsx`; an ASP.NET base's would be Razor partials. Different stacks, different idioms. The strategic decision #8 (`bytetalent/web-sections` shared package) is **deferred to R1.5** — extract a shared package only after the second copy-paste pass surfaces what's actually shared.

### "Why does the client repo have a `bytetalent-base.json`?"

It's the audit trail. Without it, six months from now a developer can't tell: "This client repo was scaffolded against which base version, with which templates?" The manifest answers that. It's also what enables Strategy B (diff-aware upgrades) when that tooling lands.

### "What if the base changes after a project ships?"

By design, it doesn't flow into existing client repos. The consultant decides per-project whether to back-port. If the base drops a v2.0.0 that restructures folders, projects on v1.x stay on v1.x. If a security fix lands in v1.5.1, the consultant manually applies it (or uses future Strategy B tooling).

### "Where does the agent live?"

In `bytetalent/bt-ai-web` — the pipeline app. It has access to:

- The project's PRD, brand kit, design, architecture
- The base repo (read-only)
- The templates repo (read-only)
- The skills + phase prompts in the pipeline's DB
- The Anthropic API via the proxy layer

It doesn't live in the bases or templates. Those are pure scaffolding inputs.

### "What about agencies that aren't Bytetalent?"

Strategic decision #1: Bytetalent's app is the *product*. Any consulting firm or agency is a potential customer. Bytetalent itself is just one agency using the product. The pipeline supports multi-tenancy — `accounts` with `accountType = "agency"` are the pipeline operators, `clients` = each agency's clients, `projects` = engagements.

**Account-scoped base customization** (agencies forking the base for their own conventions) is **deferred to backlog post-R6**. R1 ships with Bytetalent's bases as the only option.

---

## 9. Agency vs. client roles in the pipeline

The pipeline has two distinct user types with different access patterns. Understanding the boundary prevents accidentally exposing agency-internal data to client users.

### Agency role in the pipeline

Accounts with `accountType = "agency"` are the **operators** of the pipeline. They:

- Create and configure projects (setup wizard)
- Author and iterate on every phase: idea, PRD, brand kit, design, architecture, code
- Own credentials (Anthropic key, GitHub, Vercel, etc.)
- Publish approved deliverables to the client portal
- Review and merge the generated PRs in the client GitHub repo

Inside an agency account, `consultant` is the default member role for the people doing the work. `owner` holds billing and admin authority. `viewer` is for read-only observers (partners, auditors).

### Client role in the pipeline

Accounts with `accountType = "client"` are the **recipients** of the pipeline output. They:

- View published deliverables via the client portal (`/portal/[token]`)
- Approve or request changes on deliverables
- Cannot see the pipeline's internal phase work (idea reviews, PRD iterations, prompt history)
- Cannot create projects or access any agency-level route

Inside a client account, `product-owner` is the key role for whoever approves scope and deliverables. `reviewer` is used when approval is delegated to a specific person per phase.

### Authorization boundary

The pipeline enforces this boundary at the API layer:

- All `/api/projects/...` pipeline routes require `accountType = "agency"` (the project's `accountId` must belong to an agency account, not a client account).
- The `/portal/[token]` route is intentionally outside the dashboard auth shell — it accepts a project portal token and shows only what has been published.
- Client accounts may only reach pipeline data through the portal, not through any `/api/projects` or `/api/phases` route.

---

## Related

- [`guide-base-development.md`](guide-base-development.md) — how a base is structured + versioned
- [`guide-template-development.md`](guide-template-development.md) — how a template is paired + applied
- [`guide-arch.md`](guide-arch.md) — org model section (§9) — account types + role taxonomy
- [`guide-arch.md`](guide-arch.md) — universal access-path discipline that all the above inherit
- [`guide-pencil.md`](guide-pencil.md) — Pencil MCP for `.pen` design files
