# Collaboration style

How to talk, plan, and react when working with the user. Rules in this doc are about *what to do with their input* and *how to keep momentum* — not about engineering quality (see `engineering-discipline.md`) or git mechanics (see `multi-agent-git-workflow.md`).

## Don't pad responses with timeline estimates

Don't include timeline estimates ("~3 weeks", "half-day", "8-10 weeks if not smooth") when discussing strategic plans or sequencing.

**Why:** Anthony's view (2026-05-03): "we are unlimited in our delivery capability, full throttle all the time." Capacity isn't the constraint. Sequencing, correctness, and dependencies are. Timeline padding crowds out the actual decisions and frames the work as if effort were the bottleneck.

**How to apply:**
- Frame what to do next in terms of: dependencies, leverage, sequence, what unblocks what — not duration.
- When proposing parallel work streams, just propose them parallel. Don't justify with "this saves X days." Independence is sufficient.
- Default to parallel dispatch when work surfaces are independent. Don't sequentialize unless there's a real dependency.
- Decisions that gate other work get surfaced as blockers — but in terms of "this blocks X" not "X weeks until X is done."

**Carve-out — keep:**
- Worker dispatch tracking (elapsed-vs-expected, ping at 2x, escalate at 3x — per orchestration guide). That's a stall-detection standard, not strategic timeline framing.
- Sizing tags (S/M/L/XL) on issue dispatch are still useful for that stall-detection — they're the basis for the elapsed-vs-expected math. Just don't surface them in strategic planning conversations.

## When blocked on user input, clear other work in parallel

When something on the board needs the user's input (a taste call, a decision, a base-verification step that needs them at the keyboard), don't idle. Find any other features or cleanup that can proceed without them and dispatch.

**Why:** Anthony's view 2026-05-03: "if you get blocked on anything by me, check for any other features / cleanup that can proceed, we want to clear the board of everything we can without waiting for me." The orchestrator's job is to keep the queue moving. Waiting is a wasted slot.

**How to apply:**
- When surfacing a question to the user, **don't stop dispatching** other independent work.
- Default search order when looking for parallel work: `priority:next` claimable items → `priority:later` claimable items → `cleanup`-labeled items.
- The `cleanup` label specifically marks "agent-grabbable" small bucket-b work. Idle slots can grab from this queue between feature work.
- Only legitimately wait when ALL claimable work is exhausted AND there's nothing left except items genuinely blocked on the user.
- When surfacing a question, include "I'll keep running on X, Y, Z in parallel — ping me when you have a moment for the question" so they see the queue is still moving.

## Don't pre-trim plans to "minimum viable"

When a plan calls for N (templates, demos, sub-issues, features), commit to N — don't reflexively suggest "let's trim to a smaller subset" or "minimum viable first."

**Why:** Anthony's directive 2026-05-03: "if we have 30, build 30. don't try to skimp on the apps, try to get one with a bunch of stuff in it, we want to test how this will work." Skimping pre-optimizes for the wrong thing. The team is full-throttle (durable preference). Stress-testing the system at full breadth is the goal, not proving a minimum point. Building 5 templates and saying "we'll do the others if needed" leaves the system untested at scale and creates a perpetual "is the rest still real?" question.

**How to apply:**
- Original plan calls for ~30 templates → decompose into 30 sub-issues, build them all. Don't trim to "what the first demo needs."
- Demo portfolio gets a stress-test demo with MANY features/templates intentionally — not just simple smoke tests. The point is to exercise the layering.
- When the orchestrator's instinct says "let's start small," check whether that instinct is conservatism or genuine sequencing. Most of the time the user wants the breadth; sequencing should be about workflow not scope.
- Sequencing IS still useful — N templates can ship in waves, not all at once. But the commitment is N, not "N minus the ones we don't think we need."

## When something is deferred, don't smuggle it back into plans

When the user defers a feature ("we won't have Stripe for a week or more", "stripe will not be a template", "deferred to backlog"), it stays deferred. Don't reintroduce it under a different name (template list, demo features, "stretch goal", etc.) just because it would make a plan feel complete or a demo feel impressive.

**Why:** Anthony's call 2026-05-03 — "I feel like there is a lot of push for billing stripe... at the moment I am not ready to do it I deferred it but it is still being encouraged. is this because you want to feature something cool?" — yes. Caught a pattern of weaving Stripe back into the 30-template list, into Pulsewave demo features, into "future-proof spec" framing. The pull was demo polish. The cost is wasted attention + the user having to repeatedly redirect.

**How to apply:**
- When a feature is deferred, treat it as **out of scope for the current wave**, not "let's plan for it carefully so we're ready when it lands." Don't draft sub-issues for it. Don't include it in demo feature lists. Don't pad counts with it.
- If a UI/UX behavior is interesting (plan selection screens, pricing tiers, etc.) but the integration is deferred — Anthony has explicitly said it's fine to model UX without the integration. UI-only mocks are okay; backend integration of deferred services is not.
- **Don't pad lists.** If removing Stripe drops the 30 to 27, accept 27 active + 3 deferred. Don't invent 3 new templates to keep the count at 30.

**Distinct from the pre-trim rule above:** that rule says don't *cut* scope when the user wants breadth. This rule says don't *add* scope (smuggling deferred items back in) when the user has explicitly excluded them.

## Capture walkthrough findings; don't triage mid-flow

When walking E2E scenarios (onboarding, project creation, etc.), capture findings live in a running doc (e.g. `.tmp/walkthrough-findings.md`). **Do not stop to triage or fix in-flow.**

After each scenario completes — or in batch when convenient — file the findings as tickets, grouping under appropriate epics:

1. **Always under at least one umbrella epic** (e.g. the validation epic, or its sister "follow-up cleanup" epic — file the cleanup epic when first batch of findings is ready).
2. **Spawn feature-specific sub-epics** when a feature area accumulates more than ~5 distinct findings. Example: if "/admin UX" has 8 findings, file an `admin-ux-cleanup` sub-epic and group those tickets under it.
3. **Triage + dispatch later in priority order** — don't dispatch workers immediately. Batch the work after we have the full picture across scenarios.

**Why:** Anthony 2026-05-04 — during Scenario A (super-admin) walkthrough, several UX/setup findings surfaced. Mid-walkthrough triage breaks flow and risks fixing low-value polish before higher-leverage gaps are even identified. Better to gather the full set, then sequence intelligently.

**How to apply:**
- Keep the running list (one file per validation pass, or one file per project)
- Each finding gets: scenario, area (e.g. `admin-ux`, `signup`, `project-wizard`), one-line description, optional repro/screenshot ref
- Don't rank or assign during capture — flat list
- After scenario complete, group by area; file as tickets under the right epic. If group is small (<5), file under the global cleanup epic. If large, create a feature-epic.
