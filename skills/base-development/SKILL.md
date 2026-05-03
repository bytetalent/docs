---
name: Base Development
description: How to structure, version, and evolve a Bytetalent base app — manifest keys, semver rules, CI smoke test, lessons log, and new-base recipe.
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase, astro-cloudflare]
version: 1
---

Rules for authoring and maintaining Bytetalent base apps. Per `bytetalent/docs/guide-base-development.md`. For how bases relate to templates and generated client repos, see the `pipeline-patterns` skill.

## Base is a real app

A base is not a skeleton or a starter. It is a **real running application** deployed to a stable dogfood URL. Every change ships to that URL before any generated client repo sees it. If a change is not safe to ship to the dogfood app, it is not safe to ship to client repos.

## Base anatomy

Every base must contain:

```
README.md
package.json
bytetalent-base.json      # manifest — see below
.env.example              # placeholder values only, never real secrets
.github/workflows/ci.yml  # smoke job: typecheck + build
src/                      # application source
docs/
  lessons.md              # real-engagement learnings
design/
```

## bytetalent-base.json — required fields

```json
{
  "name": "nextjs-clerk-supabase",
  "version": "1.0.0",
  "stack": {
    "framework": "nextjs",
    "host": "vercel"
  },
  "bundleId": "nextjs-clerk-supabase",
  "envVars": {
    "required": ["DATABASE_URL", "CLERK_SECRET_KEY", "NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY"]
  },
  "dogfoodUrl": "https://base-nextjs.bytetalent.dev"
}
```

Required keys: `name`, `version`, `stack.framework`, `stack.host`, `bundleId`, `envVars.required`, `dogfoodUrl`. Missing keys block the pipeline wizard from listing the base.

## Semver rules

| Bump    | When                                                                          |
| ------- | ----------------------------------------------------------------------------- |
| `patch` | Bug fix; no API or structure change                                           |
| `minor` | Additive: new pre-wired feature, new env var with a default value             |
| `major` | Breaking: removed/renamed/restructured — generated clients need a manual port |

Update `bytetalent-base.json` `version` on every release. Tag the commit. Never re-tag an existing version — create a new one.

```bash
# Good
git tag base-nextjs-clerk-supabase@1.2.0

# Bad — re-tagging breaks pinning in generated client repos
git tag -f base-nextjs-clerk-supabase@1.1.0
```

## CI smoke job

Every base must have a CI workflow that runs on every PR:

1. `tsc --noEmit` (typecheck)
2. `next build` / equivalent build command

No PR merges before both pass. This is the minimum bar; unit tests are encouraged.

## Lessons log

Add an entry to `docs/lessons.md` when:

- A real engagement surfaces a gap in the base
- A pattern is proved or disproved in production
- An assumption baked into the base turns out to be wrong

Skip lessons log entries for routine bug fixes. The log is for signal that should shape future base revisions, not a changelog.

## New base recipe

1. Identify a similar existing base and read its `docs/lessons.md` first.
2. Write `bytetalent-base.json` before writing any source code.
3. Write `.env.example` — every required var documented with a comment.
4. Sequence the implementation after the closest existing base.
5. Ship to the dogfood URL before registering the base in the pipeline wizard.

```json
// bytetalent-base.json written first — defines the contract
{
  "name": "astro-cloudflare",
  "version": "0.1.0",
  "stack": { "framework": "astro", "host": "cloudflare" },
  "bundleId": "astro-cloudflare",
  "envVars": { "required": ["CLERK_SECRET_KEY"] },
  "dogfoodUrl": "https://base-astro.bytetalent.dev"
}
```
