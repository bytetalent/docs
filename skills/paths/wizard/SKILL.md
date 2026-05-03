---
name: Wizard
description: Multi-step form with URL-driven step + field state, progress indicator, back/next nav, auto-advance on selection, and a terminal action — supports both create and edit modes.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 2
composes_with: [async-toast-mutation, server-client-page-split, form-patterns, optimistic-crud-with-etag]
nests: [modal-dialog-action]
conflicts_with: []
---

A wizard is a linear multi-step user journey. The PRD specifies what the steps are and what each step collects. This skill documents the code structure the code-gen agent should produce for any wizard regardless of domain content.

Wizards are dual-mode: **create** (new entity, POST) and **edit** (resume from an existing entity by ID, PUT + ETag). Both modes share the same wizard shell; the mode is determined by the presence of a query param (e.g., `?projectId=X`).

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #5.

## Core rules

- **All state is URL-encoded.** Step number AND all field values are stored as query params (`?step=N&type=X&name=Y&slug=Z&...`). Survives hard refresh. Enables direct-link edit mode.
- Each step is a separate component. The wizard shell renders only the active step component.
- Back/Next buttons live in a **fixed bottom nav footer** — always visible, not scrolled away. Product wizards use `DashboardPageLayout` (sidebar visible). Onboarding wizards use a standalone full-screen layout.
- The Next button is **disabled** until the current step's required fields are filled.
- **Auto-advance on card selection**: for selection steps (radio cards, type pickers), the wizard advances automatically on click. Next is the keyboard-user fallback.
- The final step is a review/confirm screen. Its "Next" is replaced by a "Create" (create mode) or "Save Changes" (edit mode) button.
- **Edit mode: confirm-gate before submit.** Edit wizards always gate the terminal action with a `ConfirmDialog` showing the pending changes (`pendingPayload`, `pendingResets`). See cross-reference to `modal-dialog-action`.
- Step components receive their current value + `onChange` as props — they do not manage wizard-level state.

## URL state contract

All wizard state lives in the URL. Example for a project wizard:

```
/space/projects/new?step=2&type=marketing&name=Launch+Campaign&clientId=abc123
```

Edit mode URL:

```
/space/projects/abc/edit?step=1&projectId=abc123&type=marketing&name=Launch+Campaign
```

The wizard hook reads all params on mount and initializes state from them. Writing back to the URL on every field change keeps the browser history clean — use `router.replace` for field updates to avoid polluting the Back button stack, and `router.push` only for step transitions.

## Wizard hook — URL-encoded state

```ts
// use-<entity>-wizard.ts
import { useRouter, useSearchParams } from "next/navigation";
import { useCallback } from "react";

export interface WizardState {
  // PRD-specified fields per step — all primitives (string | null)
  type: string;
  name: string;
  slug: string;
  clientId: string;
  // ... more fields from PRD
}

const TOTAL_STEPS = 3; // from PRD

// Derive initial state from URL params
function stateFromParams(searchParams: URLSearchParams): WizardState {
  return {
    type: searchParams.get("type") ?? "",
    name: searchParams.get("name") ?? "",
    slug: searchParams.get("slug") ?? "",
    clientId: searchParams.get("clientId") ?? "",
  };
}

// Serialize state back to URL params (merged with current params)
function paramsFromState(
  current: URLSearchParams,
  patch: Partial<WizardState>,
): string {
  const p = new URLSearchParams(current.toString());
  Object.entries(patch).forEach(([k, v]) => {
    if (v != null && v !== "") p.set(k, v);
    else p.delete(k);
  });
  return p.toString();
}

export function useEntityWizard() {
  const router = useRouter();
  const searchParams = useSearchParams();

  const step = Math.max(
    1,
    Math.min(TOTAL_STEPS, Number(searchParams.get("step") ?? 1)),
  );

  // Edit mode: presence of entityId in URL
  const entityId = searchParams.get("entityId") ?? null;
  const isEditMode = entityId !== null;

  const state = stateFromParams(searchParams);

  const setField = useCallback(
    <K extends keyof WizardState>(key: K, value: WizardState[K]) => {
      const qs = paramsFromState(searchParams, { [key]: value });
      router.replace(`?${qs}`);
    },
    [router, searchParams],
  );

  const canAdvance = (): boolean => {
    if (step === 1) return state.type.trim().length > 0;
    if (step === 2) return state.name.trim().length > 0 && state.slug.trim().length > 0;
    return true;
  };

  const next = () => {
    if (!canAdvance()) return;
    const qs = paramsFromState(searchParams, {});
    const p = new URLSearchParams(qs);
    p.set("step", String(step + 1));
    router.push(`?${p.toString()}`);
  };

  const back = () => {
    if (step <= 1) return;
    const p = new URLSearchParams(searchParams.toString());
    p.set("step", String(step - 1));
    router.push(`?${p.toString()}`);
  };

  // Auto-advance: call this from selection steps after the user picks a card.
  const advanceAfterSelect = <K extends keyof WizardState>(
    key: K,
    value: WizardState[K],
  ) => {
    const p = new URLSearchParams(searchParams.toString());
    p.set(key as string, String(value));
    p.set("step", String(step + 1));
    router.push(`?${p.toString()}`);
  };

  return {
    step,
    totalSteps: TOTAL_STEPS,
    state,
    setField,
    canAdvance,
    next,
    back,
    advanceAfterSelect,
    isEditMode,
    entityId,
  };
}
```

