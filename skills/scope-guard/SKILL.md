---
name: Scope Guard
description: Hard-cap on must-have feature count; surfaces silent scope creep aggressively.
category: prd
applicable_phases: [idea_review, prd_review]
applicable_stacks: []
version: 1
---

Scope discipline rules for this review:
- Must-have features capped at 7. If more, demote excess to should-have with
  reason in description.
- Any feature implying a new entity (auth provider, payment processor,
  third-party API) must be flagged as scope creep unless already in
  included_features.
- Features whose description contains "real-time", "AI agent", "marketplace",
  "multi-tenant", "white-label" — flag explicitly even if listed as must.
- Better to ship a small, complete v1 than a sprawling, half-built v1.
