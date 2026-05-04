# Glossary

Canonical definitions for terms used across Bytetalent docs, code, and prompts.

---

## Org types

### agency

An account with `accountType = "agency"`. The primary pipeline user persona — a consultant, product developer, or small agency that takes client ideas through the app-flow.ai pipeline to produce delivered software.

An agency account owns its credentials (Anthropic key, GitHub App install, etc.), manages multiple client accounts, and operates every phase of the pipeline on behalf of those clients. Multi-seat agency accounts have a team of members (see: `owner`, `consultant`, `viewer`).

### client

An account with `accountType = "client"`. A client company served by an agency. Client users have a limited view: they can see only their own projects and deliverables, approve or request changes on published deliverables, and access their portal. They cannot create projects, see other clients, or access any agency-level pipeline features.

Client accounts are created when an agency sends a client invitation and the invitation is redeemed.

### platform

An account with `accountType = "platform"`. Bytetalent's own super-admin org. Exactly one instance should exist in production. Used for data-model queries and may gate platform-only features. Distinct from the `isSuperAdmin()` check (see below).

---

## Member roles

Roles are stored in `account_memberships.role`. Each org type has a valid role set enforced by `isValidRoleForAccountType()` in `src/lib/auth/roles.ts`.

### owner

Valid for: agency, client, platform.

The account creator or designated primary admin. For an agency, `owner` holds billing authority and full admin access. For a client, `owner` is the primary contact. For platform, `owner` is the Bytetalent founder or top admin.

### consultant

Valid for: agency.

The person (or team member) who does the pipeline work — creating projects, running phases, authoring prompts, reviewing AI output, and pushing code. Default role for invited agency members.

### product-owner

Valid for: client.

The client stakeholder who decides product scope and approves deliverables (PRD, design, architecture). Not a role inside the agency — this is the client-side decision maker.

### reviewer

Valid for: client.

A client user authorized to approve specific phases (e.g., design-only approval delegated to a design lead). More limited than `product-owner`.

### payor

Valid for: client.

The person on the client side who holds the billing relationship. Often the same person as `owner`; sometimes a separate finance contact.

### stakeholder

Valid for: client.

A client user with read-only access to deliverables — executives, advisors, or anyone who needs visibility but not approval authority.

### staff

Valid for: platform.

A Bytetalent employee with platform-wide visibility. Can see across all agency and client accounts for support and monitoring purposes.

### viewer

Valid for: agency, client, platform.

Read-only access within the org context. For agencies: partners or auditors who can observe but not modify projects. For clients: additional read-only contacts. For platform: read-only platform observers.

---

## Pipeline concepts

### pipeline

The five-phase AI-powered workflow in app-flow.ai: idea capture → PRD → brand kit → design → architecture → code generation. Each phase produces an approved deliverable. Nothing moves to the next phase without the consultant's explicit approval.

### consultant (role vs. noun)

As a role: the `consultant` membership role in an agency account — the person doing the pipeline work.

As a noun in product copy: may refer generically to anyone using the app as a pipeline operator, regardless of whether they hold the exact `consultant` role (e.g., an `owner` may also do consultant work). Context determines which meaning applies.

### phase

One step in the pipeline: `idea`, `prd`, `brand_kit`, `design`, `architecture`, or `code`. Each phase has a `project_phases` row tracking its status and working state (`data` JSONB column). On approval, a `project_deliverables` snapshot is created.

### deliverable

An approved snapshot of a phase's output stored in `project_deliverables.content`. Immutable after approval — edits to the source (e.g., brand kit) do not retroactively change a deliverable.

### skill

A reusable, curated chunk of expert context that is injected into the AI's system prompt for a given phase. Skills encode the consultant's standards: naming conventions, error handling patterns, accessibility requirements, security patterns. Stored in the `skills` table; active skills for a project are resolved by `src/lib/proxy/resolve-skills.ts`.

### phase prompt

The base system prompt for a specific phase + project type + stack combination. Stored in `phase_prompts`. The most-specific matching row wins (projectType + stack > projectType only > stack only > universal). Defined the agent's role and expected output format.

### client portal

A read-only view of a client's published deliverables, accessible via a shareable link (`/portal/[token]`). No agency-internal data is exposed. Clients can approve or request changes on published deliverables.

---

## Auth concepts

### super_admin

A session-level role resolved by `isSuperAdmin()` — checks whether the current user is a member of the Clerk org identified by `CLERK_ADMIN_ORG_ID` with role `org:admin` or `org:super_admin`. Controls access to `/admin/*` routes.

Distinct from `accountType = "platform"`: a user can be a super_admin (Clerk org membership) without their account having `accountType = "platform"`, and vice versa.

### resolveAccount()

The canonical auth resolution function in `src/lib/session.ts`. Looks up the Clerk session user, calls `getOrCreateAccount`, and returns the `Account` row. Every authenticated API handler calls this. Returns `null` (unauthenticated) or an `Account`.
