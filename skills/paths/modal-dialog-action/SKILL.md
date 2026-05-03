---
name: Modal Dialog Action
description: Overlay-based action pattern — ConfirmDialog for destructive/approval gates, slide-over panel for multi-field forms, read-only modal for contextual detail.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
composes_with: [async-toast-mutation, soft-delete-api-resource]
nests: []
conflicts_with: []
---

Interactive overlays block the main content while collecting a user decision or displaying contextual detail. This pattern covers all modal shapes: destructive confirmation, data-entry slide-over, and read-only informational modal.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #18.

## Core rules

- Use `ConfirmDialog` (from `@/components/shared/comp-confirm-dialog`) for all destructive, approval, and irreversible-action gates.
- Use a slide-over `Sheet` for multi-field data entry forms that shouldn't navigate away from the page.
- Use a standard `Dialog` for read-only contextual modals (version history, preview).
- Every modal has an `open: boolean` prop and an `onClose` callback — controlled by parent state.
- Destructive confirms use `severity="destructive"`. Approvals use `severity="success"`. Informational uses `severity="info"`.
- `ConfirmDialog` always shows: title, description, optional `consequences` bullet list, and a confirm button disabled while the async action is in flight.
- Never use `window.confirm()` or `alert()` for destructive actions.

## ConfirmDialog shape

```tsx
interface ConfirmDialogProps {
  open: boolean;
  onClose: () => void;
  onConfirm: () => Promise<void>;
  title: string;
  description: string;
  consequences?: string[];
  confirmLabel?: string;
  severity?: "destructive" | "success" | "info";
}
```

## Destructive confirmation usage

```tsx
function ArchiveEntityButton({ entityId }: { entityId: string }) {
  const [showConfirm, setShowConfirm] = useState(false);
  const [archiving, setArchiving] = useState(false);

  const handleArchive = async () => {
    setArchiving(true);
    try {
      await fetch(`/api/entities/${entityId}`, { method: "DELETE" });
      appToast.success("Archived");
      router.push("/space/entities");
    } catch {
      appToast.error("Failed to archive");
    } finally {
      setArchiving(false);
      setShowConfirm(false);
    }
  };

  return (
    <>
      <Button variant="destructive" onClick={() => setShowConfirm(true)}>
        Archive
      </Button>

      <ConfirmDialog
        open={showConfirm}
        onClose={() => setShowConfirm(false)}
        onConfirm={handleArchive}
        title="Archive Entity"
        description="This entity will be hidden from your workspace."
        consequences={[
          "All associated records will become inaccessible",
          "This action can be reversed by contacting support",
        ]}
        confirmLabel="Archive"
        severity="destructive"
      />
    </>
  );
}
```

## Approval confirmation usage

```tsx
<ConfirmDialog
  open={showApprove}
  onClose={() => setShowApprove(false)}
  onConfirm={handleApprove}
  title="Approve PRD"
  description="Approving will lock this version and unblock the next phase."
  consequences={[
    "Design phase will become available",
    "PRD content will be versioned and cannot be regenerated without a new version",
  ]}
  confirmLabel="Approve"
  severity="success"
/>
```

## Slide-over panel (multi-field data entry)

```tsx
import {
  Sheet,
  SheetContent,
  SheetHeader,
  SheetTitle,
} from "@/components/ui/sheet";

function AddConnectionSheet({
  open,
  onClose,
  onAdded,
}: {
  open: boolean;
  onClose: () => void;
  onAdded: () => void;
}) {
  const [name, setName] = useState("");
  const [apiKey, setApiKey] = useState("");
  const [saving, setSaving] = useState(false);

  const handleSave = async () => {
    setSaving(true);
    try {
      await fetch("/api/connections", {
        method: "POST",
        body: JSON.stringify({ name, apiKey }),
      });
      appToast.success("Connection added");
      onAdded();
      onClose();
    } catch {
      appToast.error("Failed to add connection");
    } finally {
      setSaving(false);
    }
  };

  return (
    <Sheet open={open} onOpenChange={(o) => !o && onClose()}>
      <SheetContent>
        <SheetHeader>
          <SheetTitle>Add Connection</SheetTitle>
        </SheetHeader>
        <div className="space-y-4 mt-4">
          <div>
            <Label>Name</Label>
            <Input value={name} onChange={(e) => setName(e.target.value)} />
          </div>
          <div>
            <Label>API Key</Label>
            <Input
              type="password"
              value={apiKey}
              onChange={(e) => setApiKey(e.target.value)}
            />
          </div>
          <Button onClick={handleSave} disabled={saving} className="w-full">
            {saving ? "Saving…" : "Save"}
          </Button>
        </div>
      </SheetContent>
    </Sheet>
  );
}
```

## Read-only modal (contextual detail)

```tsx
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";

function VersionHistoryModal({
  open,
  onClose,
  versions,
}: {
  open: boolean;
  onClose: () => void;
  versions: Version[];
}) {
  return (
    <Dialog open={open} onOpenChange={(o) => !o && onClose()}>
      <DialogContent className="max-w-2xl">
        <DialogHeader>
          <DialogTitle>Version History</DialogTitle>
        </DialogHeader>
        <div className="space-y-2 max-h-96 overflow-y-auto">
          {versions.map((v) => (
            <div
              key={v.id}
              className="flex justify-between text-sm border-b py-2"
            >
              <span>Version {v.version}</span>
              <span className="text-muted-foreground">
                {formatDate(v.createdAt)}
              </span>
            </div>
          ))}
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

## What NOT to do

```tsx
// Bad — window.confirm for destructive action
const ok = window.confirm("Archive this entity?");
if (ok) handleArchive();

// Bad — modal open state managed globally (context or Zustand)
// Each modal's open state is local to its parent component
const { openModal } = useModalStore();

// Bad — ConfirmDialog without consequences for downstream-impacting action
<ConfirmDialog title="Approve PRD" description="Are you sure?" />;
// Include consequences bullets when the action has non-obvious downstream effects
```

## Related skills

- `skills/paths/async-toast-mutation` — the onConfirm handler frame
- `skills/paths/soft-delete-api-resource` — DELETE route called by destructive confirms
