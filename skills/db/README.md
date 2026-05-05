# DB

Database query patterns, Drizzle migrations, and RLS policy templates.

## Skills

- [db-query-patterns](db-query-patterns/SKILL.md) — Every read includes accountId + deletedAt filters; every write includes version match in WHERE + increment in SET + updatedAt + updatedBy; use Drizzle transactions for multi-write atomicity.
- [drizzle-migration-patterns](drizzle-migration-patterns/SKILL.md) — One migration file per schema change; never edit applied migrations; include both schema + RLS in the same migration; apply dev → staging → prod in sequence.
- [rls-policies](rls-policies/SKILL.md) — Supabase RLS patterns for user-owned and shared resources; Drizzle defense-in-depth; Clerk JWT wiring; service-role bypass rules.
