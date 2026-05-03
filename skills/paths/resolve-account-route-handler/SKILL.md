---
name: Resolve Account Route Handler
description: Every protected Next.js Route Handler opens with resolveAccount() + 401 guard — the single auth/account resolution path.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Every protected Route Handler resolves the authenticated account through one function: `resolveAccount()`. This is an architectural invariant, not just a convention — using parallel auth paths caused a real production incident (PGRST002 Vault incident, 2026-04-28).

Per `app-flow-ai-web/docs/patterns-inventory.md` entry #12.

Related skill: `skills/clerk-auth` — Clerk middleware + `resolveAccount()` implementation.

## Core rules

- Every protected route handler calls `resolveAccount()` as the **first line** before any business logic.
- If `resolveAccount()` returns `null` (unauthenticated or no DB account), return `401` immediately — no further processing.
- Use `account.accountId` for all DB queries that scope to the authenticated user.
- Admin-only routes use `requireSuperAdmin()` instead of `resolveAccount()`.
- Never call Clerk SDK directly (`auth()`, `currentUser()`) from within a route handler — that's `resolveAccount()`'s job.
- Never introduce a second auth resolution path. If you need a different resolution behavior, extend `resolveAccount()` in `src/lib/session.ts`.

## Standard opening block (every protected route)

```ts
// src/app/api/entities/route.ts
import { resolveAccount } from "@/lib/session";
import { apiError } from "@/lib/api-error";
import { NextResponse } from "next/server";

export async function GET(_req: Request) {
  const account = await resolveAccount();
  if (!account)
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  try {
    const items = await db.query.entities.findMany({
      where: (t, { and, eq, isNull }) =>
        and(eq(t.accountId, account.accountId), isNull(t.deletedAt)),
    });
    return NextResponse.json(items);
  } catch (err) {
    return apiError("GET /api/entities", err);
  }
}

export async function POST(req: Request) {
  const account = await resolveAccount();
  if (!account)
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  // ...
}
```

## Account type gating

Some resources are consultancy-only (clients, projects, brand kits). Check `accountType` after resolving:

```ts
export async function POST(req: Request) {
  const account = await resolveAccount();
  if (!account)
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  if (account.accountType !== "consultancy") {
    return NextResponse.json({ error: "Forbidden" }, { status: 403 });
  }

  // proceed with consultancy-only logic
}
```

## Admin-only routes

```ts
// src/app/api/admin/consultancies/route.ts
import { requireSuperAdmin } from "@/lib/session";

export async function GET(_req: Request) {
  const admin = await requireSuperAdmin();
  if (!admin) return NextResponse.json({ error: "Forbidden" }, { status: 403 });

  // admin-only logic
}
```

## What NOT to do

```ts
// Bad — calling Clerk SDK directly, bypassing the canonical path
import { auth } from "@clerk/nextjs/server";

export async function GET() {
  const { userId } = await auth(); // Wrong — use resolveAccount()
  if (!userId)
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  // This bypasses the DB account lookup and misses accountId, accountType, etc.
}
```

```ts
// Bad — skipping the null check (treats missing auth as authenticated)
export async function GET() {
  const account = await resolveAccount();
  // Forgot the null check — if account is null, account.accountId throws
  const items = await db.query.entities.findMany({
    where: (t, { eq }) => eq(t.accountId, account.accountId), // null dereference
  });
}
```

```ts
// Bad — adding a second resolution path for a special case
async function resolveAccountFromHeader(req: Request) {
  const token = req.headers.get("X-Account-Token");
  // ... new auth path — prohibited without architecture review
}
```

## The account object shape

`resolveAccount()` returns `null` or an object with at minimum:

```ts
interface ResolvedAccount {
  accountId: string;
  accountType: "consultancy" | "user";
  clerkUserId: string;
}
```

Do not assume additional fields exist without checking `src/lib/session.ts`.

## Related skills

- `skills/clerk-auth` — Clerk middleware, `resolveAccount()` implementation
- `skills/paths/api-error-handler` — the try/catch frame that follows auth
- `skills/standardized-access-paths` — why one canonical path per concern is enforced