## Wizard shell component — create mode

```tsx
"use client";

export function EntityWizardPage() {
  const { step, totalSteps, state, setField, canAdvance, next, back } =
    useEntityWizard();
  const [submitting, setSubmitting] = useState(false);
  const router = useRouter();

  const handleCreate = async () => {
    setSubmitting(true);
    try {
      const res = await fetch("/api/entities", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(state),
      });
      if (!res.ok) throw new Error("Failed to create");
      const { id } = await res.json();
      appToast.success("Created successfully");
      router.push(`/space/entities/${id}`);
    } catch {
      appToast.error("Something went wrong");
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <DashboardPageLayout title="New Entity">
      {/* Header with progress */}
      <div className="border-b px-6 py-4">
        <WizardProgress current={step} total={totalSteps} />
      </div>

      {/* Step content */}
      <div className="flex-1 overflow-y-auto px-6 py-8 max-w-2xl mx-auto w-full">
        {step === 1 && (
          <StepOneTypeSelect
            value={state.type}
            onChange={(v) => setField("type", v)}
          />
        )}
        {step === 2 && (
          <StepTwoDetails
            name={state.name}
            slug={state.slug}
            onNameChange={(v) => setField("name", v)}
            onSlugChange={(v) => setField("slug", v)}
          />
        )}
        {step === 3 && <StepThreeReview state={state} />}
      </div>

      {/* Fixed bottom nav */}
      <div className="border-t px-6 py-4 flex justify-between bg-background">
        <Button variant="outline" onClick={back} disabled={step === 1}>
          Back
        </Button>
        {step < totalSteps ? (
          <Button onClick={next} disabled={!canAdvance()}>
            Next
          </Button>
        ) : (
          <Button onClick={handleCreate} disabled={submitting}>
            {submitting ? "Creating…" : "Create"}
          </Button>
        )}
      </div>
    </DashboardPageLayout>
  );
}
```

## Edit mode — hydration, ETag, and confirm-gate

Edit mode hydrates step fields from the existing entity on mount, captures the ETag, and requires a `ConfirmDialog` before submitting changes.

```tsx
"use client";
import { ConfirmDialog } from "@/components/shared/comp-confirm-dialog";

export function EntityEditWizardPage() {
  const { step, totalSteps, state, setField, canAdvance, next, back, entityId } =
    useEntityWizard();
  const router = useRouter();

  // Hydrate from existing entity on mount
  const [etag, setEtag] = useState<string | null>(null);
  useEffect(() => {
    if (!entityId) return;
    fetch(`/api/entities/${entityId}`, { method: "HEAD" }).then((r) => {
      setEtag(r.headers.get("ETag"));
    });
  }, [entityId]);

  // Confirm-gate state
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [pendingPayload, setPendingPayload] = useState<typeof state | null>(null);
  const [saving, setSaving] = useState(false);

  const requestSave = () => {
    setPendingPayload({ ...state });
    setConfirmOpen(true);
  };

  const handleConfirmSave = async () => {
    if (!pendingPayload || !entityId) return;
    setSaving(true);
    try {
      const res = await fetch(`/api/entities/${entityId}`, {
        method: "PUT",
        headers: {
          "Content-Type": "application/json",
          ...(etag ? { "If-Match": etag } : {}),
        },
        body: JSON.stringify(pendingPayload),
      });
      if (res.status === 412) throw new Error("Conflict — reload and try again");
      if (!res.ok) throw new Error("Failed to save");
      appToast.success("Saved");
      router.push(`/space/entities/${entityId}`);
    } catch (err) {
      appToast.error(err instanceof Error ? err.message : "Something went wrong");
    } finally {
      setSaving(false);
      setConfirmOpen(false);
    }
  };

  return (
    <>
      <DashboardPageLayout title="Edit Entity">
        <div className="border-b px-6 py-4">
          <WizardProgress current={step} total={totalSteps} />
        </div>

        <div className="flex-1 overflow-y-auto px-6 py-8 max-w-2xl mx-auto w-full">
          {/* ... same step components as create mode */}
        </div>

        <div className="border-t px-6 py-4 flex justify-between bg-background">
          <Button variant="outline" onClick={back} disabled={step === 1}>
            Back
          </Button>
          {step < totalSteps ? (
            <Button onClick={next} disabled={!canAdvance()}>
              Next
            </Button>
          ) : (
            <Button onClick={requestSave} disabled={saving}>
              Save Changes
            </Button>
          )}
        </div>
      </DashboardPageLayout>

      <ConfirmDialog
        open={confirmOpen}
        onClose={() => setConfirmOpen(false)}
        onConfirm={handleConfirmSave}
        title="Save Changes"
        description="Review your changes before saving."
        consequences={[
          `Name: ${pendingPayload?.name}`,
          `Type: ${pendingPayload?.type}`,
        ]}
        confirmLabel="Save"
        severity="info"
        loading={saving}
      />
    </>
  );
}
```

