# Template Development

How a Bytetalent template is structured, paired, versioned, and added. Companion to [`guide-base-development.md`](guide-base-development.md) — bases are the always-on scaffold; templates are the opt-in additions.

The actual mechanics + JSON Schemas live in [`bytetalent/templates/CONTRIBUTING.md`](https://github.com/bytetalent/templates/blob/main/CONTRIBUTING.md). Rules for anatomy, `feature.json` fields, `template.json` file actions, token bindings, pairing rule, versioning, and the add-bundle/add-feature recipes live in the **`architecture/template-development` skill**. This guide covers the conceptual model.

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

## Related

- `architecture/template-development` skill — anatomy, feature.json + template.json schemas, pairing rule, versioning, add-bundle/add-feature recipes
- [`guide-base-development.md`](guide-base-development.md) — how bases are structured (the layer templates layer on top of)
- [`guide-pencil.md`](guide-pencil.md) — Pencil MCP workflow for `.pen` files
- [`bytetalent/templates/README.md`](https://github.com/bytetalent/templates/blob/main/README.md) — the templates repo's purpose + matrix concept
- [`bytetalent/templates/CONTRIBUTING.md`](https://github.com/bytetalent/templates/blob/main/CONTRIBUTING.md) — the actual mechanics of adding a template
