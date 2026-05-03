---
name: Wizard
description: Multi-step form with URL-driven step state, progress indicator, back/next nav, and a terminal action — PRD defines the steps and per-step content.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

A wizard is a linear multi-step user journey. The PRD specifies what the steps are and what each step collects. This skill documents the code structure the code-gen agent should produce for any wizard regardless of domain content.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #5.

## Core rules

- Step state is **URL-driven**: the current step is stored in a `?step=<n>` query param so the browser Back button works and deep links are shareable. Use `useSearchParams` / `useRouter`.
- Each step is a separate component. The wizard shell renders only the active step component.
- Back/Next buttons live in a **fixed bottom nav bar** — always visible, not scrolled away with content.
- The Next button is **disabled** until the current step's required fields are filled.
- The final step's "Next" is replaced by a "Create" / "Submit" / action label (from the PRD).
- A step progress indicator (numbered steps or a stepper bar) lives in the page header.
- Step components receive their current value + `onChange` as props — they do not manage wizard-level state.
- Wizard state (collected form data) lives in the wizard hook, not spread across step components.

## Wizard hook shape

```ts
// use-<entity>-wizard.ts
import { useRouter, useSearchParams } from "next/navigation";
import { useState } from "react";

export interface WizardState {
  // PRD-specified fields per step
  stepOneField: string;
  stepTwoField: string;
  // ...
}

const TOTAL_STEPS = 3; // from PRD

export function useEntityWizard() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const step = Math.max(
    1,
    Math.min(TOTAL_STEPS, Number(searchParams.get("step") ?? 1)),
  );

  const [state, setState] = useState<WizardState>({
    stepOneField: "",
    stepTwoField: "",
  });

  const setField = <K extends keyof WizardState>(
    key: K,
    value: WizardState[K],
  ) => {
    setState((prev) => ({ ...prev, [key]: value }));
  };

  const canAdvance = (): boolean => {
    if (step === 1) return state.stepOneField.trim().length > 0;
    if (step === 2) return state.stepTwoField.trim().length > 0;
    return true;
  };

  const next = () => {
    if (!canAdvance()) return;
    router.push(`?step=${step + 1}`);
  };

  const back = () => {
    if (step > 1) router.push(`?step=${step - 1}`);
  };

  return {
    step,
    totalSteps: TOTAL_STEPS,
    state,
    setField,
    canAdvance,
    next,
    back,
  };
}
```

## Wizard shell component

```tsx
"use client";

export function EntityWizardPage() {
  const { step, totalSteps, state, setField, canAdvance, next, back } =
    useEntityWizard();
  const [submitting, setSubmitting] = useState(false);
  const router = useRouter();

  const handleSubmit = async () => {
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
    <div className="flex flex-col min-h-screen">
      {/* Header with progress */}
      <div className="border-b px-6 py-4">
        <WizardProgress current={step} total={totalSteps} />
      </div>

      {/* Step content */}
      <div className="flex-1 overflow-y-auto px-6 py-8 max-w-2xl mx-auto w-full">
        {step === 1 && (
          <StepOne
            value={state.stepOneField}
            onChange={(v) => setField("stepOneField", v)}
          />
        )}
        {step === 2 && (
          <StepTwo
            value={state.stepTwoField}
            onChange={(v) => setField("stepTwoField", v)}
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
          <Button onClick={handleSubmit} disabled={submitting}>
            {submitting ? "Creating…" : "Create"}
          </Button>
        )}
      </div>
    </div>
  );
}
```

## Step component shape

Each step receives typed props — no global state access inside step components:

```tsx
interface StepOneProps {
  value: string;
  onChange: (value: string) => void;
}

export function StepOne({ value, onChange }: StepOneProps) {
  return (
    <div className="space-y-4">
      <h2 className="text-xl font-semibold">Step title from PRD</h2>
      <p className="text-muted-foreground">Step description from PRD.</p>
      <Input
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder="Enter value"
      />
    </div>
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

## Standalone full-screen layout variant

Some wizards run outside the dashboard layout (no sidebar) for onboarding flows:

```tsx
// page.tsx — use a standalone layout instead of DashboardPageLayout
export default function WizardPage() {
  return (
    <main className="min-h-screen bg-background">
      <EntityWizardPage />
    </main>
  );
}
```

## What NOT to do

```tsx
// Bad — step state in local useState with no URL sync
const [step, setStep] = useState(1); // browser Back doesn't work, no deep link

// Bad — step components manage their own fetch/state
export function StepOne() {
  const [options, setOptions] = useState([]); // fetch should be in wizard hook or step's own effect, not mixed with form
}

// Bad — terminal action not on the last step
<Button onClick={handleSubmit}>Create</Button>; // on step 2 of 4 — always put on last step only
```

## Related skills

- `skills/paths/async-toast-mutation` — the submit handler frame
- `skills/paths/modal-dialog-action` — for confirmation before submit if needed
- `skills/form-patterns` — validation within each step
