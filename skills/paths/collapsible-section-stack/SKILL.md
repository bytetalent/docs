---
name: Collapsible Section Stack
description: Dense entity-editing pages organized as a vertical stack of named CollapsiblePanel sections — title, icon, default-open/closed state.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
composes_with: [async-toast-mutation, paginated-data-table]
nests: [form-patterns]
conflicts_with: [detail-page-with-tabs]
---

When an entity has many fields that don't all need to be visible at once, organize them into a vertical stack of named collapsible panels. Each panel groups logically related fields under a title.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #16.

## Core rules

- Use `CollapsiblePanel` (from `@/components/shared/comp-collapsible`) as the section primitive — do not build custom accordion components.
- Each panel has: a `title` string, an optional `icon` (Lucide icon), and `defaultOpen` boolean.
- High-importance sections (above the fold) default open; advanced/secondary sections default closed.
- Each panel manages only its own fields — no cross-panel state.
- Page renders the panels in a `<div className="space-y-4">` container.
- A "Save" button can live inside each panel's footer, or outside in a page-level footer (but not both).

## Usage

```tsx
"use client";
import { CollapsiblePanel } from "@/components/shared/comp-collapsible";
import { FileText, Users, Zap, Settings } from "lucide-react";

export function EntityEditorPage({ entityId }: { entityId: string }) {
  return (
    <DashboardPageLayout title="Edit Entity">
      <div className="space-y-4 max-w-3xl">
        <CollapsiblePanel
          title="Overview"
          icon={<FileText className="w-4 h-4" />}
          defaultOpen={true}
        >
          <OverviewFields entityId={entityId} />
        </CollapsiblePanel>

        <CollapsiblePanel
          title="Audience"
          icon={<Users className="w-4 h-4" />}
          defaultOpen={true}
        >
          <AudienceFields entityId={entityId} />
        </CollapsiblePanel>

        <CollapsiblePanel
          title="Features"
          icon={<Zap className="w-4 h-4" />}
          defaultOpen={false}
        >
          <FeaturesList entityId={entityId} />
        </CollapsiblePanel>

        <CollapsiblePanel
          title="Advanced"
          icon={<Settings className="w-4 h-4" />}
          defaultOpen={false}
        >
          <AdvancedSettings entityId={entityId} />
        </CollapsiblePanel>
      </div>
    </DashboardPageLayout>
  );
}
```

## CollapsiblePanel component contract

```tsx
// comp-collapsible.tsx
interface CollapsiblePanelProps {
  title: string;
  icon?: React.ReactNode;
  subtitle?: string;
  defaultOpen?: boolean;
  children: React.ReactNode;
  className?: string;
}

export function CollapsiblePanel({
  title,
  icon,
  subtitle,
  defaultOpen = true,
  children,
  className,
}: CollapsiblePanelProps) {
  const [open, setOpen] = useState(defaultOpen);
  return (
    <div className={cn("border rounded-lg", className)}>
      <button
        onClick={() => setOpen(!open)}
        className="flex items-center gap-2 w-full px-4 py-3 text-left hover:bg-muted/50"
      >
        {icon}
        <div className="flex-1">
          <span className="font-medium text-sm">{title}</span>
          {subtitle && (
            <p className="text-xs text-muted-foreground">{subtitle}</p>
          )}
        </div>
        <ChevronDown
          className={cn("w-4 h-4 transition-transform", open && "rotate-180")}
        />
      </button>
      {open && <div className="px-4 pb-4">{children}</div>}
    </div>
  );
}
```

## When to choose this vs. tabs

Use collapsible panels when:

- The entity has 4+ logical field groups.
- Users need to scroll through multiple groups in one session (editor-style).
- Some groups are rarely changed (can default-closed).

Use tabs (see `skills/paths/detail-page-with-tabs`) when:

- Field groups are mutually exclusive views (user looks at one at a time).
- Groups are complex enough to each fill a full viewport.

## Content inside panels

Each panel's content component follows the same pattern: props-in, events-out, no direct fetch of unrelated data:

```tsx
// Panel content component
function OverviewFields({
  overview,
  onChange,
}: {
  overview: string;
  onChange: (v: string) => void;
}) {
  return (
    <div className="space-y-3 pt-2">
      <Label>Overview</Label>
      <Textarea
        value={overview}
        onChange={(e) => onChange(e.target.value)}
        rows={4}
      />
    </div>
  );
}
```

## What NOT to do

```tsx
// Bad — inline expansion logic instead of CollapsiblePanel
const [openSections, setOpenSections] = useState<Set<string>>(
  new Set(["overview"]),
);
// Use the shared CollapsiblePanel component
```

```tsx
// Bad — all sections default open on a page with 6+ panels
<CollapsiblePanel title="Overview" defaultOpen={true}>
<CollapsiblePanel title="Audience" defaultOpen={true}>
<CollapsiblePanel title="Features" defaultOpen={true}>
<CollapsiblePanel title="Tech Stack" defaultOpen={true}>
<CollapsiblePanel title="Timeline" defaultOpen={true}>
<CollapsiblePanel title="Advanced" defaultOpen={true}>
{/* Page loads showing 6 full panels — too much. Default close secondary ones. */}
```

## Related skills

- `skills/paths/settings-sections-page` — uses stacked cards; this skill uses collapsible panels
- `skills/paths/detail-page-with-tabs` — alternate organization for mutually exclusive views
- `skills/paths/paginated-data-table` — when a paginated list is inside a page using collapsible panels, stat cards go inside a `CollapsiblePanel` (stat-card-in-panel pattern)
