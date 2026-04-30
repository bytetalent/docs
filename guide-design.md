# Design Standards

Cross-stack visual design and UX standards. Tokens, typography, theming, stat-card anatomy, marketing-page spacing, and CTA discipline.

Stack-specific component implementations (React + Tailwind + shadcn vs. Astro + Tailwind vs. native iOS) live in each base or product's `docs/` overlay.

---

# Part 1 — Design System

## 1. Design Tokens

Design tokens are CSS custom properties defined in the project's `globals.css` (or stack equivalent) using a `@theme` block in Tailwind v4+.

Token categories every project ships with:

- **Base:** `--background`, `--foreground`, `--primary`, `--border`, `--radius`
- **Brand surfaces:** `--brand-surface`, `--brand-dark-border`, etc. (project-specific brand)
- **Status:** success, info, warning, error, neutral — each with `*-bg` and `*-text` variants
- **Accents:** purple, amber, blue, pink, green, cyan
- **Typography utilities:** `text-section-heading`, `text-field-label`, `text-table-header`, `text-xs-plus`, `text-auth-heading`
- **Per-feature tokens:** scoped to specific surfaces (e.g., `--turbo-canvas`, `--turbo-card` for a flow editor)

**Rule:** charts and dynamic inline styles must use CSS variables only — no hex fallbacks. Use `var(--color-primary)`, not `var(--color-primary, #96c237)`.

## 2. Typography

**Default fonts** for web stacks: Inter (sans) + JetBrains Mono (mono), loaded via the framework's font loader and applied as CSS variables `--font-sans` / `--font-mono`.

### Icon System

**Lucide React** (or stack-equivalent icon set) for all UI icons. Custom brand/integration icons live in `src/components/icons/` as React components or as inline SVG in Astro.

Icon styling per entity type is resolved by helper functions:

- `getToolIcon(type)`, `getChannelTypeIcon(type)`, `getModelIcon(type)`, etc.
- Each returns `{ icon, iconColor, iconBg }` for consistent badge rendering.

## 3. Theming — Light / Dark / System

Theme is controlled by toggling the `.dark` class on `<html>`. All color tokens have dark-mode overrides defined in `globals.css` under `.dark { }`.

### Three sources of truth

Reconciled in the `useTheme` hook:

