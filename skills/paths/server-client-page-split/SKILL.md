---
name: Server Client Page Split
description: Next.js App Router RSC + "use client" page-pair pattern — thin server wrapper delegates to the interactive client component. Includes Suspense requirement for useSearchParams.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 2
composes_with: [wizard, entity-list-hook]
nests: []
conflicts_with: []
---

Every interactive page in Next.js App Router needs a thin RSC shell that satisfies the server boundary requirement. This pattern standardizes that split.

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #1.

## Core rules

- Every interactive page is a **pair**: `page.tsx` (RSC) + `page-client.tsx` (`"use client"` component).
- `page.tsx` does one of two things only: pure delegation (no props) or server-resolved hydration (slug→id, auth check, initial DB fetch passed as props).
- `page-client.tsx` owns all state, effects, and event handlers. It never imports server-only modules.
- The RSC layer exists solely to satisfy the App Router server/client boundary — it has no UI logic.
- File naming is always `page.tsx` / `page-client.tsx`. No `page-wrapper.tsx`, no `page-container.tsx`.

## Two shapes

### Pure delegation (no server work)

```tsx
// page.tsx — RSC, pure delegation
import { ClientPage } from "./page-client";

export default function Page() {
  return <ClientPage />;
}
```

```tsx
// page-client.tsx
"use client";

export function ClientPage() {
  // all state, effects, handlers here
}
```

### Server-resolved hydration (props passed down)

```tsx
// page.tsx — RSC, resolves slugs + fetches minimal seed data
import { db } from "@/lib/db";
import { ClientPage } from "./page-client";
import { notFound } from "next/navigation";

interface Props {
  params: { clientSlug: string; projectSlug: string };
}

export default async function Page({ params }: Props) {
  const project = await db.query.projects.findFirst({
    where: (t, { eq }) => eq(t.slug, params.projectSlug),
  });
  if (!project) notFound();

  return <ClientPage projectId={project.id} slug={params.projectSlug} />;
}
```

```tsx
// page-client.tsx
"use client";

interface Props {
  projectId: string;
  slug: string;
}

export function ClientPage({ projectId, slug }: Props) {
  // client-side data fetching, state, handlers
}
```

## What NOT to do

```tsx
// Bad — page.tsx with UI logic + client imports mixed in
"use client"; // Wrong — page.tsx must be a server component
import { useState } from "react";

export default function Page() {
  const [open, setOpen] = useState(false); // Move this to page-client.tsx
  return <div onClick={() => setOpen(true)}>...</div>;
}
```

```tsx
// Bad — page-client.tsx doing slug→id resolution (belongs in page.tsx)
"use client";
import { useParams } from "next/navigation";
import { db } from "@/lib/db"; // Can't import server-only in "use client"

export function ClientPage() {
  const { slug } = useParams();
  // ...
}
```

## When to use server-resolved hydration vs. pure delegation

- Use **server-resolved hydration** when: you need to resolve a URL slug to a DB row ID, or you want to pass initial seed data to avoid a loading flash.
- Use **pure delegation** when: the page handles its own data fetching via SWR/fetch hooks, or there is no route parameter to resolve.

## Suspense requirement for useSearchParams

**When a client component reads `useSearchParams()`, `usePathname()`, or `useRouter()` from `next/navigation`, Next.js 14+ requires a `<Suspense>` boundary wrapping the consumer.** Without it, the build fails (or static export fails) with an error like:

```
Error: useSearchParams() should be wrapped in a suspense boundary at page "/space/flows"
```

This commonly affects wizard pages (URL-encoded step/field state), entity list pages (URL-driven filters), and any page using `useRouter` for navigation.

### Pure delegation — no useSearchParams (no Suspense needed)

```tsx
// page.tsx — safe, no Suspense needed
import { ClientPage } from "./page-client";

export default function Page() {
  return <ClientPage />;
}
```

```tsx
// page-client.tsx
"use client";

export function ClientPage() {
  // Does NOT call useSearchParams / usePathname / useRouter
}
```

### Client component reads useSearchParams — Suspense required

```tsx
// page.tsx — wrap with Suspense
import { Suspense } from "react";
import { ClientPage } from "./page-client";
import { LoadingSpinner } from "@/components/shared/comp-loading-spinner";

export default function Page() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <ClientPage />
    </Suspense>
  );
}
```

```tsx
// page-client.tsx
"use client";
import { useSearchParams } from "next/navigation";

export function ClientPage() {
  const searchParams = useSearchParams(); // safe — wrapped in Suspense above
  const status = searchParams.get("status") ?? "all";
  // ...
}
```

### Server-resolved hydration — no Suspense needed in page.tsx

When `page.tsx` resolves the slug and passes props down, the client component typically doesn't need `useSearchParams` for initialization (seed data comes from props). Suspense is still needed if the client component calls `useSearchParams` for secondary state (e.g., tab selection):

```tsx
// page.tsx — resolves slug, no Suspense for the seed data
export default async function Page({ params }: Props) {
  const project = await db.query.projects.findFirst({ ... });
  if (!project) notFound();

  return (
    // Still wrap if page-client.tsx uses useSearchParams internally
    <Suspense fallback={<LoadingSpinner />}>
      <ClientPage projectId={project.id} />
    </Suspense>
  );
}
```

### Rule of thumb

> If `page-client.tsx` (or any component it renders) imports anything from `next/navigation`, wrap `<ClientPage />` in `<Suspense>` in `page.tsx`. When in doubt, add it — a redundant Suspense boundary is harmless; a missing one breaks the build.

Both `wizard` and `entity-list-hook` use `useSearchParams` — their page.tsx wrappers always need `<Suspense>`.

## Related skills

- `skills/nextjs-app-router` — App Router conventions, layouts, loading states
- `skills/data-access-patterns` — how the client component should fetch data
- `skills/paths/wizard` — uses useSearchParams for URL-encoded step + field state
- `skills/paths/entity-list-hook` — domain hook reads useSearchParams for initial filters
