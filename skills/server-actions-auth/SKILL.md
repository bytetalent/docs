---
name: Server Actions and Auth Guard Order
description: Zod validation before data touch, auth guard sequence, authz inside the action, security headers.
category: code
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Conventions for server actions, mutations, and auth guard sequencing.
Source: guide-code.md §6, guide-arch.md.

## Auth guard order (source: guide-arch.md)

Every route handler and server action must follow this exact sequence:

1. **Schema validate** — parse input with zod; return 400 on failure.
2. **Auth check** — verify the caller is authenticated; return 401 if not.
3. **Authz check** — verify the caller is authorized to act on the resource; return 403 if not.
4. **Business logic** — only after all three guards pass.

```typescript
// Good — correct guard sequence
export async function PUT(req: Request, { params }: { params: { id: string } }) {
  // 1. Schema validate
  const body = await req.json();
  const parsed = updateFlowSchema.safeParse(body);
  if (!parsed.success) return NextResponse.json({ error: "validation" }, { status: 400 });

  // 2. Auth check
  const account = await resolveAccount();
  if (!account) return NextResponse.json({ error: "unauthorized" }, { status: 401 });

  // 3. Authz check
  const existing = await flowsRepository.getById(params.id, account.id);
  if (!existing) return NextResponse.json({ error: "not_found" }, { status: 404 });

  // 4. Business logic
  await flowsRepository.update(params.id, account.id, parsed.data);
  return NextResponse.json({ ok: true });
}
```

## Server actions / mutations (source: guide-code.md §6)

- Validate all input with zod **before** touching data.
- Perform authz checks inside the action/handler — **not** only in UI.
- Return typed success/error shapes; avoid leaking raw DB errors.
- Never trust client-side auth state for authorization decisions.

```typescript
// Good
"use server";
export async function updateFlow(id: string, input: unknown) {
  const parsed = updateFlowSchema.safeParse(input);
  if (!parsed.success) return { error: "Invalid input" };

  const account = await resolveAccount();
  if (!account) return { error: "Unauthorized" };

  await flowsRepository.update(id, account.id, parsed.data);
  return { ok: true };
}

// Bad — no zod, no authz check, trusting the caller
("use server");
export async function updateFlow(id: string, data: FlowData) {
  await db.update(flows).set(data).where(eq(flows.id, id));
}
```

## RLS + app-layer defense in depth (source: guide-code.md §6, guide-db.md)

- RLS is the **primary** security boundary.
- App-layer `accountId` filtering in Drizzle queries is **defense in depth**
  (not a replacement for RLS).
- Never disable RLS as a workaround; fix the policy instead.

## Security headers (source: guide-code.md §10)

Security headers are configured in `next.config.mjs` via the `headers()` function:

- `Content-Security-Policy` — restrict script/style/connect sources.
- `Strict-Transport-Security` — HSTS with preload.
- `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`.

When adding a new external service (analytics, CDN, etc.), update `connect-src`
and any other relevant CSP directives.

## Performance defaults (source: guide-code.md §13)

- Prefer Server Components and streaming; minimize client component boundaries.
- Use `next/image` for all images — never raw `<img>` with external URLs.
- Cache reads intentionally; revalidate explicitly after mutations.

```tsx
// Good
import Image from "next/image";
<Image src={logo} alt="Logo" width={120} height={40} />

// Bad
<img src={logoUrl} alt="Logo" />
```
