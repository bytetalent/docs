---
name: Async Toast Mutation
description: Every write action follows set-busy → fetch → show toast → clear-busy — with appToast as the single toast path, separate boolean per action.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Every mutation in the app (save, delete, archive, approve, generate) follows the same frame: set an in-flight boolean, call fetch, show a success or error toast, clear the in-flight boolean. `appToast.*` is the single toast path.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #20.

## Core rules

- Every async write action has its own `boolean` state: `saving`, `deleting`, `archiving`, `approving`, `generating`. Never reuse a single `loading` boolean for multiple actions.
- The in-flight boolean is set `true` before the `fetch` call and cleared `false` in the `finally` block — always, even on error.
- Use `appToast.success(...)` for success, `appToast.error(...)` for failures. Never call `sonner`, `toast()`, or `react-hot-toast` directly.
- Buttons that trigger mutations are `disabled` when their in-flight boolean is `true`.
- Button label changes to reflect in-flight state: `"Saving…"` while saving, `"Deleting…"` while deleting.
- Mutations that navigate on success call `router.push()` after the toast, inside the try block.

## Standard save handler

```tsx
"use client";

export function EntityDetailPage({ entityId }: { entityId: string }) {
  const [entity, setEntity] = useState<Entity | null>(null);
  const [saving, setSaving] = useState(false);

  const handleSave = async () => {
    if (!entity) return;
    setSaving(true);
    try {
      const res = await fetch(`/api/entities/${entityId}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ name: entity.name }),
      });
      if (!res.ok) throw new Error("Save failed");
      const updated = await res.json();
      setEntity(updated);
      appToast.success("Saved");
    } catch {
      appToast.error("Failed to save. Please try again.");
    } finally {
      setSaving(false);
    }
  };

  return (
    <Button onClick={handleSave} disabled={saving}>
      {saving ? "Saving…" : "Save"}
    </Button>
  );
}
```

## Delete handler (navigates on success)

```tsx
const [deleting, setDeleting] = useState(false);

const handleDelete = async () => {
  setDeleting(true);
  try {
    const res = await fetch(`/api/entities/${entityId}`, { method: "DELETE" });
    if (!res.ok) throw new Error("Delete failed");
    appToast.success("Deleted");
    router.push("/space/entities");
  } catch {
    appToast.error("Failed to delete");
  } finally {
    setDeleting(false);
  }
};
```

## Multiple independent mutations on one page

```tsx
// Good — separate boolean per action
const [saving, setSaving] = useState(false);
const [archiving, setArchiving] = useState(false);
const [approving, setApproving] = useState(false);

// Each button disabled by its own flag
<Button onClick={handleSave} disabled={saving}>{saving ? "Saving…" : "Save"}</Button>
<Button onClick={handleArchive} disabled={archiving} variant="outline">
  {archiving ? "Archiving…" : "Archive"}
</Button>
<Button onClick={handleApprove} disabled={approving} variant="default">
  {approving ? "Approving…" : "Approve"}
</Button>
```

## Optimistic update variant

For mutations where the server response is predictable, update local state optimistically and roll back on error:

```tsx
const handleToggleActive = async (entityId: string) => {
  // Optimistic: flip locally
  setItems((prev) =>
    prev.map((item) =>
      item.id === entityId ? { ...item, isActive: !item.isActive } : item,
    ),
  );

  try {
    await fetch(`/api/entities/${entityId}/toggle`, { method: "POST" });
    appToast.success("Updated");
  } catch {
    // Rollback
    setItems((prev) =>
      prev.map((item) =>
        item.id === entityId ? { ...item, isActive: !item.isActive } : item,
      ),
    );
    appToast.error("Failed to update");
  }
};
```

## appToast API

```tsx
// src/components/shared/comp-toast.tsx
import { appToast } from "@/components/shared/comp-toast";

appToast.success("Operation succeeded");
appToast.error("Something went wrong");
appToast.info("Note: changes will take effect shortly");

// Good
appToast.success("Saved");

// Bad — calling sonner directly
import { toast } from "sonner";
toast.success("Saved"); // Use appToast instead
```

## What NOT to do

```tsx
// Bad — single loading flag for multiple actions
const [loading, setLoading] = useState(false);
const handleSave = async () => {
  setLoading(true);
  /* ... */ setLoading(false);
};
const handleDelete = async () => {
  setLoading(true);
  /* ... */ setLoading(false);
};
// Can't tell which action is in flight; Save and Delete buttons both disable together
```

```tsx
// Bad — no finally block (loading state can get stuck on error)
const handleSave = async () => {
  setSaving(true);
  try {
    await fetch(/* ... */);
    appToast.success("Saved");
  } catch {
    appToast.error("Failed");
    // Missing: setSaving(false) — button stays disabled forever on error
  }
};
```

```tsx
// Bad — error swallowed, no toast
try {
  await fetch(`/api/entities/${entityId}`, {
    method: "PUT",
    body: JSON.stringify(data),
  });
} catch {
  // Nothing — user has no feedback that the save failed
}
```

## Related skills

- `skills/paths/modal-dialog-action` — confirm dialogs that call these mutation handlers
- `skills/paths/optimistic-crud-with-etag` — adds ETag/If-Match to the save handler
- `skills/paths/fetch-error-retry` — for initial load failures (not mutation failures)
