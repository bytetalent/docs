# Skills

Universal Bytetalent skill library. Each skill is a SKILL.md file consumed by the Claude agent during phase generation.

Skills are grouped into category subdirectories based on their `category:` frontmatter field.

## Categories

| Category | Count | Description |
|---|---|---|
| [api](api/README.md) | 6 | REST routes, validation, pagination, streaming, webhooks, concurrency |
| [architecture](architecture/README.md) | 8 | Next.js, Astro, Cloudflare Workers, Supabase schema, templates |
| [code](code/README.md) | 11 | TypeScript, components, forms, tables, error handling, styling |
| [connections](connections/README.md) | 1 | Credential management and the connections framework |
| [db](db/README.md) | 3 | Query patterns, Drizzle migrations, RLS policies |
| [design](design/README.md) | 5 | Accessibility, design tokens, marketing layouts, Pencil, interactive components |
| [infra](infra/README.md) | 2 | Deployment conventions, infrastructure service defaults |
| [meta](meta/README.md) | 4 | Planning philosophy and decision heuristics |
| [paths](paths/) | — | Path alias and import resolution patterns (flat structure, no subdir) |
| [prd](prd/README.md) | 1 | Product requirements and scope management |
| [process](process/README.md) | 7 | Backlog, board conventions, branch hygiene, GitHub API, worker dispatch |
| [security](security/README.md) | 3 | Auth patterns, CSP configuration, and security review |
| [testing](testing/README.md) | 1 | Testing pyramid, Playwright, isolation conventions |

## Usage

Skills are loaded by the bt-ai-web skill manifest (`src/lib/proxy/skill-manifest.ts`), which traverses `<category>/<slug>/SKILL.md` and filters by `applicable_phases` and `applicable_stacks` frontmatter fields.

The `paths/` directory keeps its existing flat structure and is skipped by the 2-level traversal.