1. `localStorage` (highest priority — user's last explicit choice)
2. Auth-provider user metadata (persisted to the user account; e.g., Clerk `unsafeMetadata.theme`)
3. `window.matchMedia("prefers-color-scheme")` (system fallback)

### Flash prevention

A `theme-init.js` script runs `beforeInteractive` in the root layout to apply the correct class before first paint — prevents flash of wrong theme.

Cross-tab sync handled via `storage` event listener.

## 4. Stat Card Anatomy

Stat cards surface key entity metrics on dashboard and list pages. Apply these rules to every stat card.

**Layout:**
- Title label: upper-left, `text-[10px] font-bold uppercase tracking-wider text-muted-foreground`
- Primary value: `text-3xl font-extrabold` (fontSize 32, weight 800), heading color
- Context text: inline with primary, entity accent color, `text-xl font-semibold` (fontSize 20, weight 600) — e.g. `/ 50 configured`
- Progress bar: `h-2 rounded-full bg-muted` with filled inner bar in entity color, immediately above the stats text line
- Stats text: bottom of card, `text-[11px] text-muted-foreground` — contextual summary (e.g., "455 complete this month")

**Interaction:**
- The whole card is the click target — navigates to the filtered list page
- **No CTA button inside stat cards** — selecting the card achieves navigation; a "Manage →" button is redundant noise

**Value + context alignment:**
- Primary value and context text share a horizontal row (`h-11 alignItems:center`) for vertical alignment regardless of font size differences
- `justifyContent: start` on the title row — no spacer or button needed

**Spacing:**
- Card padding: `px-4 py-3` (16px / 12px)
- Gap between top block and bottom block: `space-between` on the card's vertical layout
- Gap within value row: `gap-1.5` between primary value and context text

## 5. UI Component Patterns

Established patterns to follow:

| Pattern | Notes |
|---|---|
| Collapsible chart card | ChevronDown toggle, `collapsed` state, body + footer hidden when collapsed |
| Dark chart panel | `bg-brand-dark` card with `theme="dark"` controls; separated from light tables card below |
| Slide-in edit panel | `fixed right-0 top-0 h-full w-[380px]` with backdrop overlay, `flex flex-col` form |
| Detail page config + canvas | Config section emits `onCollapsedChange`; parent adjusts canvas height |
| Stat card grid | Collapsible via "Overview" toggle; card = `<Link>`; no CTA buttons inside |

---

# Part 2 — Tables

Standard component contracts and tokens for data tables in dashboards. See [`guide-tables.md`](guide-tables.md) for the full spec — pagination, sorting, filtering, FilterActionBar, SortableTableHeader, PaginationBar, and cell content templates.

---

# Part 3 — Marketing / Landing Pages

## 6. Vertical Spacing Scale

| Section type | Mobile | Desktop | Tailwind |
|---|---|---|---|
| Hero section | 80px | 96px | `py-20 md:py-24` |
| Primary body section | 64px | 80px | `py-16 md:py-20` |
| Secondary/tight section | 48px | 64px | `py-12 md:py-16` |
| CTA section | 64px | 80px | `py-16 md:py-20` |
| Footer | 40px | 48px | `py-10 md:py-12` |

**Rule:** Never use bare `py-20`, `py-24`, `py-28`, `py-32` — always the responsive pairs above. Prevents height shifts between breakpoints.

## 7. Hero Section Standards

All hero sections (standalone and carousel) must:

1. Use `py-20 md:py-24` on the outer `<section>` element
2. Constrain text/content area to a **fixed height** (`h-[Npx]` or `min-h-[Npx]`) — prevents shifts when carousel slides change
3. Never use a variable-height container for carousel content (use `min-h` on the inner content area)

### Standard Hero Heights

| Layout | Content min-height | Tailwind |
|---|---|---|
| Single-column centered | 320px | `min-h-[320px]` |
| Two-column split | 380px | `h-[380px]` |

### Max Widths

- Two-column layout: `max-w-[1200px]`
- Single-column centered: `max-w-[800px]`

## 8. Carousel Standards

### Preventing height/button jump

Fixed-height outer container, position absolute on inactive slides:

```tsx
<div className="relative h-[380px]">
  {slides.map((slide, i) => (
    <div
      key={i}
      className={cn(
        "transition-opacity duration-700",
        i === active ? "relative opacity-100" : "pointer-events-none absolute inset-0 opacity-0"
      )}
    >
      {/* slide content */}
    </div>
  ))}
</div>

{/* Dots — outside the fixed container */}
<div className="mt-8 flex gap-2">...</div>
```

### Auto-advance interval

Default: 7000ms (7s). Don't change without design review.

### Dots indicator

- Active: `w-7 bg-primary rounded-full h-2.5`
- Inactive: `w-2.5 bg-[#D4D6D9] rounded-full h-2.5 hover:bg-[#B0B3B8]`

(The hex inactive colors here should be replaced with theme tokens in production — they're called out as a known violation in code-review audits.)

## 9. CTA Discipline

### Approved label → destination pairs

| Label | Destination | When to use |
|---|---|---|
| `Get Started Free` | `ROUTES.SIGNUP` | Primary CTA (default) |
| `Get Early Access` | `ROUTES.SIGNUP` | Invite-only / waitlist period |
| `View Pricing` | `ROUTES.PRICING` | From any page |
| `Contact Sales` | `mailto:sales@<consultancy>` | Enterprise tier only |

### Banned labels

These must not be used unless an actual demo/video exists at the linked destination:

- ~~View a Demo~~
- ~~Book a Demo~~
- ~~Watch Demo~~
- ~~Schedule a Demo~~

### Button pairing rules

1. Every hero has at most **2 CTAs**: one primary (`default` variant), one secondary (`outline` or `outline-dark`).
2. Secondary CTA must link to a **different** page or anchor than the primary.
3. Both buttons must link to pages within the site — no external links in hero CTAs.
4. Primary is always `ROUTES.SIGNUP`; secondary is always another landing section.

### Variant selection

| Background | Primary | Secondary |
|---|---|---|
| Light / white | `default` | `outline` |
| Dark | `default` | `outline-dark` |

## 10. Navigation

- Sticky: `sticky top-0 z-50`
- Height: `h-[72px]` — do not change; determines page offset calculations
- Active link: determined by `pathname.startsWith(link.href)` — no manual override needed
- Nav CTA: always "Get Started Free" → `ROUTES.SIGNUP`
- Sign in link: always present to the left of the CTA button

## 11. Section Grid Layouts

| Columns | Mobile | Desktop | Tailwind |
|---|---|---|---|
| 2-column cards | 1 | 2 | `grid-cols-1 md:grid-cols-2` |
| 3-column cards | 1 | 3 | `grid-cols-1 md:grid-cols-3` |
| 4-column cards | 2 | 4 | `grid-cols-2 md:grid-cols-4` |
| Gap | 16px | 20px | `gap-5` |

Card border radius: `rounded-2xl`. Card padding: `p-7` (standard) or `p-8` (featured).

---

# Part 4 — Pipeline UX Patterns

Patterns for product surfaces where multi-step workflows hand off between human input and long-running AI calls.

## 12. ConfirmDialog — explain consequences

`comp-confirm-dialog.tsx` — replaces every `window.confirm()` with a structured modal that tells the user what's about to happen.

### When to use

Any non-trivial action — approval gates, archive, regenerate, destructive ops. Skip for trivial saves; toasts cover those.

### Structure

```tsx
<ConfirmDialog
  open={confirmApproveOpen}
  onClose={() => setConfirmApproveOpen(false)}
  onConfirm={handleApprove}
  title="Approve PRD & start Brand Kit?"
  description="One-paragraph summary above the bullets."
  consequences={[
    "PRD is saved as a versioned approved deliverable",
    "PRD phase is marked complete with today's timestamp",
    "Brand Kit phase moves from Not Started to In Progress",
  ]}
  severity="success"
  confirmLabel="Approve PRD"
  loading={approving}
/>
```

### Rules

- **`consequences` over prose** — bullets are scannable; prose buries the action.
- **Forward-motion approvals → `severity="success"`** — they're not warnings.
- **Archive / regenerate → `severity="destructive"`** — uses the destructive button variant.
- **Caller owns the `loading` flag** — dialog stays open while the async action runs so the user sees "Approving…" instead of a flash close.
- **Each approval explains the _full_ state transition** — what becomes a deliverable, which phase opens next, what stays editable.

## 13. AIProgressPanel — show progress on long calls

`comp-ai-progress.tsx` — replaces "click button, page sits frozen for 60s" with a panel showing what's happening.

### When to use

Any AI call ≥10s.

### Structure

```tsx
{generating && (
  <AIProgressPanel
    stage="Assembling PRD"
    expectedSeconds={{ min: 20, max: 60 }}
    model="claude-sonnet-4-6"
  />
)}
```

### Rules

- **Stage label is verb-led** — "Reviewing the idea", "Assembling PRD", not "Loading" or "Please wait".
- **Show elapsed time + expected range** — honest beats fake. After expected max, message switches to "Still working — larger inputs can take a bit longer."
- **Surface the model name** — helps the user understand cost/speed.
- **Hide the trigger UI while running** — show only the progress panel, not a disabled button next to it.
- **On failure** — show an actionable retry card, not a permanent silence.

### Auto-trigger on mount when sensible

If a page is _known_ to need an AI call to be useful (e.g., a phase page on a fresh project with no draft), auto-trigger generation on mount. State guards against loops:

```tsx
const [autoGenerateAttempted, setAutoGenerateAttempted] = useState(false);
const [autoGenerateFailed, setAutoGenerateFailed] = useState(false);

useEffect(() => {
  if (autoGenerateAttempted) return;
  if (!phase) return;
  if (draft) return;
  if (generating) return;
  if (autoGenerateFailed) return;
  setAutoGenerateAttempted(true);
  void handleGenerate();
}, [autoGenerateAttempted, phase, draft, generating, autoGenerateFailed, handleGenerate]);
```

Defensive against: re-firing each render, racing the initial fetch, looping on a failed generate.

## 14. Phase data refetch on focus

For pages where another tab / route / phase might mutate data underneath, refetch on `window` focus + `document` visibility change:

```tsx
useEffect(() => {
  const onFocus = () => void fetchProject();
  window.addEventListener("focus", onFocus);
  document.addEventListener("visibilitychange", onFocus);
  return () => {
    window.removeEventListener("focus", onFocus);
    document.removeEventListener("visibilitychange", onFocus);
  };
}, [fetchProject]);
```

Lets the user navigate away, do work, and come back to see latest status without manual reload.

---

## Related Standards

- [`guide-code.md`](guide-code.md) — naming, component standards
- [`guide-tables.md`](guide-tables.md) — full table spec
- [`guide-forms.md`](guide-forms.md) — validation + form patterns
- [`guide-arch.md`](guide-arch.md) — data flow, hooks pattern
