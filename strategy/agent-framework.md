# Agent framework — the meta-product behind app-flow.ai

While building app-flow.ai (the consultant-facing platform), Bytetalent has accumulated a separate body of operational infrastructure for working WITH AI agents (Claude Code primarily). It's distinct from the product itself — a reusable framework for any team running multi-agent / agent-orchestrated development.

This doc captures the strategic frame: what the framework is, how it ships, how it improves, and how it monetizes.

## Four delivery channels

The framework isn't a single deliverable. It operates across four channels in concert.

### Channel 1 — app-flow.ai (the platform)

The consultant-facing product. The SaaS revenue line; consultants pay to generate + manage projects. Stays the primary commercial offering.

### Channel 2 — Reference implementation / shining model

app-flow.ai itself is the proof: built using the framework it teaches. Every architectural decision, skill, prompt, dispatch protocol, review step we've made is the demo. The consulting angle: "use your pilot project as an app-flow project, produce working software, train your team on the framework while you're at it." Marketing asset + credibility.

### Channel 3 — Extractable SDLC framework

The methodology, packaged as something teams can adopt to do agentic delivery WITHOUT app-flow.ai if they want. Skills, prompts, base repos, templates, review checklists, dispatch protocols, memory hygiene, branch lifecycle. **This is the key differentiator.** Without channel 3, channel 1 is just another code-gen SaaS; with it, it's a delivery methodology with peace of mind ("I got this, I have tools, I have a support system") and clean handoff to client teams.

### Channel 4 — Per-generated-app inheritance

Every app app-flow.ai generates ships with the framework baked in: setup guides, infrastructure helpers, preloaded skills + prompts, CLAUDE.md template, possibly a VS Code extension. Every generated app comes ready to keep developing in the same agentic style with the same standards. The framework doesn't get extracted from app-flow once — it propagates into every project app-flow produces.

### Channel value-prop

| Channel | What it is | Why it matters |
|---|---|---|
| 1 | SaaS platform | Revenue line; recurring subscriptions |
| 2 | Reference implementation | Marketing credibility; consulting hook |
| 3 | Extractable framework | Differentiation; extensibility; handoff |
| 4 | Per-app inheritance | Sticky outcome; buyer keeps developing the same way |

When auditing a framework artifact, ask not just "is this generic vs Bytetalent-specific" but "which channel(s) does this serve, and how does it adapt to each?" Many artifacts are useful across multiple channels with adaptation.

## The continuous-improvement operating loop

The framework's value isn't its current state — it's the **rate at which it improves**. Every interaction between a human (or orchestrator) and a generated app is a potential framework upgrade. We operate against a six-sigma DMAIC loop:

### Define
Codify rules / skills / seeds when patterns first surface. New friction → new artifact: a skill SKILL.md, a memory seed, a prompt revision, a dispatch-brief template, a process doc. The artifact captures *how to handle this pattern* so the next encounter doesn't relitigate it.

### Measure
Backlog grooming, walkthrough findings, board hygiene observations, branch-and-worktree audits, GraphQL rate-limit checks. The board (and its supporting artifacts — `walkthrough-findings.md`, audit reports, ticket queues) is the instrument panel. Every audit measures the framework's coverage against actual experience.

### Analyze
Triage observed friction into categories:
- **Framework gaps** — the artifact doesn't exist, or exists but doesn't capture this case
- **Product bugs** — code-level fixes; not framework material
- **Scope decisions** — strategic boundaries to surface to the human
- **One-off accidents** — won't recur; don't codify

Only the first category produces framework upgrades. Discipline matters: not everything observed is worth codifying.

### Improve
Update skills, seeds, prompts, dispatch protocols, agent-framework artifacts. PR-track these changes — peer review acts as the durability check. The active memory dir promotes durable entries to seeds (see `bytetalent/docs/agent-framework/memory-seeds/README.md`).

### Control
Orchestration runs against the upgraded artifacts; the next wave inherits the improvement. Channel 4 inheritance means the upgrade lands in *every* generated app downstream. Channel 3 framework consumers pull the latest.

### Why this loop matters

Backlog grooming is no longer just product work. It's the queue of *framework improvements observed in flight*. Ticket management becomes the orchestration layer because it's where the improvements get sequenced into the next dispatch. The human + orchestrator pairing is highly productive specifically because the loop is short and codified — friction surfaces, gets named, gets fixed at the framework level, doesn't recur.

This is what makes the framework a sellable methodology rather than a pile of utilities. Buyers aren't paying for the current state of skills + templates; they're paying for the rate-of-improvement and the discipline that produces it.

## Inheritance model

Framework artifacts compose by layer. A generated app's effective configuration is reconstituted by stacking applicable layers:

```
universal       → bytetalent/docs/{skills,agent-framework,strategy}/
base            → bytetalent/<base-repo>/
template        → bytetalent/templates/<template>/
product         → <product-repo>/

memory seeds:    universal ∪ base ∪ templates_in_use ∪ product
skills:          universal ∪ base ∪ product (at minimum; templates may add)
prompts:         universal phase prompts; product extends with audience/brand
CLAUDE.md:       template ships with every generated app, loaded into the agent's context
GitHub Actions:  per-repo, but patterns shared via the framework
```

For a non-generated project, the chain is `universal + product`. For a generated client app, the full chain applies.

The seed model documented at `bytetalent/docs/agent-framework/memory-seeds/README.md` is one concrete instance of inheritance; the same pattern extends to skills, prompts, and process docs.

## Packaging / monetization questions

These get answered when the artifact set has matured. Open questions, captured here so the eventual review has continuity:

### What's free vs licensed?

