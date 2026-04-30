# Base Development

How a Bytetalent base repo is structured, modified, and versioned. Stack-specific deviations live in each base's own `docs/guide-stack.md` overlay; this guide covers the cross-stack contract.

---

## 1. What is a base?

A **base** is a real, dogfoodable, deployed-at-a-stable-URL application that the Bytetalent pipeline reads at code-gen time. Every generated client repo starts as a copy of a base + opt-in templates layered on top.

Bases serve four jobs simultaneously:

1. **Scaffold input** — the pipeline pulls files from this repo at code-gen time
2. **Pattern-proving vehicle** — strategic decision #3: bases are real running apps, not skeletons
3. **Reference implementation** — when a developer wants to know "how do we do X in stack Y?", the base is the answer
4. **Regression catcher** — every change ships to the dogfood URL; failures show up before generated clients see them

**Naming:** `bytetalent-{stack}-{host}-base`. Examples: `bytetalent-astro-cloudflare-base`, `bytetalent-nextjs-clerk-supabase-base`, `bytetalent-expo-clerk-supabase-base`. Local folder names may add a `bytetalent-` prefix to match GitHub repo names; this is convention, not contract.

---

## 2. Anatomy

Every base follows this top-level layout:

```
{base-repo}/
  README.md                  # purpose + how the pipeline consumes the base
  package.json               # deps + scripts
  bytetalent-base.json       # version manifest (see §3)
  .env.example               # required + optional env vars
  .gitignore
  .github/workflows/ci.yml   # smoke test (typecheck + build) on every PR
  src/                       # source code
  public/                    # static assets (or stack equivalent — Astro public/, etc.)
  docs/
    README.md                # overlay map referencing bytetalent/docs
    guide-stack.md           # stack-specific patterns
    guide-deploy.md          # stack-specific deploy commands + env-var checklist
    lessons.md               # pattern-proving findings (see §6)
  design/
    .gitkeep                 # slot for the paired .pen baseline
    {base-name}.pen          # the actual design file (added in a Pencil session)
```

Stack-specific additions go inside `src/` or alongside it (e.g., `astro.config.mjs` for Astro, `next.config.mjs` for Next.js, `wrangler.toml` for Cloudflare deploys, `drizzle.config.ts` for ORM-using bases).

**The base must build** with `bun install && {build command}` against a clean checkout. CI (`smoke` job) verifies this.

---

## 3. `bytetalent-base.json` contract

Every base ships a `bytetalent-base.json` at the repo root. The pipeline reads this to understand what kind of base it is and what version is being applied.

```json
{
  "$schema": "https://bytetalent.com/schema/base-manifest.json",
  "name": "bytetalent-nextjs-clerk-supabase-base",
  "version": "1.0.0",
  "description": "Brief one-liner shown in the project setup wizard",
  "stack": {
    "framework": "nextjs",
    "frameworkVersion": "^16.0.0",
    "host": "vercel",
    "language": "typescript",
    "styling": "tailwindcss",
    "stylingVersion": "^4.0.0",
    "database": "supabase-postgres",
    "orm": "drizzle",
    "auth": "clerk",
    "runtime": "vercel-edge"
  },
  "bundleId": "nextjs-clerk-supabase",
  "supportedTemplateBundles": ["nextjs-clerk-supabase"],
  "preWired": [
    "api-error",
    "session-resolver",
    "drizzle-singleton",
    "..."
  ],
  "ships": {
    "auth": ["sign-in", "sign-up", "..."],
    "dashboard": ["layout-shell", "sidebar", "topbar", "..."],
    "marketing": ["site-nav", "hero-section", "..."]
  },
  "envVars": {
    "required": ["..."],
    "requiredForBilling": ["..."],
    "requiredForProd": ["..."]
  },
  "dogfoodUrl": "https://base.bytetalent.com",
  "docsRepo": "https://github.com/bytetalent/docs",
  "lastUpdated": "2026-04-30"
}
```

