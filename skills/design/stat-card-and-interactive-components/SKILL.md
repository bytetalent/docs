---
name: Stat Card and Interactive Components
description: Stat card anatomy, ConfirmDialog usage, AIProgressPanel usage, and AI auto-trigger patterns for phase pages.
category: design
applicable_phases: [code_gen, design_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

## Stat card anatomy

Stat cards display a metric value with a title. Fixed typographic scale:

- **Title** — `text-[10px] font-bold uppercase tracking-wider text-muted-foreground`
- **Value** — `text-3xl font-extrabold`
- No CTA button ("Manage →" etc.) inside the stat card

```tsx
// bad — CTA inside stat card
<Card>
  <p className="text-xs font-semibold uppercase text-gray-500">Active Flows</p>
  <p className="text-2xl font-bold">{count}</p>
  <Button size="sm">Manage →</Button>
</Card>

// good — click target is the whole card; no CTA button
<Card
  className="cursor-pointer hover:shadow-md transition-shadow"
  onClick={() => router.push(ROUTES.FLOWS + "?status=active")}
>
  <p className="text-[10px] font-bold uppercase tracking-wider text-muted-foreground">
    Active Flows
  </p>
  <p className="text-3xl font-extrabold">{count}</p>
</Card>
```

The whole card is the click target and navigates to the relevant filtered list page.

## ConfirmDialog

Use `ConfirmDialog` for any non-trivial or destructive action. Never use `window.confirm()`.

Required props:

- `consequences` — bulleted list of what will happen (so the user can make an informed decision)
- `severity` — `"success"` for forward-motion actions (archive, regenerate); `"destructive"` for irreversible deletes

```tsx
// bad
if (window.confirm("Delete this project?")) {
  await deleteProject(id);
}

// good
<ConfirmDialog
  open={showConfirm}
  onOpenChange={setShowConfirm}
  title="Delete Project"
  severity="destructive"
  consequences={[
    "All project deliverables will be permanently deleted.",
    "Connected integrations will be unlinked.",
    "This cannot be undone.",
  ]}
  onConfirm={() => deleteProject(id)}
/>;
```

For regeneration or other forward-motion actions, use `severity="success"` (green confirm button):

```tsx
<ConfirmDialog
  severity="success"
  title="Regenerate PRD"
  consequences={["The existing PRD will be replaced with a new AI-generated version."]}
  onConfirm={handleRegenerate}
/>
```

## AIProgressPanel

Show `AIProgressPanel` for any AI call expected to take ≥ 10 seconds.

Rules:

- Stage label is **verb-led** ("Analyzing requirements…", "Generating architecture…")
- Show elapsed time + expected range
- **Hide the trigger UI** while the panel is running (disable or visually remove the "Generate" button)

```tsx
// bad — no progress feedback; button stays enabled
<Button onClick={runGeneration}>Generate PRD</Button>;

// good
{
  isGenerating ? (
    <AIProgressPanel stage="Generating PRD…" elapsedSeconds={elapsed} expectedRange="2–4 min" />
  ) : (
    <Button onClick={runGeneration}>Generate PRD</Button>
  );
}
```

## Auto-trigger generation on mount

For phase pages that need an AI call to be useful (e.g., the page is empty without one), auto-trigger the generation on mount.

Guard with an `autoGenerateAttempted` state flag to prevent infinite re-trigger loops:

```tsx
const [autoGenerateAttempted, setAutoGenerateAttempted] = useState(false);

useEffect(() => {
  if (!autoGenerateAttempted && !existingContent && !isGenerating) {
    setAutoGenerateAttempted(true);
    runGeneration();
  }
}, [autoGenerateAttempted, existingContent, isGenerating]);
```

Never trigger generation on every render; never re-trigger after the first attempt, even on error (show a manual retry button instead).

## Phase refetch on focus

On pages where another browser tab could mutate the same data (any shared phase page), refetch when the window regains focus or the document becomes visible:

```tsx
useEffect(() => {
  const handleFocus = () => refetch();
  const handleVisibility = () => {
    if (document.visibilityState === "visible") refetch();
  };
  window.addEventListener("focus", handleFocus);
  document.addEventListener("visibilitychange", handleVisibility);
  return () => {
    window.removeEventListener("focus", handleFocus);
    document.removeEventListener("visibilitychange", handleVisibility);
  };
}, [refetch]);
```
