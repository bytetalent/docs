---
name: Infrastructure Service Defaults
description: When adding caching, queues, storage, realtime, or analytics — use the already-in-stack service. Avoid a fourth service when Vercel+Supabase+Upstash covers it.
category: infra
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Where two services overlap, favor the one already in the stack with the deepest native support. Adding a fourth service requires explicit justification. Per `bytetalent/docs/guide-arch.md` infrastructure section.

## Service decision table

| Concern                      | Use                                           | Do not add                           |
| ---------------------------- | --------------------------------------------- | ------------------------------------ |
| Caching                      | Upstash Redis                                 | Vercel KV (same infra, less control) |
| Rate limiting                | Upstash Ratelimit                             | Custom middleware                    |
| Job queue / fan-out / delays | Upstash QStash                                | BullMQ, Redis queues                 |
| Scheduled lightweight tasks  | Vercel Cron                                   | QStash for simple recurring tasks    |
| File / asset storage         | Supabase Storage                              | Vercel Blob                          |
| Real-time updates            | Supabase Realtime (Postgres change listeners) | Pusher, Ably, third-party WebSocket  |
| Observability                | Grafana Cloud (Loki + Prometheus + Tempo)     | Axiom (logs only)                    |
| Product analytics            | PostHog                                       | Vercel Analytics, Segment            |
| Error tracking               | Sentry                                        | Custom logging solution              |
| Transactional email          | Resend                                        | SendGrid, Postmark                   |
| Payments                     | Stripe                                        | —                                    |

## Rules

- **Vercel KV vs Upstash Redis** — Vercel KV is Upstash under the hood with an abstraction layer. Use Upstash Redis directly.
- **Vercel Blob vs Supabase Storage** — Supabase Storage is already in the stack alongside the DB. Use it for user uploads (avatars, design assets).
- **QStash vs Vercel Cron** — use both but for different purposes: QStash for the execution engine (retries, delays, callbacks, fan-out); Vercel Cron for lightweight periodic server tasks (monthly invoice generation, cleanup).
- **Supabase Realtime vs WebSocket services** — Supabase Realtime supports Postgres change listeners. Use it for live run status updates; no third-party WebSocket service needed.
- For Next.js projects, app is hosted on Vercel; DB on Supabase PostgreSQL; auth on Clerk.

## Realtime subscription pattern

```ts
// Client-side: subscribe to run status changes
const channel = supabase
  .channel(`run:${runId}`)
  .on(
    "postgres_changes",
    {
      event: "UPDATE",
      schema: "public",
      table: "runs",
      filter: `id=eq.${runId}`,
    },
    (payload) => setRunStatus(payload.new.status)
  )
  .subscribe();

return () => {
  supabase.removeChannel(channel);
};
```

## Bad

```ts
// Never add Pusher when Supabase Realtime covers the use case
import Pusher from "pusher"; // unnecessary third-party WebSocket

// Never reach for Vercel KV when Upstash Redis is in the stack
import { kv } from "@vercel/kv"; // prefer @upstash/redis directly
```