### Required keys

- **`name`** — repo name (`bytetalent-{stack}-{host}-base`)
- **`version`** — semver. Bump on any base change, tag the commit accordingly
- **`stack.framework`** + **`stack.host`** — drive bundle resolution
- **`bundleId`** — the key templates match against in their `supportedBundles`
- **`envVars.required`** — what must be set for the base to even boot
- **`dogfoodUrl`** — the deployed URL (or `null` if not yet deployed)

### Why this matters

When the pipeline scaffolds a generated client repo:

1. Reads the project's chosen `bundleId` from setup wizard
2. Finds the base whose `bundleId` matches
3. Pins the client to that base at the current version (`baseVersion: "1.2.0"`, `baseSha: "abc123"`)
4. Records this in the **client repo's** `bytetalent-base.json`

So if the base evolves to v1.3.0, the existing client at v1.2.0 doesn't auto-receive changes (per Strategy A snapshot, decision #7). Either the consultant manually back-ports, or future Strategy B tooling (deferred per plan) handles diff-aware upgrades.

---

## 4. Per-stack `docs/` overlay

Every base ships a thin `docs/` folder that:

1. **References `bytetalent/docs`** for cross-stack standards (linked at the top of the overlay's index)
2. **Adds stack-specifics** — file conventions for that framework, deployment commands, build setup
3. **Captures pattern-proving findings** as the base evolves (see §6)

### Required files

| File | Required? | Purpose |
|---|---|---|
| `docs/README.md` | ✅ | Overlay map — links to bytetalent/docs guides; lists this base's stack-specific deviations |
| `docs/guide-stack.md` | ✅ | Stack-specific patterns: file conventions, framework idioms, integration details |
| `docs/guide-deploy.md` | ✅ | Stack-specific deploy commands + env-var checklist + rollback procedure |
| `docs/lessons.md` | ✅ | Findings captured as the base ships and is dogfooded |

### `docs/README.md` shape

Always opens with:

```markdown
# {Stack Name} Base — Docs Overlay

This is the **{stack}-specific overlay**. It complements the universal
[`bytetalent/docs`](https://github.com/bytetalent/docs) repo with patterns
and details that are only relevant when building on {stack}.

## How to read this overlay

Always start with [`bytetalent/docs`](...) for cross-stack standards.
Come here only for {stack}-specific details.
```

Followed by a table mapping each concern to either `bytetalent/docs/guide-X.md` or `guide-stack.md` / `guide-deploy.md`.

---

## 5. `design/` slot

Every base reserves a `design/` folder for the paired `.pen` baseline. The Pencil file accumulates designs for:

- Marketing surface (signed-out home sections)
- Auth pages (where applicable)
- Dashboard shell (where applicable)
- Reference patterns (data tables, stat cards, detail-page shells)

**Workflow** (per [`bytetalent/docs/guide-pencil.md`](guide-pencil.md)):

1. `design/.gitkeep` until the first design pass
2. A focused Pencil session creates `design/{base-name}.pen`
3. Subsequent sessions accumulate sections within the same file
4. The .pen file is the source of truth for the base's visual language

**Important:** `.pen` files are encrypted and only editable through the Pencil app. Don't try to merge them — only one person edits a `.pen` file at a time.

---

## 6. `lessons.md` cadence

Every base ends with `docs/lessons.md` — a running log of findings from the dogfood phase. Each lesson should:

```markdown
## YYYY-MM-DD — short title

What we tried, what we found, what changed.
- For the base: vN.M.K — what changed in this repo
- For other bases: what to apply / avoid
- For the pipeline: signal type (UX punch list / base coverage gap /
  convention violation / etc.)
```

**When to add a lesson:**

- A real engagement surfaces a gap that the base should cover
- A pattern proves out (or fails) under real use
- An assumption baked into the base turns out wrong
- A version bump introduces a meaningful change

**When NOT to add a lesson:**

- Routine bug fixes (commit message + maybe a code comment is sufficient)
- Refactors that don't change behavior
- Style cleanups

The signal is: *"would the next person building a base benefit from knowing this?"* If yes, lesson. If no, just commit.

---

## 7. Versioning a base

Bases follow semver:

- **Patch (1.0.0 → 1.0.1):** bug fix, no API change. Generated clients can adopt with no work.
- **Minor (1.0.0 → 1.1.0):** additive — new pre-wired feature, new shared component, new env var with a default. Generated clients can adopt by re-applying scaffold over the existing one.
- **Major (1.0.0 → 2.0.0):** breaking — removed file, changed exported symbol, restructured folders. Generated clients require manual back-port or Strategy B tooling (deferred).

**On every release:**

1. Update `bytetalent-base.json`'s `version` field
2. Update `lastUpdated` to today's date
3. If breaking, update `lessons.md` with the migration guidance
4. Tag the commit: `git tag v1.1.0 && git push --tags`
5. The pipeline's pin uses tags, so projects scaffolded against v1.0.0 keep getting v1.0.0 even after v1.1.0 ships

**Never re-tag** an existing version — that breaks pin contracts in already-generated client repos.

---

## 8. Adding a new base for a new stack

When the pipeline needs to support a new stack (e.g., ASP.NET Core, Expo, Vue + Nuxt), create a new base repo. The recipe:

1. **Sequence after a similar stack base lands.** Strategic decision #5: each new base reads `lessons.md` from the most-similar existing base before starting. The Astro base's lessons informed the Next.js base; the Next.js base's lessons will inform the Expo base.

2. **Pick names:**
   - GitHub: `bytetalent/{stack}-{host}-base`
   - `bundleId`: `{stack}-{host}-{auth?}-{db?}` — should be unique across all bases

3. **Create the empty repo:**
   - GitHub: empty repo with `main` as default
   - Locally: `git clone` (or `git init` then add the remote)

4. **Scaffold the directory layout** per §2

5. **Write `bytetalent-base.json` first** — without it, the pipeline can't even recognize the base. v1.0.0, all required keys filled in

6. **Build the working scaffold:**
   - The minimum app that runs (`bun install && bun run dev` works)
   - Pre-wired infra (depending on stack: api-error, session resolver, ORM singleton, middleware, env validation)
   - Auth flows (where applicable)
   - Dashboard shell (where applicable)
   - Marketing surface — copy section components from the Astro base, port them to the new stack

7. **Write the `docs/` overlay** per §4

8. **Set up CI** — `.github/workflows/ci.yml` runs `bun install` + typecheck + lint + build on every PR

9. **Deploy to a dogfood URL** — `{stack}.bytetalent.com` or similar. Strategic decision #3: bases are real running apps. The first deploy might be a "hello world"; that's OK as long as it works.

10. **Open `lessons.md`** with the v1.0.0 intentional-vs-deferred decisions (what's in the base by design vs. what's deferred to templates or future versions)

11. **Add the base to the pipeline's wizard** — the consultant should be able to pick this stack when creating a new project (this lives in `bt-ai-web`, not in the base itself)

---

## 9. Modifying an existing base

For non-breaking changes (typical):

1. Open a PR with the change
2. CI (smoke test) must pass
3. Update `bytetalent-base.json` version
4. Update `lessons.md` if the change is a meaningful pattern shift
5. Merge to main
6. Tag + push the new version

For breaking changes:

1. Same as above, plus:
2. Bump major version
3. `lessons.md` documents the migration path
4. **Coordinate** with anyone running active generated client repos — they may need back-ports

---

## Related

- [`guide-template-development.md`](guide-template-development.md) — how templates layer on top of bases
- [`guide-pencil.md`](guide-pencil.md) — Pencil MCP workflow for `.pen` design files
- [`guide-deploy.md`](guide-deploy.md) — universal deploy principles (stack-specifics in each base's overlay)
