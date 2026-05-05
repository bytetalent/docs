---
name: Template Development
description: How to structure, pair, version, and lint a Bytetalent feature template — anatomy, feature.json fields, bundle file actions, token bindings, and the pairing rule.
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase, astro-cloudflare]
version: 1
---

Rules for authoring Bytetalent feature templates. Per `bytetalent/docs/guide-template-development.md`. For context on how templates relate to bases and generated client repos, see the `pipeline-patterns` skill.

## Template anatomy

A template lives at the feature level. Every feature folder contains:

```
<featureKey>/
  feature.json         # feature contract — see required fields below
  design.pen           # Pencil design file (encrypted — access via MCP only)
  tokenBindings.json   # brand-kit → Pencil variable mappings
  README.md            # what this template does + integration notes
  bundles/
    <bundleId>/
      template.json    # bundle-level metadata
      README.md
      files/           # file actions (add/replace/merge-json)
```

## feature.json — required fields

```json
{
  "featureKey": "auth-email-otp",
  "displayName": "Email OTP Authentication",
  "description": "Sign-in and sign-up with one-time passwords via Clerk.",
  "supportedBundles": ["nextjs-clerk-supabase"],
  "category": "auth",
  "version": "1.0.0"
}
```

Required: `featureKey`, `displayName`, `description`, `supportedBundles`, `category`, `version`.

## Template categories

Valid values for `category`:

`auth` | `billing` | `marketing` | `communication` | `admin` | `data` | `infra` | `other`

Do not introduce new category values — the pipeline UI uses this list to group features.

## Bundle file actions

Each entry in `files/` declares one of three actions:

| Action       | When to use                                                                    |
| ------------ | ------------------------------------------------------------------------------ |
| `add`        | New file that does not exist in the base                                       |
| `replace`    | Full overwrite of an existing file                                             |
| `merge-json` | Deep-merge JSON keys into an existing file (e.g., `package.json` for new deps) |

Use `merge-json` for `package.json` additions so multiple templates can contribute dependencies without overwriting each other.

## The pairing rule

Every code template **must** have a matching design template for the same `featureKey` + `bundleId`. CI runs `node scripts/lint-pairing.mjs` and blocks merge on any unpaired template.

```bash
# Run locally before opening a PR
node scripts/lint-pairing.mjs
```

A code template with no matching `design.pen` will fail lint. A `design.pen` with no code counterpart will also fail.

## Token bindings

`tokenBindings.json` maps brand-kit fields to Pencil design variables, enabling one-click theme application when a client's brand kit is applied:

```json
{
  "bindings": [
    { "brandField": "primaryColor", "pencilVariable": "--color-primary" },
    { "brandField": "fontFamily", "pencilVariable": "--font-sans" }
  ]
}
```

An empty `{ "bindings": [] }` is valid for templates that don't use brand-kit variables.

## Semver rules

| Bump                              | When                                                |
| --------------------------------- | --------------------------------------------------- |
| `feature.json` bump               | Feature contract or `supportedBundles` list changes |
| `bundles/<id>/template.json` bump | Implementation files change without contract change |

Template versions are pinned by git tag/SHA in client repos. Never re-tag an existing version.

## Sparse matrix

The same `featureKey` does **not** imply the same code across bundles. Each `bundleId` has a fully distinct implementation. A `next.js` auth template and an `astro` auth template may share a featureKey but their `files/` contents are completely separate.

Do not try to share code between bundle implementations — maintain them independently.

## When to add a new template

Add a template only when one of these is true:

1. A real engagement explicitly requested this feature.
2. There is strong evidence (R1-1 sprint data) of a recurring pattern across multiple engagements.
3. It is foundational and obviously needed by nearly every project of a given type.

Do **not** add templates speculatively. Features needed by 80%+ of projects belong in the base, not as templates (see `pipeline-patterns` skill).
