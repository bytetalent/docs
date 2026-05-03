---
name: API Webhook Security
description: Validate provider HMAC signature first on every inbound webhook; require idempotency via delivery ID; respond within 5 seconds.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Security and reliability rules for inbound webhook handlers. Source: `bytetalent/docs/guide-api.md`.

## Three rules

Every webhook handler must:

1. **Validate HMAC signature first** — before parsing the body or touching the DB
2. **Enforce idempotency** — track delivery IDs; short-circuit on repeats
3. **Respond within 5 seconds** — acknowledge immediately; offload slow work asynchronously

## HMAC validation

```ts
// Good — signature validated before any other processing
export async function POST(req: Request) {
  const rawBody = await req.text(); // must read as text for HMAC
  const signature = req.headers.get("x-webhook-signature") ?? "";

  const isValid = await verifyHmac(rawBody, signature, process.env.WEBHOOK_SECRET!);
  if (!isValid) {
    console.warn("[webhook] invalid HMAC signature");
    return new Response("Unauthorized", { status: 401 });
  }

  const payload = JSON.parse(rawBody);
  // proceed ...
}

// Bad — body parsed before signature check
export async function POST(req: Request) {
  const payload = await req.json(); // skip signature check, process blind
}
```

```ts
// HMAC helper (timing-safe comparison)
async function verifyHmac(body: string, signature: string, secret: string): Promise<boolean> {
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey("raw", encoder.encode(secret), { name: "HMAC", hash: "SHA-256" }, false, [
    "sign",
  ]);
  const mac = await crypto.subtle.sign("HMAC", key, encoder.encode(body));
  const expected = Buffer.from(mac).toString("hex");
  // timing-safe comparison
  return expected.length === signature.length && crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}
```

## Idempotency

Track the provider's delivery ID in a `webhook_deliveries` table (or equivalent). Short-circuit with `200` on a repeat — do not re-process.

```ts
// Good — idempotency check after signature validation
const deliveryId = req.headers.get("x-delivery-id") ?? payload.id;
const existing = await db.query.webhookDeliveries.findFirst({
  where: eq(webhookDeliveries.deliveryId, deliveryId),
});
if (existing) {
  return Response.json({ status: "already_processed" }); // 200, not 4xx
}

// Process the event
await processWebhookEvent(payload);

// Record delivery
await db.insert(webhookDeliveries).values({ deliveryId, processedAt: new Date() });
return Response.json({ status: "ok" });
```

## Respond within 5 seconds

Webhook providers retry on slow responses. Acknowledge immediately and enqueue slow work.

```ts
// Good — immediate ack, async processing via QStash/queue
await enqueueJob("process-webhook", { deliveryId, payload });
return Response.json({ status: "queued" }); // fast response

// Bad — long synchronous processing in handler
await runSlowMigration(payload); // provider may time out and retry
return Response.json({ status: "ok" });
```

## Signature algorithms by provider

| Provider | Header                | Algorithm     |
| -------- | --------------------- | ------------- |
| GitHub   | `x-hub-signature-256` | HMAC-SHA-256  |
| Stripe   | `stripe-signature`    | HMAC-SHA-256  |
| Resend   | `svix-signature`      | Svix (SHA256) |
| Linear   | `linear-delivery`     | HMAC-SHA-256  |

Always check the provider's official docs — never skip or weaken the signature check even in development.
