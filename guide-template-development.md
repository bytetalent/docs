# Template Development

How a Bytetalent template is structured, paired, versioned, and added. Companion to [`guide-base-development.md`](guide-base-development.md) — bases are the always-on scaffold; templates are the opt-in additions.

The actual mechanics + JSON Schemas live in [`bytetalent/templates/CONTRIBUTING.md`](https://github.com/bytetalent/templates/blob/main/CONTRIBUTING.md). This guide covers the conceptual model.

---

## 1. What is a template?

A **template** is an opt-in feature that layers on top of one or more Bytetalent bases at code-gen time. Each template ships paired code + design + brand-kit token bindings for a single feature, on one or more stack bundles.

Templates are how the pipeline supports features that:

- Only some projects need (so they don't bloat the base)
- Have stack-specific implementations (different code for Next.js vs. ASP.NET vs. Astro)
- Have paired design surfaces that should evolve in lockstep with the code

**Examples** (per the pipeline plan's R1-2):

| Template | What it adds |
|---|---|
| `auth-magic-link` | Passwordless email-code sign-in route alongside the base's password sign-in |
| `stripe-subscriptions` | Full Stripe Subs implementation: products, plan gating, customer portal |
| `notifications-center` | Notifications table + bell + page (the base only ships notification primitives) |
| `admin-panel` | Account-level admin pages + role gating |
| `marketing-blog-mdx` | MDX content collection + blog index + article template (Astro-specific) |

If a feature is needed by 80%+ of projects, it belongs in the **base**. If it's optional, it's a **template**.

---

## 2. The matrix concept

Templates are a sparse matrix of **feature × stack bundle**:

```
                          nextjs-clerk-supabase  astro-cloudflare  aspnet-azure-sql
auth-magic-link                  ✅                  N/A                  ✅
stripe-subscriptions             ✅                  N/A                  ✅
notifications-center             ✅                  N/A                  ✅
marketing-blog-mdx               (template)          ✅                  N/A
```

Same `featureKey` across bundles ≠ same code. A template ships **fully-distinct implementations per stack** — different ORM, different framework idioms, different auth library calls. Only the **feature contract** + **design** + **token bindings** are shared across bundles for a given feature.

This is on purpose. A "Stripe Subscriptions" template for Next.js (TypeScript, App Router, Drizzle, route handlers) is fundamentally different code from one for ASP.NET Core (C#, controllers, EF Core, dependency injection). Pretending they share code creates the wrong abstraction.

**Sparse matrix is OK.** Some features only make sense for one stack (e.g., `marketing-blog-mdx` is Astro-specific). Templates declare which `bundleId`s they support.

---

## 3. Anatomy of a template

```
bytetalent/templates/
  features/
    <featureKey>/                    # one folder per feature, kebab-case
      feature.json                   # metadata + supportedBundles
      design.pen                     # paired design (or .placeholder until Pencil session)
      tokenBindings.json             # brand-kit field → Pencil variable map
      README.md                      # what this feature does, when to apply it
      bundles/
        <bundleId>/                  # one folder per supported stack
          template.json              # bundle-specific manifest
          README.md                  # bundle-specific install notes
          files/
            src/...                  # files to copy into the client repo,
            drizzle/...              # mirroring target paths
```

**Required at the feature level:**

- `feature.json` — metadata (featureKey, displayName, description, category, version, supportedBundles)
- `design.pen` (or `design.pen.placeholder` until a Pencil session creates the real one)
- `tokenBindings.json` — even if empty (`{}`)
- `README.md`

**Required per bundle:**

- `template.json` — file manifest, deps, env vars, post-install steps
- `README.md` — install + verification + removal notes for this stack
- `files/` — at least one file under it

The lint script (`scripts/lint-pairing.mjs` in the templates repo) enforces all of the above.

---

## 4. The pairing rule

> **Every code template has a matching design template for the same featureKey + bundleId.** CI lint catches drift.

The pairing rule means: you can't ship code for a feature without also shipping (or slotting) a design for it. The reverse also holds — a design without code is incomplete.

In practice:

- `feature.json` + `design.pen` + `tokenBindings.json` + `README.md` are required at the feature level — a feature can't exist without its design slot
- `bundles/<bundleId>/template.json` + `README.md` + `files/` are required per bundle — code can't ship without the per-stack manifest

The CI workflow (`.github/workflows/ci.yml`) runs the lint on every PR. Failed lints block merges.

---

## 5. `feature.json` — metadata

```json
{
  "featureKey": "auth-magic-link",
  "displayName": "Passwordless email sign-in",
  "description": "Adds a /sign-in-magic route that signs users in via email OTP",
  "supportedBundles": ["nextjs-clerk-supabase"],
  "category": "auth",
  "version": "1.0.0",
  "tags": ["passwordless", "email", "otp"],
  "addsTo": "base"
}
```

**Required fields:**

- `featureKey` — must match the parent folder name; kebab-case
- `displayName` — shown in the setup wizard's feature picker (1–80 chars)
- `description` — one-line summary next to the wizard checkbox (1–280 chars)
- `supportedBundles` — non-empty array of `bundleId`s this template implements; must match `bundles/` subdirectory names
- `category` — one of: `auth`, `billing`, `marketing`, `communication`, `admin`, `data`, `infra`, `other`
- `version` — semver

**Optional fields:** `tags`, `addsTo` (`base` | `template`), `requires` (other featureKeys that must be present first), `conflictsWith` (featureKeys that can't coexist).

---

## 6. `template.json` — bundle manifest

Each bundle's `template.json` declares:

- **`files`** — what files to copy and what action to take (`add`, `replace`, `merge-json`)
- **`deps`** — dependencies to add to the project's `package.json`
- **`envVars`** — required + optional env vars the template consumes
- **`postInstall`** — migrations to run, commands to invoke, notes to display

```json
{
  "bundleId": "nextjs-clerk-supabase",
  "version": "1.0.0",
  "deps": {
    "add": [
      { "name": "react-email", "version": "^3.0.0", "kind": "dependency" }
    ]
  },
  "envVars": {
    "required": ["RESEND_API_KEY"]
  },
  "files": [
    {
      "from": "files/src/app/(auth)/sign-in-magic/[[...sign-in]]/page.tsx",
      "to": "src/app/(auth)/sign-in-magic/[[...sign-in]]/page.tsx",
      "action": "add"
    },
    {
      "from": "files/src/lib/routes.ts",
      "to": "src/lib/routes.ts",
      "action": "replace"
    }
  ],
  "postInstall": [
    {
      "type": "migration",
      "command": "bun run db:push"
    }
  ]
}
```

**File actions:**

- `add` — file doesn't exist in the base; create it
- `replace` — file exists in the base; overwrite it (the template ships a full replacement, not a diff)
- `merge-json` — JSON file (e.g., `package.json` deps); deep-merge keys

For modifications to existing base files (like adding a link to the sign-in page that points to the magic-link route), the template ships the **full replacement** with the modification baked in. The pipeline's apply step replaces the base's file with the template's version.

---

## 7. `tokenBindings.json` — brand-kit ↔ Pencil

```json
{
  "bindings": [
    {
      "brandKitField": "colors.primary",
      "pencilVariable": "$primary",
      "usage": "submit button background, OTP code highlight"
    },
    {
      "brandKitField": "logo.url",
      "pencilVariable": "$logo-image",
      "usage": "header image in the OTP email"
    }
  ]
}
```

When a project applies a template, the pipeline:

1. Reads `tokenBindings.json` from the template
2. For each binding, looks up the field on the project's brand kit
3. Sets the corresponding Pencil variable in the rendered design
4. Result: per-project email + UI screens render with the project's actual brand

**Empty token bindings** (`{ "bindings": [] }`) are valid — a feature might have no brand-driven design (e.g., a backend-only template).

---

## 8. Versioning + pinning

Templates follow semver. **Bump `version`** in:

- `feature.json` (the feature-level version) when the feature contract or supported bundles change
- `bundles/<bundleId>/template.json` (the bundle-level version) when this bundle's implementation changes

Often both bump together. The feature-level version is the public API; bundle-level versions track per-stack implementation work.

**Templates are pinned by git tag/SHA in client repos.** When the pipeline scaffolds a generated client repo with `auth-magic-link@v1.2.0`, the client's `bytetalent-base.json` records:

```json
{
  "templates": [
    { "featureKey": "auth-magic-link", "version": "1.2.0", "sha": "abc123" }
  ]
}
```

Even if the template later evolves to v2.0.0, the client at v1.2.0 keeps getting v1.2.0 on rebuild. Strategy A snapshot (decision #7) — diff-aware upgrades are deferred.

**Never re-tag** an existing version — that breaks pin contracts in already-generated client repos.

---

## 9. Adding a new bundle to an existing feature

When a new stack base ships and an existing feature should support it:

1. Verify the feature is applicable to the new stack (some features are stack-specific by nature — `marketing-blog-mdx` for Astro doesn't make sense for ASP.NET)
2. Add the new `bundleId` to `feature.json`'s `supportedBundles`
3. Create `features/<featureKey>/bundles/<newBundleId>/`
4. Implement the feature for that stack:
   - `template.json` declaring files, deps, env vars
   - `README.md` with stack-specific install + verification notes
   - `files/...` mirroring target paths in the base for that stack
5. **Bump the feature-level version** in `feature.json` (minor — it's additive)
6. Run `node scripts/lint-pairing.mjs` to verify the pairing rule
7. Open a PR — CI runs the lint on every push
8. After merge, tag the new version: `git tag v1.1.0 && git push --tags`

---

## 10. Adding a new feature

When R1-1 evidence (or real-engagement signal) shows a feature is needed:

1. Pick a `featureKey` (kebab-case, descriptive)
2. Create `features/<featureKey>/`:
   - `feature.json` with `version: "0.1.0"` (pre-stable until a real engagement validates the contract)
   - `design.pen.placeholder` — replace with `.pen` after a Pencil session
   - `tokenBindings.json` — even if empty: `{ "bindings": [] }`
   - `README.md` describing what it does + when to apply
3. Create at least one `bundles/<bundleId>/` directory with `template.json`, `README.md`, and `files/`
4. Run the lint locally
5. Open a PR
6. After CI passes + review, merge + tag at the appropriate semver

**When NOT to add a feature:**

- 80%+ of projects need it → it belongs in the **base**, not a template
- It's nice-to-have but no real engagement has asked for it → defer until R1-1 evidence
- It overlaps significantly with an existing template → extend the existing one

---

## 11. The R1-1 dependency

The pipeline plan's R1-2 explicitly says:

> **Initial template set (informed by R1-1 findings on what's not already in the bases).**

R1-1 is the 6–10 staging projects across both bases that surface real coverage gaps. Building templates **before** R1-1 means guessing what consultants need.

**The discipline:** add a template only when:

1. A real engagement has asked for it, OR
2. R1-1 surfaces it as a recurring pattern across multiple staging projects, OR
3. It's foundational and obviously needed (e.g., the `auth-magic-link` example template that exists primarily to validate the templates-repo structure)

Premature template implementation is expensive — the wrong abstraction lands as code that has to be deleted or restructured. Defer until evidence justifies it.

---

## Related

- [`guide-base-development.md`](guide-base-development.md) — how bases are structured (the layer templates layer on top of)
- [`guide-pencil.md`](guide-pencil.md) — Pencil MCP workflow for `.pen` files
- [`bytetalent/templates/README.md`](https://github.com/bytetalent/templates/blob/main/README.md) — the templates repo's purpose + matrix concept
- [`bytetalent/templates/CONTRIBUTING.md`](https://github.com/bytetalent/templates/blob/main/CONTRIBUTING.md) — the actual mechanics of adding a template
