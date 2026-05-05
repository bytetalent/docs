---
name: Theme Hook
description: Single-source useTheme hook pattern — three-priority reconciliation (localStorage → user metadata → system preference) and the rule against parallel theme state.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Theme state must have exactly one source of truth: the `useTheme` hook (`src/hooks/use-theme.ts`). Multiple components independently managing theme state produces flash, desync, and conflicting writes. Source: `bytetalent/docs/guides/code.md` §9 Established Patterns — Theme.

## Priority reconciliation order

The hook resolves the active theme by checking sources in this order, stopping at the first defined value:

1. **`localStorage`** — the user's explicit selection on this device (survives page reload)
2. **Auth provider user metadata** — the user's saved preference on the server (survives device switch)
3. **System preference** — `prefers-color-scheme` media query (fallback for new users)

```ts
// Good — single hook, priority cascade
const { theme, setTheme } = useTheme();

// Bad — reading localStorage directly in a component
const theme = typeof window !== "undefined"
  ? localStorage.getItem("theme") ?? "system"
  : "system";
```

## What `useTheme` owns

- Reading from all three sources and returning the resolved value
- Writing the user's explicit choice to both `localStorage` and the auth provider's user metadata
- Emitting a DOM class update (`document.documentElement.classList`) so Tailwind dark-mode picks it up

## Usage rules

- **Every component that needs the current theme calls `useTheme()`** — the top-bar, profile page, and any theme toggle all read from the same hook.
- **Never create separate theme state** (`useState("dark")` at component level, a second context, a second localStorage key).
- **Never let a page-level component or layout update `localStorage` directly** — route the write through `setTheme()` so the hook's internal state stays consistent.

## Flash-prevention

The hook's initial render must resolve before hydration to prevent a flash of the wrong theme. The standard approach is a blocking inline script in `<head>` that reads `localStorage` and sets the `dark` class synchronously — before React hydrates. This script is the only legitimate place to read `localStorage` outside the hook.

```html
<!-- Good — blocking class set before hydration -->
<script>
  (function () {
    var stored = localStorage.getItem("theme");
    if (stored === "dark" || (!stored && window.matchMedia("(prefers-color-scheme: dark)").matches)) {
      document.documentElement.classList.add("dark");
    }
  })();
</script>
```

## Anti-patterns

- Duplicating theme state in a second React context alongside `useTheme`.
- Reading `document.documentElement.classList` to determine the current theme — that's the output, not the source of truth.
- Skipping the auth-provider metadata write on `setTheme` — the user's preference will reset on a new device.

## Refs

- `bytetalent/docs/guides/code.md` §9 — Established Patterns, Theme sub-section
- `skills/design/design-tokens-and-theming/SKILL.md` — design token system that the theme class applies
