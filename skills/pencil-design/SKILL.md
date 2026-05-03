---
name: Pencil Design
description: Working with .pen design files via Pencil MCP — encrypted access, batch limits, cross-file components, session hygiene, and parallel agent caps.
category: design
applicable_phases: [design_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase, astro-cloudflare]
version: 1
---

Rules for working with Pencil MCP and `.pen` design files. Per `bytetalent/docs/guide-pencil.md`.

## .pen files are encrypted

`.pen` files are encrypted at rest. **Never** use `Read`, `Grep`, `cat`, or any file-access tool on a `.pen` file. All reads and writes must go through the Pencil MCP tools:

```
get_editor_state, open_document, get_guidelines, batch_get, batch_design,
snapshot_layout, get_screenshot, get_variables, set_variables,
find_empty_space_on_canvas, search_all_unique_properties,
replace_all_matching_properties, export_nodes
```

Attempting to read a `.pen` file directly will produce garbled binary output, not design data.

## Pencil editor must be open

The Pencil desktop app must be running and the target file open before any MCP call. There is no headless mode. If an MCP call returns an error about the editor not being available, open the Pencil app first.

## batch_design limit

`batch_design` accepts a maximum of **25 operations per call**. Exceeding this limit causes the call to fail. Split larger design operations across multiple `batch_design` calls.

```typescript
// Good — split across two calls if > 25 operations
const firstBatch = operations.slice(0, 25);
const secondBatch = operations.slice(25, 50);
await batch_design(firstBatch);
await batch_design(secondBatch);

// Bad — one call with 40 operations will fail
await batch_design(operations); // 40 items — exceeds limit
```

## Components cannot be referenced across files

Pencil components are scoped to a single `.pen` file. You cannot reference a component defined in `file-a.pen` from `file-b.pen`. If a component is needed in a different file, copy it over first using `batch_design` copy operations.

## Design session hygiene

Accumulate all sections for a project or feature within **one comprehensive `.pen` file** across sessions. Do not create a new file per task or per session. Design files grow organically with the project and provide a stable canvas for iteration.

```
// Good structure — one file per feature area
design/
  brand-kit.pen
  onboarding.pen
  dashboard.pen

// Bad — one file per task creates fragmented, unreusable design work
design/
  task-signup-form-2026-05-01.pen
  task-signup-form-v2-2026-05-02.pen
  task-dashboard-header.pen
```

## Parallel agent cap

When spawning multiple designer agents in parallel (via `spawn_agents`), cap at **8–10 agents at once**. Exceeding this limit degrades editor performance and may cause MCP timeouts.
