---
name: Design Tokens and Theming
description: Use CSS custom properties for all design values; never hardcode colors or magic-number spacing; always pair dark-mode variants.
category: design
applicable_phases: [code_gen, design_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

## Design tokens

All design values live as CSS custom properties in the `@theme` block inside `globals.css`. Required token categories:

- **base** — background, foreground, border, input, ring
- **brand surfaces** — primary, secondary, accent (+ `-foreground` pairs)
- **status** — destructive, warning, success, info (+ `-foreground` pairs)
- **accents** — muted, card, popover (+ `-foreground` pairs)
- **typography** — font-sans, font-mono, font-size scale

Never hardcode a hex, `rgb()`, or `rgba()` value directly in a component or utility class. Use the token or a Tailwind token that maps to it.

```tsx
// bad
<div className="bg-[#1a1a2e] text-[#ffffff]">
  <p style={{ color: "#6b7280" }}>...</p>
</div>

// good
<div className="bg-background text-foreground">
  <p className="text-muted-foreground">...</p>
</div>
```

For dynamic inline styles (e.g., charts), use CSS variables — no hex fallbacks:

```tsx
// bad
style={{ color: `var(--color-primary, #6366f1)` }}

// good
style={{ color: "var(--color-primary)" }}
```

## Typography

- Inter (sans) as `--font-sans`; JetBrains Mono (mono) as `--font-mono`
- Apply via the Tailwind `font-sans` / `font-mono` utilities — never hardcode a `font-family` string in CSS

## Icons

- Lucide React for all UI icons — do not reach for Heroicons, Feather, or other sets
- Custom icons live in `src/components/icons/` as plain React components
- Use icon-helper functions (`getToolIcon(type)`, etc.) that return `{ icon, iconColor, iconBg }` for consistent badge rendering; never reconstruct those values inline per-callsite

```tsx
// bad — per-callsite reconstruction
<div className="rounded bg-blue-100">
  <Wrench className="h-4 w-4 text-blue-600" />
</div>;

// good — shared helper
const { icon: Icon, iconColor, iconBg } = getToolIcon(tool.type);
<div className={cn("rounded", iconBg)}>
  <Icon className={cn("h-4 w-4", iconColor)} />
</div>;
```

## Theming (light / dark)

- The `.dark` class is applied to `<html>` — never to a subtree
- Theme resolution order: `localStorage` > auth-provider user metadata > `prefers-color-scheme`
- All resolution and sync go through the `useTheme` hook — do not read `localStorage` or `document.documentElement` directly anywhere else
- Ship `theme-init.js` as a `beforeInteractive` script to apply the correct class before first paint and prevent flash

```tsx
// bad — direct DOM manipulation outside useTheme
document.documentElement.classList.toggle("dark", isDark);

// good — always via the hook
const { theme, setTheme } = useTheme();
setTheme("dark");
```

Every custom Tailwind class or inline style that changes appearance must include a `dark:` variant:

```tsx
// bad
<div className="bg-white border-gray-200">

// good
<div className="bg-white dark:bg-zinc-900 border-gray-200 dark:border-zinc-700">
```

## Design review flags

During code review, flag:

- Any hardcoded hex, `rgb()`, or `rgba()` — replace with CSS variable or Tailwind token
- Any `dark:` variant missing on a custom background, border, or text color
- Any `font-family` string outside `globals.css`
- Any icon imported from a library other than Lucide React
