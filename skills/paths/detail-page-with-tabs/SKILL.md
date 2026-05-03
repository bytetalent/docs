---
name: Detail Page With Tabs
description: Multi-view entity detail page with a tab bar — each tab renders a distinct view of the same entity.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Entity detail pages with more than one view dimension use a tab bar with `border-b` styling. Tabs can be local `useState` (simple) or URL-synced (shareable deep links).

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #4.

## Core rules

- Tab bar is a row of `<button>` elements with `border-b-2` active highlight, rendered inline in JSX using `cn()`.
- Active tab state is `string` (tab id). Default to the first tab or URL param on mount.
- Each tab renders its content only when active — no hidden/display-none mounting.
- Tab visibility can be gated (e.g., "Runs" tab only appears on saved entities, not new-entity forms).
- For deep-linkable tabs (billing, settings), sync tab to `?tab=<id>` via `useRouter` / `useSearchParams`.
- For local tabs (flow editor, run detail), plain `useState` is sufficient.

## Inline tab bar shape

```tsx
"use client";
import { useState } from "react";
import { cn } from "@/lib/utils";

type Tab = "overview" | "runs" | "settings";

const TABS: { id: Tab; label: string }[] = [
  { id: "overview", label: "Overview" },
  { id: "runs", label: "Runs" },
  { id: "settings", label: "Settings" },
];

export function EntityDetailPage({ entityId }: { entityId: string }) {
  const [activeTab, setActiveTab] = useState<Tab>("overview");

  return (
    <DashboardPageLayout title="Entity Name">
      {/* Tab bar */}
      <div className="border-b mb-6">
        <nav className="flex gap-1">
          {TABS.map((tab) => (
            <button
              key={tab.id}
              onClick={() => setActiveTab(tab.id)}
              className={cn(
                "px-4 py-2 text-sm font-medium border-b-2 transition-colors",
                activeTab === tab.id
                  ? "border-primary text-primary"
                  : "border-transparent text-muted-foreground hover:text-foreground",
              )}
            >
              {tab.label}
            </button>
          ))}
        </nav>
      </div>

      {/* Tab content */}
      {activeTab === "overview" && <OverviewTab entityId={entityId} />}
      {activeTab === "runs" && <RunsTab entityId={entityId} />}
      {activeTab === "settings" && <SettingsTab entityId={entityId} />}
    </DashboardPageLayout>
  );
}
```

## URL-synced tab state

For tabs that should be deep-linkable:

```tsx
"use client";
import { useRouter, useSearchParams } from "next/navigation";

type Tab = "usage" | "plan" | "billing";

export function BillingPage() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const activeTab = (searchParams.get("tab") as Tab) ?? "usage";

  const setTab = (tab: Tab) => {
    router.push(`?tab=${tab}`, { scroll: false });
  };

  return (
    <>
      <nav className="flex border-b mb-6">
        {(["usage", "plan", "billing"] as Tab[]).map((tab) => (
          <button
            key={tab}
            onClick={() => setTab(tab)}
            className={cn(
              "px-4 py-2 text-sm border-b-2",
              activeTab === tab
                ? "border-primary text-primary"
                : "border-transparent text-muted-foreground",
            )}
          >
            {tab.charAt(0).toUpperCase() + tab.slice(1)}
          </button>
        ))}
      </nav>
      {activeTab === "usage" && <UsageTab />}
      {activeTab === "plan" && <PlanTab />}
      {activeTab === "billing" && <BillingInfoTab />}
    </>
  );
}
```

## Gated tab visibility

```tsx
// Tab only appears when the entity is saved (has an id)
const visibleTabs = TABS.filter((tab) => {
  if (tab.id === "runs") return entityId !== "new";
  return true;
});
```

## Reusable shell variant

When multiple entity types share the same tab structure (e.g., assistants, tools, channels all have Overview + Runs), extract a `DetailShell` component that accepts the entity + renders the tabs:

```tsx
// comp-detail-shell.tsx
interface Props {
  entity: { id: string; name: string; type: string };
  defaultTab?: "overview" | "runs";
}

export function DetailShell({ entity, defaultTab = "overview" }: Props) {
  const [tab, setTab] = useState(defaultTab);
  // render tab bar + content
}
```

## What NOT to do

```tsx
// Bad — all tab content mounted simultaneously with display:none
{
  TABS.map((tab) => (
    <div
      key={tab.id}
      style={{ display: activeTab === tab.id ? "block" : "none" }}
    >
      <TabContent />{" "}
      {/* Mounts and runs effects for every tab, not just the active one */}
    </div>
  ));
}
```

```tsx
// Bad — using a UI library Tabs component when the inline pattern is established
import { Tabs, TabsList, TabsTrigger, TabsContent } from "@/components/ui/tabs";
// Use the inline border-b pattern consistently instead of library Tabs
```

## Related skills

- `skills/paths/server-client-page-split` — the RSC shell that wraps this page-client
- `skills/nextjs-component-conventions` — component file naming + co-location
