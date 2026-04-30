# Forms — Standards

Cross-stack form patterns. Validation rules, no-submit-flash discipline, custom-hook ownership of complex form state, accessibility.

Stack-specific form primitives (React + Tailwind, Astro forms, ASP.NET Razor inputs) live in each base/product's overlay.

---

## 1. No Form Library by Default

Bytetalent webapps don't use a form library (no `react-hook-form`, no Formik). Forms use:

- Direct `useState` per field, OR
- A custom hook that owns all form state for complex flows (`use-step-editor.ts`, `use-decision-editor.ts`)

The principle is **clarity over indirection**. Most forms are simple enough that a library adds boilerplate without saving anything. For complex multi-step or conditional forms, an extracted custom hook keeps `page-client.tsx` clean without pulling in a library.

If you find yourself implementing `react-hook-form` features by hand (deep validation, field arrays, dirty tracking), reconsider — but at that point use Zod's `parse` + a thin custom hook, not a third-party library.

---

## 2. Validation Rules — Inline Visible

Validation rules are expressed as arrays of `{ test, label }` objects rendered inline with pass/fail icons:

```tsx
const passwordRules = [
  { test: (p: string) => p.length >= 12, label: "At least 12 characters" },
  { test: (p: string) => /[A-Z]/.test(p), label: "An uppercase letter" },
  { test: (p: string) => /[0-9]/.test(p), label: "A number" },
  { test: (p: string) => /[^A-Za-z0-9]/.test(p), label: "A special character" },
];

// In the JSX:
<ul className="mt-2 space-y-1">
  {passwordRules.map((rule, i) => {
    const passed = rule.test(password);
    return (
      <li key={i} className="flex items-center gap-2 text-xs">
        {passed ? <Check className="text-status-success-text" /> : <X className="text-muted-foreground" />}
        <span className={passed ? "text-status-success-text" : "text-muted-foreground"}>
          {rule.label}
        </span>
      </li>
    );
  })}
</ul>
```

### Rules

- **Submit buttons are disabled until all rules pass** — no submit-time validation flash, no "click submit, wait, see red errors appear."
- **Show rules from the moment the field is focused** — they're a guide, not a punishment.
- **Pass/fail icons** — visual recognition is faster than text for repeat users.
- **Validation logic lives in `src/lib/`** — `password-rules.ts`, `phone.ts`, etc. Reusable across signup, change-password, profile edit.

---

## 3. Field Anatomy

Standard text input field:

```tsx
<div className="space-y-1.5">
  <label htmlFor="email" className="text-field-label">
    Email
  </label>
  <input
    id="email"
    type="email"
    value={email}
    onChange={(e) => setEmail(e.target.value)}
    className="h-10 w-full rounded-lg border border-border bg-background px-3 text-sm text-foreground placeholder:text-muted-foreground focus:border-primary focus:outline-none focus:ring-1 focus:ring-primary"
    placeholder="you@example.com"
    required
    autoComplete="email"
  />
  {/* Validation rules go below the input, not above */}
</div>
```

### Tokens

- Label: `text-field-label` (utility class — `text-[11px] font-medium uppercase tracking-wide text-muted-foreground`)
- Input: `h-10 w-full rounded-lg border border-border bg-background px-3 text-sm`
- Focus: `focus:border-primary focus:outline-none focus:ring-1 focus:ring-primary`
- Disabled: `disabled:opacity-50 disabled:cursor-not-allowed`
- Error state: `border-status-error-text` (only when validation has run after submit attempt)

### Rules

- **Every form control has an accessible name** — visible `<label>` linked via `htmlFor` / `id`, or `aria-label` only when no visible label is possible
- **`autoComplete` attribute is set correctly** — `email`, `current-password`, `new-password`, `tel`, `name`, etc. Helps password managers.
- **`required` attribute is set** when the field is required (HTML5 native validation as the floor; JS rules as the ceiling)

---

## 4. Submit Button States

| State | Style | When |
|---|---|---|
| Disabled (rules failing) | `disabled:opacity-50` | Validation rules not all passing |
| Active | `default` button variant | Rules passing, ready to submit |
| Loading | Spinner inside button + text "Submitting…" / verb-led | Submit in flight |
| Success | Brief green flash + reset | Submit succeeded (when not navigating away) |

**Loading state pattern:**

```tsx
<Button disabled={!canSubmit || submitting}>
  {submitting ? (
    <>
      <Loader2 className="animate-spin" /> Submitting…
    </>
  ) : (
    "Sign in"
  )}
</Button>
```

---

## 5. Error Surfacing

Form-level errors (not field-level rules):

```tsx
{formError && (
  <div className="rounded-lg border border-status-error-text bg-status-error-bg p-3 text-sm text-status-error-text">
    {formError}
  </div>
)}
```

### Rules

- **One error message location per form** — not multiple scattered alerts
- **Verb-first language** — "Couldn't reach the server" not "Server unreachable"
- **Don't wipe user input on error** — the user shouldn't have to retype after a network blip
- **Clear the error as soon as the user changes any field** — they're trying again

---

## 6. Complex Form State — Custom Hooks

For multi-step forms, conditional field visibility, or forms with > 5 fields, extract state into a custom hook:

```typescript
// src/hooks/use-step-editor.ts
export function useStepEditor(initialStep: Step | null) {
  const [name, setName] = useState(initialStep?.name ?? "");
  const [actionType, setActionType] = useState<ActionType>(initialStep?.actionType ?? "manual");
  const [config, setConfig] = useState<StepConfig>(initialStep?.config ?? {});
  // ... more fields

  const isDirty = useMemo(() => /* compare to initialStep */, [/* deps */]);
  const isValid = useMemo(() => /* run all rules */, [name, actionType, config]);

  return {
    fields: { name, actionType, config /* ... */ },
    setters: { setName, setActionType, setConfig /* ... */ },
    isDirty,
    isValid,
    reset: () => /* reset to initial */,
  };
}
```

This keeps `page-client.tsx` clean and lets the form be tested in isolation.

---

## 7. Server-side Validation Mirror

Every client-side validation rule must be mirrored on the server with the same Zod schema:

```typescript
// src/lib/api-schemas.ts — used by client + server
export const SignupSchema = z.object({
  email: z.string().email().max(254),
  password: z.string()
    .min(12)
    .regex(/[A-Z]/, "Must contain uppercase")
    .regex(/[0-9]/, "Must contain digit")
    .regex(/[^A-Za-z0-9]/, "Must contain special character"),
});
```

Client-side rules are convenience — they prevent useless submissions. Server-side validation is the security boundary.

---

## 8. Accessibility

- Use semantic HTML before ARIA
- Every form control has a visible label or `aria-label`
- Keyboard navigation works — Tab order is logical, Enter submits when in a single-field form
- Focus visible — never `outline: none` without replacing the focus indicator
- Error messages are announced — use `aria-live="polite"` on the form-error region

---

## 9. Forbidden Patterns

- **No `window.confirm()`** for destructive actions — use `<ConfirmDialog>` (see [`guide-design.md`](guide-design.md) §12)
- **No client-side-only validation for security-sensitive constraints** (password complexity, email allowlists, etc.) — always mirror server-side
- **No silently swallowing errors** — every async action surfaces success or failure to the user
- **No "click again to confirm" anti-patterns** — explicit confirmation via dialog with consequences listed

---

## Related

- [`guide-design.md`](guide-design.md) — input tokens, button states, ConfirmDialog
- [`guide-api.md`](guide-api.md) — server-side validation pattern, error response shape
- [`guide-code.md`](guide-code.md) — TypeScript discipline, no-`any` rule
