---
name: Next.js App Router Conventions
description: Folder structure, server/client split, route handlers — Next.js 16 App Router.
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Next.js App Router conventions for this codebase:
- src/app/ for routes. Each dashboard page is page.tsx (server, thin) +
  page-client.tsx ("use client", all state).
- Each route directory carries error.tsx, loading.tsx, not-found.tsx.
- API endpoints under src/app/api/[domain]/route.ts. Use Route Handlers,
  not Server Actions, for any mutation.
- src/hooks/use-[entity]-list.ts owns domain state — fetches from /api/[entity]
  via useEffect; returns { items, stats, filters, ... }.
- Components grouped by domain under src/components/[domain]/, prefix comp-.
- src/lib/ for utilities; src/lib/db/ for Drizzle; src/lib/services/ for
  cross-feature services.
- Server-only modules import 'server-only' at the top.
