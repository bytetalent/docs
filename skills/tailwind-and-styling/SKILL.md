---
name: Tailwind and Styling
description: cn() for class composition, cva for variants, CSS variables only in dynamic styles, @utility for custom utilities.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Styling conventions for Tailwind CSS v4. Source: guide-code.md §8 and guide-design.md.

## Class composition — cn() (source: guide-code.md §8)

Use `cn()` from `src/lib/utils.ts` for all conditional class composition.

```typescript
import { cn } from "@/lib/utils";

// Good
<div className={cn("base-class", isActive && "active-class", className)} />

// Bad — manual string concatenation
<div className={`base-class ${isActive ? "active-class" : ""} ${className}`} />
```

## Variant-heavy components — cva (source: guide-code.md §8)

Use `cva` (class-variance-authority) for reusable components with multiple
visual variants.

```typescript
import { cva, type VariantProps } from "class-variance-authority";

const badge = cva("inline-flex items-center rounded-full px-2 py-1 text-xs font-medium", {
  variants: {
    variant: {
      default: "bg-muted text-muted-foreground",
      success: "bg-status-success-bg text-status-success-text",
      error: "bg-status-error-bg text-status-error-text",
    },
  },
  defaultVariants: { variant: "default" },
});

type BadgeProps = React.HTMLAttributes<HTMLSpanElement> & VariantProps<typeof badge>;
```

## CSS variables only in dynamic styles (source: guide-code.md §8)

When a style value must be set dynamically (e.g. chart colors, entity accent colors),
use CSS custom properties. **Never use hex fallbacks**.

```typescript
// Good
<div style={{ color: "var(--color-primary)" }} />
<div style={{ backgroundColor: `var(--color-${entity.accentBg})` }} />

// Bad — hex fallback makes CSS variables optional; tokens drift
<div style={{ color: "var(--color-primary, #96c237)" }} />
<div style={{ color: "#96c237" }} />
```

## Custom utilities — @utility (source: guide-code.md §8)

Reusable text utilities are defined as `@utility` in `globals.css`.
Use them instead of repeating raw class strings:
`text-field-label`, `text-section-heading`, `text-2xs`, etc.

```tsx
// Good
<label className="text-field-label">Email</label>

// Bad — repeating the full class string at every call site
<label className="text-xs font-medium uppercase tracking-wider text-muted-foreground">
  Email
</label>
```

## Dark mode (source: guide-design.md)

- `.dark` class is applied to `<html>` via `useTheme`.
- Every custom style must have a `dark:` variant.
- Flag any hardcoded hex, rgb, or rgba value in code review — use CSS variables
  or Tailwind tokens instead.

```typescript
// Good
<div className="bg-background text-foreground dark:bg-background dark:text-foreground" />

// Bad — hardcoded color with no dark variant
<div className="bg-[#1a1a2e] text-white" />
```
