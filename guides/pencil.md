# Pencil MCP — Capabilities for `.pen` Design Files

Reference for what Claude agents can do via the Pencil MCP server when working with `.pen` design files. The canonical tool catalog, node types, operation syntax, and verified capabilities live in [`skills/design/pencil-design`](../skills/design/pencil-design/SKILL.md).

---

## Workflow Friction

The MCP is editor-coupled by design. Practical implications:

- The Pencil app must be open before any MCP call works
- `open_document` opens an existing file but doesn't create a fresh blank file from nothing — use **File → New** in the Pencil UI for that
- File renames (e.g., adding `.pen` extension) need to happen via the Pencil UI's Save As, not via filesystem operations alone — Pencil's file watcher needs to track the file

For long design sessions, open one comprehensive `.pen` file (e.g., `bt-internal-design.pen`) and accumulate sections within it across multiple working sessions, rather than creating one file per task.

---

## Canonical reference

**Tool catalog, node types, `batch_design` operation syntax, agentic parallelism, verified capabilities, limitations:** see [`skills/design/pencil-design`](../skills/design/pencil-design/SKILL.md).