## Auto-advance on card selection

For selection steps (type pickers, radio cards), call `advanceAfterSelect` instead of `setField + next`:

```tsx
// Step 1 — type selection, auto-advances on pick
export function StepOneTypeSelect({
  value,
  onSelect,
}: {
  value: string;
  onSelect: (v: string) => void; // wizard passes advanceAfterSelect("type", v)
}) {
  return (
    <div className="space-y-4">
      <h2 className="text-xl font-semibold">What type of entity?</h2>
      <div className="grid grid-cols-2 gap-3">
        {TYPE_OPTIONS.map((opt) => (
          <button
            key={opt.value}
            onClick={() => onSelect(opt.value)} // auto-advances
            className={cn(
              "border rounded-lg p-4 text-left hover:border-primary",
              value === opt.value && "border-primary bg-primary/5",
            )}
          >
            <span className="font-medium">{opt.label}</span>
          </button>
        ))}
      </div>
      <p className="text-xs text-muted-foreground">
        You can also press Next to continue with your selection.
      </p>
    </div>
  );
}

// In wizard shell, wire it up:
{step === 1 && (
  <StepOneTypeSelect
    value={state.type}
    onSelect={(v) => advanceAfterSelect("type", v)}
  />
)}
```

## Cross-entity prefetch on mount

Wizards often need related data on mount (client list, current user, etc.). Fetch in the wizard hook or in a server component passed as a prop:

```ts
// In useEntityWizard or a separate hook called from the wizard shell
const [clients, setClients] = useState<Client[]>([]);
useEffect(() => {
  fetch("/api/clients").then((r) => r.json()).then(setClients);
}, []);
```

## Layout variants

| Context | Layout |
|---|---|
| Product wizard (inside dashboard) | `DashboardPageLayout` with sidebar visible + fixed-bottom nav footer |
| Onboarding flow (no auth context) | Standalone full-screen layout (no sidebar) |

```tsx
// Standalone layout for onboarding
export default function OnboardingWizardPage() {
  return (
    <main className="min-h-screen bg-background">
      <EntityWizardPage />
    </main>
  );
}
```

## Progress indicator

```tsx
function WizardProgress({
  current,
  total,
}: {
  current: number;
  total: number;
}) {
  return (
    <div className="flex items-center gap-2">
      {Array.from({ length: total }, (_, i) => i + 1).map((n) => (
        <div key={n} className="flex items-center gap-2">
          <div
            className={cn(
              "w-7 h-7 rounded-full flex items-center justify-center text-xs font-medium",
              n < current
                ? "bg-primary text-primary-foreground"
                : n === current
                  ? "border-2 border-primary text-primary"
                  : "border border-muted text-muted-foreground",
            )}
          >
            {n < current ? <Check className="w-3 h-3" /> : n}
          </div>
          {n < total && (
            <div
              className={cn(
                "h-px w-8",
                n < current ? "bg-primary" : "bg-muted",
              )}
            />
          )}
        </div>
      ))}
    </div>
  );
}
```

## What NOT to do

```tsx
// Bad — step state in local useState with no URL sync
const [step, setStep] = useState(1); // browser Back broken, no deep link

// Bad — field state in local useState (not URL-encoded)
const [state, setState] = useState<WizardState>({ type: "", name: "" });
// Use URL params so state survives hard refresh and edit mode works.

// Bad — auto-advance not implemented for selection steps
<TypeCard onClick={() => setField("type", v)} />
// Missing the router.push to next step after selection.

// Bad — edit mode submits without confirm-gate
const handleSave = async () => {
  await fetch(`/api/entities/${id}`, { method: "PUT", body: JSON.stringify(state) });
  // No ConfirmDialog — always gate edit-mode terminal actions.
};

// Bad — terminal action not on the last step
<Button onClick={handleSubmit}>Create</Button>; // on step 2 of 4

// Bad — full-screen standalone layout for a product wizard inside dashboard
<main className="min-h-screen"><WizardPage /></main>
// Product wizards use DashboardPageLayout so the sidebar stays visible.
```

## Related skills

- `skills/paths/async-toast-mutation` — the submit handler frame
- `skills/paths/modal-dialog-action` — `ConfirmDialog` used for edit-mode confirm-gate; see `loading` prop and `severity` vocab
- `skills/paths/optimistic-crud-with-etag` — ETag capture + If-Match header pattern for edit mode
- `skills/form-patterns` — validation within each step
