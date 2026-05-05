# Pencil MCP — Capabilities for `.pen` Design Files

Reference for what Claude agents can do via the Pencil MCP server when working with `.pen` design files.

## Document Management

| Tool | What it does |
|---|---|
| `get_editor_state` | Get current file, selection, top-level nodes, reusable components |
| `open_document` | Open an existing `.pen` file or create a new one |
| `get_variables` | Read design tokens (colors, fonts, spacing variables) |
| `set_variables` | Create or update design variables and themes |
| `get_guidelines` | Load design guides (Web App, Mobile, Table, Slides, etc.) and style presets |

## Reading & Searching

| Tool | What it does |
|---|---|
| `batch_get` | Search nodes by type/name/reusable pattern, read node trees by ID with configurable depth |
| `snapshot_layout` | Get computed layout rectangles, detect clipping/overlap issues |
| `get_screenshot` | Render any node as an image for visual verification |
| `find_empty_space_on_canvas` | Find open areas to place new content near existing nodes |
| `search_all_unique_properties` | Find all unique values for a given property across the document |
| `export_nodes` | Export nodes to image/SVG formats |

## Writing & Designing

| Tool | What it does |
|---|---|
| `batch_design` | Execute up to 25 operations per call |
| `replace_all_matching_properties` | Bulk find-and-replace property values across the document |

### `batch_design` Operations

| Operation | Syntax | Description |
|---|---|---|
| **Insert** | `foo=I("parent", { ... })` | Create a new node inside a parent |
| **Copy** | `baz=C("nodeId", "parent", { ... })` | Duplicate a node into a target parent |
| **Update** | `U("nodeId", { ... })` | Modify properties of an existing node |
| **Replace** | `foo2=R("nodeId", { ... })` | Replace a node with a new one |
| **Delete** | `D("nodeId")` | Remove a node |
| **Move** | `M("nodeId", "newParent", index)` | Move a node to a new parent at a specific index |
| **Image** | `G("nodeId", "ai"\|"stock", "prompt")` | Apply AI-generated or stock image fill to a node |

## Node Types Agents Can Create

| Category | Types |
|---|---|
| **Layout** | `frame`, `group` |
| **Shapes** | `rectangle`, `ellipse`, `line`, `polygon`, `path` |
| **Content** | `text`, `icon_font` |
| **Annotations** | `note`, `prompt`, `context` |
| **Instances** | `ref` (component instances with descendant overrides) |

## Agentic Parallelism

| Tool | What it does |
|---|---|
| `spawn_agents` | Launch up to 8–10 parallel designer agents, each with assigned container nodes and independent prompts |

Agents work independently and concurrently on different parts of the canvas. The main session continues working on remaining tasks while spawned agents run in parallel.

## Verified Capabilities

The following capabilities have been tested and confirmed in practice:

1. **Search** — Find nodes by type across the entire document
2. **Read** — Extract content, position, and properties of any node
3. **Create** — Build full screens from scratch (stat cards, charts, tables, pagination, action bars)
4. **Copy & Modify** — Duplicate frames and customize descendants with property overrides
5. **Layout** — Flexbox with `gap`, `padding`, `justifyContent`, `alignItems`, `fill_container`/`fit_content` sizing
6. **Graphics** — Gradient fills, strokes, path geometry (area charts, line charts), corner radius
7. **Components** — Use reusable components via `ref` with property overrides on nested descendants
8. **Screenshots** — Visual verification at any stage of the design process
9. **Parallel Agents** — Spawned agents to build multiple screens while main session built others simultaneously
10. **Annotations** — Create `note` / `prompt` / `context` annotations positioned near specific frames
11. **Variables** — Read design tokens and bind them to node properties with `$variable-name` syntax

## Limitations

- `.pen` file contents are encrypted and can **only** be accessed via Pencil MCP tools (not `Read`, `Grep`, or `cat`)
- `batch_design` should be kept to **max 25 operations** per call for optimal performance
- Components cannot be referenced across files — must be copied over first
- Line charts and complex point-based visualizations require manual path geometry
- `spawn_agents` max is 8–10 agents at once
- Pencil MCP requires the editor app running locally — no headless mode

## Workflow Friction

The MCP is editor-coupled by design. Practical implications:

- The Pencil app must be open before any MCP call works
- `open_document` opens an existing file but doesn't create a fresh blank file from nothing — use **File → New** in the Pencil UI for that
- File renames (e.g., adding `.pen` extension) need to happen via the Pencil UI's Save As, not via filesystem operations alone — Pencil's file watcher needs to track the file

For long design sessions, open one comprehensive `.pen` file (e.g., `bt-internal-design.pen`) and accumulate sections within it across multiple working sessions, rather than creating one file per task.
