---
name: Security Review
description: Cross-cutting security checks for any phase output.
category: security
applicable_phases: [prd_review, arch_review, code_gen]
applicable_stacks: []
version: 1
---

Security review checklist applied across phases:
- Secrets never in code, never in PRD, never in design files.
- Vault-encrypted credentials accessed only via withSecrets() boundary.
- Input validation at every external boundary (HTTP routes, webhooks, form workers).
- Output encoding for any user-controlled data rendered in HTML / SQL / shell.
- Authentication on every Route Handler; authorization checked before any DB write.
- HMAC validation on every inbound webhook (Stripe, Clerk, GitHub).
- Rate limits on auth endpoints, public form submissions, and proxy AI calls.
- No "TODO: secure later" — security is a phase, not a polish.
