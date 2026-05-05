# Base Development

How a Bytetalent base repo is structured, modified, and versioned. Stack-specific deviations live in each base's own `docs/guide-stack.md` overlay; this guide covers the cross-stack contract.

Rules for anatomy, the `bytetalent-base.json` manifest, the `docs/` overlay shape, versioning semver, and the recipes for adding or modifying a base live in the **`architecture/base-development` skill**. The `lessons.md` cadence (when to log a lesson vs. just commit) is in the **`process/lessons-learned-discipline` skill**.

This guide covers the *why* — the narrative behind the model.

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

## 2. Why the manifest matters

The `bytetalent-base.json` at the root is the pipeline's handshake with the base. When the pipeline scaffolds a generated client repo:

1. Reads the project's chosen `bundleId` from setup wizard
2. Finds the base whose `bundleId` matches
3. Pins the client to that base at the current version (`baseVersion: "1.2.0"`, `baseSha: "abc123"`)
4. Records this in the **client repo's** `bytetalent-base.json`

So if the base evolves to v1.3.0, the existing client at v1.2.0 doesn't auto-receive changes (per Strategy A snapshot, decision #7). Either the consultant manually back-ports, or future Strategy B tooling (deferred per plan) handles diff-aware upgrades.

The required manifest fields and full schema are in the `architecture/base-development` skill.

---

## 3. The `design/` slot

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

## Related

- `architecture/base-development` skill — anatomy, manifest schema, versioning rules, adding/modifying recipes
- `process/lessons-learned-discipline` skill — when and how to write a lessons entry
- [`guide-template-development.md`](guide-template-development.md) — how templates layer on top of bases
- [`guide-pencil.md`](guide-pencil.md) — Pencil MCP workflow for `.pen` design files
- [`guide-deploy.md`](guide-deploy.md) — universal deploy principles (stack-specifics in each base's overlay)
