---
name: Form Patterns
description: Form validation display, submit behavior, field anatomy, accessibility, and state management conventions; no form library.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

## No form library

Use direct `useState` per field for simple forms. For forms with > 5 fields or conditional logic, extract state into a custom hook (e.g., `use-step-editor.ts` pattern). Do not reach for `react-hook-form`, Formik, or similar libraries.

## Validation rules display

Render validation as an inline `{ test: boolean, label: string }` array with pass/fail icons shown from the moment a field is focused (not just on submit):

```tsx
const rules = [
  { test: password.length >= 8, label: "At least 8 characters" },
  { test: /[A-Z]/.test(password), label: "One uppercase letter" },
  { test: /[0-9]/.test(password), label: "One number" },
];

// Render
{
  isFocused && (
    <ul className="mt-2 space-y-1">
      {rules.map(({ test, label }) => (
        <li key={label} className="flex items-center gap-1.5 text-sm">
          {test ? (
            <CheckCircle2 className="text-status-success h-3.5 w-3.5" />
          ) : (
            <XCircle className="text-muted-foreground h-3.5 w-3.5" />
          )}
          <span className={test ? "text-status-success" : "text-muted-foreground"}>{label}</span>
        </li>
      ))}
    </ul>
  );
}
```

## Submit behavior

- Submit button is **disabled** until all validation rules pass — no submit-time validation flash
- Submit button shows a spinner + verb-led loading text while in flight; button stays disabled during flight

```tsx
const allValid = rules.every((r) => r.test);

<Button type="submit" disabled={!allValid || isSubmitting}>
  {isSubmitting ? (
    <>
      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
      Submitting…
    </>
  ) : (
    "Submit"
  )}
</Button>;
```

## Form error display

- One error message region per form (not inline per-field for server errors)
- Verb-first language: "Failed to save. Please try again." not "Error 500"
- Never wipe user input when a server error occurs
- Clear the error on the next field change

```tsx
const [serverError, setServerError] = useState<string | null>(null);

const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setValue(e.target.value);
  if (serverError) setServerError(null); // clear on next change
};

{
  serverError && (
    <p className="text-status-error-text text-sm" role="alert">
      {serverError}
    </p>
  );
}
```

## Field anatomy

Standard field visual spec:

- Label: `text-field-label` (maps to `text-sm font-medium text-foreground`)
- Input: `h-10 w-full rounded-lg border border-border bg-background px-3 text-sm`

```tsx
<div className="space-y-1.5">
  <label htmlFor="project-name" className="text-field-label">
    Project name
  </label>
  <input
    id="project-name"
    name="project-name"
    type="text"
    autoComplete="off"
    required
    className="border-border bg-background focus:ring-ring h-10 w-full rounded-lg border px-3 text-sm focus:ring-2 focus:outline-none"
  />
</div>
```

## Accessibility

Every form control must:

- Have a visible `<label>` linked via matching `htmlFor` / `id`
- Have `autoComplete` set to the correct value (e.g., `"email"`, `"current-password"`, `"off"`)
- Have `required` attribute set when the field is required

Placeholder text is not a substitute for a label. The `accessibility-aa` skill covers contrast and focus-indicator requirements.

## Complex form state

When a form has > 5 fields or conditional logic (steps, dependent fields, dynamic sections), extract all state into a custom hook:

```
// file: use-project-form.ts
export function useProjectForm(initialData?: Partial<Project>) {
  const [name, setName] = useState(initialData?.name ?? "");
  const [type, setType] = useState(initialData?.type ?? "");
  // ... validation, submit handler, dirty tracking
  return { name, setName, type, setType, canSubmit, handleSubmit, isSubmitting };
}
```

The page component then calls the hook and passes props down — it owns no form state directly.

## Server validation mirror

Every client-side validation rule must have a matching server-side Zod schema. The Zod schema is the source of truth for constraints (min/max lengths, regex, required):

```ts
// src/lib/api-schemas.ts
export const CreateProjectSchema = z.object({
  name: z.string().min(2).max(80),
  type: z.enum(["web", "mobile", "api"]),
});
```

The client-side `rules` array derives its thresholds from the same constants (or mirrors them explicitly). Never silently relax a rule on the client that the server enforces.

## No silent errors

Every async action (form submit, button click, background operation) must surface success or failure to the user. Use a toast, inline error, or banner — never silently swallow a rejection:

```tsx
// bad
await saveProject(data).catch(() => {});

// good
try {
  await saveProject(data);
  toast.success("Project saved.");
} catch {
  setServerError("Failed to save project. Please try again.");
}
```