Candidate splits:
- **Free / open-source core** — backlog-grooming skill, branch-hygiene skill, board-numbering, generic process skills, base CLAUDE.md template, dispatch protocol guide. Drives adoption + credibility + top-of-funnel.
- **Premium / paid** — advanced/proprietary templates (Stripe billing, advanced auth flows, complex data models), specialized review skills (security, compliance, performance), VS Code extension premium features, white-label, training programs.
- **Bundled with app-flow.ai SaaS** — the connection between platform and generated app + the per-app inheritance bundle (channel 4). Customers don't pay separately for framework-in-their-app; it comes with their subscription.

### Who owns the framework artifacts in the marketplace?

- **First-party** (Bytetalent maintains) — the core skills + templates Bytetalent authors
- **Third-party** — marketplace where consultants/agencies publish their own templates + skills, with revenue share
- **Mixed** — a "verified" tier maintained by Bytetalent + community contributions

### Service-tier overlay

- Self-serve (subscription, no human support)
- Pro (subscription + ticket support + skill assistance)
- Enterprise (subscription + dedicated success manager + custom skill authoring + training)
- Consulting (full-service "we run the project for you using the framework" — channel 2 in commercial form)

### Open-source attribution

If channel 3 is open-source, a consultant who adopts it might use it for client work without ever paying Bytetalent. That's fine for top-of-funnel — the question is whether at some scale of adoption that path needs gating (license that requires app-flow.ai for commercial use beyond N projects, or a "Pro" license for branding rights).

### What's the pull mechanism that brings open-source adopters back to paid?

Probably one of:
- Hosting/runtime (their generated apps run on Bytetalent's infra unless self-hosted)
- Premium templates / skills marketplace
- Training / certification
- Direct support contracts
- The platform itself (channel 1) for consultants who want managed orchestration

### Comparable models

- **Vercel** — free tier + paid tiers; the hosting is the lock-in but the open-source Next.js drives adoption
- **HashiCorp Terraform** — open-core + enterprise; community uses OSS, large orgs pay for advanced governance
- **Sentry/PostHog** — open-source self-hosted + cloud paid; cloud is the easy path
- **Stripe** — free SDK / framework / docs everywhere, take-rate on transactions (the platform is the value capture)

The Bytetalent shape probably looks like Stripe + HashiCorp hybrid: free framework + free reference implementations + paid platform + paid premium templates/skills + enterprise support tier.

## When to make the packaging decisions

Not now. The decisions get made when:
- ~3+ real customers running on the platform (data on what they use vs ignore)
- Channel 3 framework has been extracted as a real artifact
- Channel 4 inheritance has shipped through deploy automation work

Until then: keep building, keep capturing patterns as skills, keep observing what's reusable. The packaging question answers itself once the artifact set is clear.

## Candidate framework artifacts

Already exist or are emerging — when we do the consolidated framework review, audit each for inclusion / placement / channel:

- **CLAUDE.md** at repo root — project conventions, deviation notes, git workflow, format gates
- **`bytetalent/docs/skills/`** — universal skills library (board-numbering, github-api-efficiency, backlog-grooming, branch-and-worktree-hygiene, etc.)
- **`bytetalent/docs/agent-framework/memory-seeds/`** — coalesced operating-instruction docs (this dir was just established)
- **`bytetalent/docs/strategy/`** — channel-level + framework-level thinking (this doc is the seed)
- **Phase prompts** — PRD / brand-kit / design / arch / code-gen prompts, with audience and brand-tone variants
- **Agent dispatch patterns** — implementer agent type, worker brief structure, isolation:worktree, run_in_background, unique branch names
- **Memory system** — typed memories (feedback / project / reference / user) with seed/active overlay model
- **GitHub Actions for process** — close-epic-when-children-done, journal monotonicity check, husky cross-platform fix, planned: branch-hygiene-audit, scheduled cleanup
- **Worker brief templates** — the dispatch-prompt structure (Why / Working setup / Scope / Done when / PR / Reporting)
- **Wave-review protocol** — epic-close + 10-PR-threshold triggers
- **Findings capture pattern** — walkthrough-findings.md doc + group-then-file approach

## The reuse question — how artifacts sort

When auditing each artifact:

1. **Generic / portable** — applies to any agent-orchestrated team. Belongs in the framework's universal layer.
2. **Bytetalent-flavored but adaptable** — uses our specific tooling (GitHub Projects, Vercel, Clerk) but the *pattern* is portable. Needs a templating pass.
3. **App-flow-specific** — only relevant to building this product. Stays in `app-flow-web/`.

The framework's universal layer ends up being category 1 + adaptable templates from category 2. Category 3 informs the reference implementation (channel 2) but doesn't ship with the framework itself.

## Capture as we go (until the formal review)

- Every time we evolve a new pattern (skill, memory, action, brief structure), note it explicitly so the future review has a clean inventory.
- When filing tickets that produce framework artifacts (skills, actions, dispatch templates), use a consistent label like `framework-candidate` so they can be filtered later.
- Don't pre-extract — premature extraction creates parallel maintenance burden. Let the patterns mature in-place; extract during the review when value is clear.
- Apply DMAIC every interaction: notice friction, name it, decide whether it's framework material, codify if so.

## Connection to in-flight work

- **#545 deploy + setup automation epic** — channel 4 (per-generated-app artifacts: setup runbooks, env-var manifests, post-install scripts).
- **#548 process improvement epic** — channel 3 deliverables (skills, scheduled hygiene, release versioning, task grouping conventions). Track A (skills authoring) is the most direct framework contribution.
- **#549 generated-app skills inheritance** — explicitly the channel 4 question of how generated apps get the framework's skills.
- **#586 docs reorg** — surfaces strategy/, restructures product docs, makes the framework artifacts visible to a new contributor.
