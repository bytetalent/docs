---
name: RLS Policy Templates
description: Supabase RLS patterns for user-owned and shared resources; Drizzle defense-in-depth; Clerk JWT wiring; service-role bypass rules.
category: db
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 2
---

Supabase RLS policies (Clerk JWT-based auth). Source: `bytetalent/docs/guide-db.md` and `bytetalent/docs/guide-arch.md`.

## RLS is the primary security boundary

RLS is the primary security boundary. App-layer `accountId` filtering is defense in depth. Both layers must be present — RLS alone is not sufficient because server-side code uses a service-role connection that bypasses RLS.

## Policy templates

### User-owned table

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users access their own rows"
  ON projects
  USING (account_id::text = auth.jwt()->>'sub');

-- INSERT additionally needs WITH CHECK
CREATE POLICY "users insert their own rows"
  ON projects
  FOR INSERT
  WITH CHECK (account_id::text = auth.jwt()->>'sub');

-- UPDATE needs both USING (read filter) and WITH CHECK (write filter)
CREATE POLICY "users update their own rows"
  ON projects
  FOR UPDATE
  USING  (account_id::text = auth.jwt()->>'sub')
  WITH CHECK (account_id::text = auth.jwt()->>'sub');
```

### Org-shared table

```sql
CREATE POLICY "org members access shared rows"
  ON project_deliverables
  USING (
    organization_id IN (
      SELECT organization_id FROM accounts
      WHERE id::text = auth.jwt()->>'sub'
    )
  );
```

### System catalog / reference table (read-only)

```sql
ALTER TABLE model_catalog ENABLE ROW LEVEL SECURITY;

CREATE POLICY "anyone can read catalog"
  ON model_catalog
  FOR SELECT
  USING (true);

-- No INSERT/UPDATE/DELETE policy → denied for all roles except service role
```

### Junction tables

Junction tables inherit the most-restrictive policy of either side.

## Rules

- Never disable RLS as a workaround. If a policy blocks a legitimate use, fix the policy or use the service role with documented intent.
- INSERT policies use `WITH CHECK`; UPDATE uses both `USING` and `WITH CHECK`.
- RLS changes are migrations — commit them alongside the schema change.

## Drizzle defense-in-depth

Drizzle queries must also filter by `accountId` from the server-side session, even though RLS enforces the same constraint. This catches errors earlier (application layer, before the DB round-trip) and provides an audit trail.

```ts
// Good — both RLS and app-layer filter
const project = await db.query.projects.findFirst({
  where: and(
    eq(projects.id, id),
    eq(projects.accountId, accountId), // defense in depth
    isNull(projects.deletedAt)
  ),
});

// Bad — relying on RLS alone, no app-layer filter
const project = await db.query.projects.findFirst({
  where: eq(projects.id, id), // RLS fires but app layer has no guard
});
```

## JWT wiring (Clerk + Supabase)

1. Create a Clerk JWT Template named **`supabase`** in the Clerk dashboard
2. Configure **Third Party Auth** in Supabase with the Clerk JWKS URL
3. The template sets `sub` to the Clerk user ID; `auth.jwt()->>'sub'` in RLS resolves to this value

```
Clerk JWT Template "supabase":
{
  "sub": "{{user.id}}",
  "role": "authenticated"
}
```

## Service-role bypass

When server-side code needs to bypass RLS (e.g. background jobs, admin operations), use Drizzle with the service-role connection and document the intent with a code comment.

```ts
// Good — Drizzle with service-role + documented intent
// Service-role: admin cleanup job; accountId filter not applicable
const staleProjects = await adminDb.query.projects.findMany({
  where: lt(projects.createdAt, cutoff),
});

// Bad — creating a parallel supabase-js admin client
import { createClient } from "@supabase/supabase-js";
const admin = createClient(url, serviceRoleKey); // never add parallel clients
```
