# Tooling permissions

What the agent should run without asking versus what genuinely needs confirmation. The intent: routine reversible operations should run instantly; only operations that destroy data, overwrite published history, or expose secrets should pause for permission.

## MCP and local file ops — autorun (read AND write)

Do not ask for permission before calling Vercel or Supabase MCP tools, OR before reading/writing local files. This includes writes — `apply_migration`, `execute_sql` with SELECT/INSERT/UPDATE, `deploy_edge_function`, `merge_branch`, `deploy_to_vercel`, etc. Just run them.

**Even SELECT queries should not be flagged for confirmation** — read queries are inherently safe.

**Why:** User has given this feedback at least four times across sessions: "please stop asking for permission on vercel and supabase" (staging Clerk/DB debugging), "stop asking me for permission to use mcp servers, or read or write files locally" (env config), "the only thing you need to ask about is data/object deletion" (2026-04-28), and again "you can add you can modify, just ask me to delete" (2026-04-30). The repeat tells me I keep partially internalizing this. Permission prompts on routine MCP/file work slow iteration and add no safety value on reversible operations. The user has manual access to revert anything; the prompt-tax is what bothers them.

**How to apply:** Call MCP tools and Read/Write/Edit immediately. No "let me check the data first" preamble that proposes a SELECT — just run the SELECT. Migrations (CREATE/ALTER), inserts/updates, deployments, merges, edge function writes, all SELECT/read calls — proceed without asking. If catching myself typing "want me to run this query?" before a SELECT, stop and just run it.

## Dev tooling — autorun

For each PR the same validation pass runs: `npx prettier --check/--write`, `npx eslint`, `npx tsc --noEmit`, sometimes `npm install`. The user has these auto-allowed in `.claude/settings.local.json` (both Bash and PowerShell variants). Don't suggest different commands or work around them — just run.

**Why:** "while I appreciate the prettier code can you stop asking me to approve powershell for the formatting work you do each pr?" (2026-04-28). Confirmation prompts on routine validation are friction with zero safety value.

**How to apply:** Use `npx prettier`, `npx eslint`, `npx tsc`, `npm install`, `npm run format/lint/test` freely during PR validation. Don't substitute `pnpm` or invent ad-hoc bash piping just to avoid the dev-tooling commands. If a new dev-tooling tool comes up that isn't covered, it's fine to add it to settings.local.json without asking.

## WebFetch — autorun

`WebFetch` is whitelisted in `.claude/settings.local.json` for any URL. Don't ask before fetching docs / vendor pages — just fetch. This pairs with `engineering-discipline.md → Read docs end-to-end before integrating an SDK`.

## What still needs confirmation — the deletion taxonomy

The ONLY operations that require explicit confirmation are ones that **delete** data or objects, **overwrite published history**, or **expose secrets**:

- **DB destructive**: `DROP TABLE`, `DROP TYPE`, `DROP COLUMN`, `DROP DATABASE`, `DELETE FROM` (any — even with WHERE), `TRUNCATE`
- **Service destructive**: `delete_branch`, `delete_project`, `delete_edge_function`
- **Filesystem destructive**: `rm -rf` of files that are not clearly throwaway/build artifacts
- **Git destructive**: force-push that overwrites *published* history. Force-push that only rewrites a branch I just locally rewrote in this session is fine — see `multi-agent-git-workflow.md`.
- **Secret exposure**: `Read` on `.env*` / credential / key files — see `engineering-discipline.md → Never read whole .env files`

For everything else: run it, surface the result, let the user revert if needed.
