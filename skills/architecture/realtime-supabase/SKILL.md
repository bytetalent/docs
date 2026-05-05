---
name: Realtime Updates — Supabase
description: Use Supabase Realtime Postgres change listeners for live status updates; no polling, no third-party WebSocket service (Pusher/Ably).
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

Pattern for live UI updates in Bytetalent projects. Source: `bytetalent/docs/guide-arch.md` §6.

## The decision

Use **Supabase Realtime** (Postgres change listeners) for any feature that needs live status updates — run/job execution progress, notifications, collaborative presence. No polling. No Pusher, Ably, or other third-party WebSocket service.

Rationale: Supabase is already in the stack. Adding Pusher or Ably adds a fourth service dependency, a second WebSocket handshake, and an extra billing relationship for a concern already covered.

## Pattern

```
Run starts → run.status = 'running' written to DB
           → Supabase Realtime broadcasts INSERT/UPDATE on runs table
           → client subscription receives change event
           → UI updates without refetch
```

Client-side subscription (Next.js / React):

```ts
import { createClient } from "@supabase/supabase-js";

// Use the client-side Supabase client (not admin/service-role)
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

useEffect(() => {
  const channel = supabase
    .channel("run-status")
    .on(
      "postgres_changes",
      { event: "UPDATE", schema: "public", table: "runs", filter: `id=eq.${runId}` },
      (payload) => {
        setRun(payload.new as Run);
      }
    )
    .subscribe();

  return () => {
    supabase.removeChannel(channel);
  };
}, [runId]);
```

## Rules

- **No polling** — don't `setInterval` a status-check fetch when Realtime covers the use case.
- **Row-level filter** — always filter by `id=eq.<id>` (or equivalent) to avoid broadcasting all-table changes to all connected clients.
- **RLS applies** — Realtime respects Postgres RLS. Clients receive only rows their RLS policy allows.
- **Cleanup subscriptions** — call `supabase.removeChannel(channel)` on unmount. Leaked channels cause connection buildup.
- **Don't add Pusher/Ably** — if Realtime's current feature set doesn't cover the use case, raise it as an architectural change before adding a new service.

## When Realtime is not the right tool

- High-frequency writes (> ~50/sec per table) — Realtime adds latency overhead; consider batching or a dedicated queue.
- Fan-out to millions of subscribers — Supabase Realtime is not a broadcast CDN.
- Bi-directional signaling (WebRTC, multiplayer cursor) — Supabase Broadcast channels are a better fit than Postgres change listeners.

## Anti-patterns

```ts
// Bad — polling for status
useEffect(() => {
  const interval = setInterval(async () => {
    const run = await fetchRun(runId);
    setRun(run);
  }, 2000);
  return () => clearInterval(interval);
}, [runId]);

// Bad — subscribing to entire table without a row filter
.on("postgres_changes", { event: "UPDATE", schema: "public", table: "runs" }, handler)
// broadcasts every run update to every subscriber
```

## See also

- [`skills/architecture/standardized-access-paths`](../standardized-access-paths/SKILL.md) — Supabase Realtime listed as the canonical real-time path; don't add parallel WebSocket services
- [`skills/infra/infra-service-defaults`](../../infra/infra-service-defaults/SKILL.md) — "Supabase Realtime vs. Pusher/Ably → Realtime" in the overlap-decisions table
- `bytetalent/docs/guide-arch.md` §6 — source section
