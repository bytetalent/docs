---
name: Clerk Auth Patterns
description: Server auth(), middleware, JWT template for Supabase RLS, social login.
category: security
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Clerk auth patterns:
- Server-side: import { auth } from "@clerk/nextjs/server"; const { userId } = await auth();
  Reject unauthenticated requests as the FIRST line of every Route Handler.
- requireAccount() / requireSuperAdmin() helpers in src/lib/session.ts wrap auth().
- Supabase access uses createServerClient() which injects the Clerk JWT
  via getToken({ template: "supabase" }) — RLS resolves auth.uid() to the
  Clerk user ID.
- Middleware (src/middleware.ts) protects routes via clerkMiddleware().
- Webhooks (src/app/api/webhooks/clerk/route.ts) use Svix HMAC validation.
- Service role client only for billing webhooks, migrations, cron — never
  imported from client code.
