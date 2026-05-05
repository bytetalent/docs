---
name: Proxy Egress Pattern
description: All LLM and tool provider calls route through POST /api/proxy/... for credential resolution, audit logging, and prompt assembly. Direct provider fetch is never allowed outside adapter.test().
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [nextjs-clerk-supabase]
version: 1
---

The proxy layer is the sole gateway for outbound calls to LLM and external tool providers. Every feature that calls a model routes through it. Per `bytetalent/docs/guide-arch.md` Proxy Layer section.

## Why the proxy exists

1. **3-tier credential resolution** — client key → consultant key → platform key, one implementation
2. **Audit logging** — `proxy_requests` row per call: tokens in/out, latency, `keySource`, `connectionId`, phase prompt + skill versions. Future charge-back depends on this
3. **Prompt assembly** — phase prompt + skill chunks + project context + user input; versioned and auditable
4. **Caching** — Anthropic prompt caching (40-60% cost reduction) only works with consistent system prompts; centralized assembly makes it automatic
5. **Rate limits + retries** — one place per provider
6. **Provider swap** — change a `phase_prompts` row, not application code

## Routes

```
POST /api/proxy/model       → model provider (Anthropic + future OpenAI/Google/xAI); streams SSE
src/lib/proxy/invoke-buffered.ts → same pipeline, buffers SSE; for phases that store output
```

The proxy is HTTP, not a function call. Server-side Route Handlers call it via `fetch('/api/proxy/...')` — same path as browser. One internal HTTP hop; worth it for consistency.

## Sole exception: adapter.test()

`adapter.test()` during BYOK credential validation is exempt from the proxy. It runs pre-proxy because there is no project context, no phase, no skills — the call's only purpose is to validate the credential being added. Lives entirely in `src/lib/connections/providers/<slug>.ts`.

## 3-tier credential resolver

```
For (projectId, phase, provider):
  1. Client-owned connection: project_connections → connections.clientId IS NOT NULL
     → client's Vault secret; burns client's tokens
  2. Consultant-owned connection: connections.accountId, clientId IS NULL
     → consultant's Vault secret; burns consultant's tokens
  3. Platform fallback: process.env.ANTHROPIC_API_KEY (or per-provider equivalent)
     → Bytetalent absorbs cost; future credit_ledger metering

Returns { keySource: "client" | "account" | "platform", secret, connectionId }
```

## Buffered invocation (server-side phases)

Pipeline phases that store model output use `invokeProxyBuffered` — same prompt assembly pipeline, buffers SSE in-process:

```ts
import { invokeProxyBuffered } from "@/lib/proxy/invoke-buffered";

const { text, usage, resolution } = await invokeProxyBuffered({
  accountId,
  projectId,
  phase: "prd_assembly",
  userInput: prdInput,
});
// text is the full model response; usage has token counts; resolution has keySource
```

## Tolerant JSON parsing

Phases that demand strict-JSON output use `parseStrictJsonTolerant` (`src/lib/proxy/parse-strict-json.ts`):

1. Strip markdown code fences
2. Try strict `JSON.parse`
3. Fall back to `JSON5.parse` (accepts trailing commas, single quotes, near-JSON-isms)

Use this for all phases where the model is instructed to return JSON (idea review, PRD generate/review, brand kit generate/review).

## SSE streaming shape

```
data: {"type":"start","provider":"anthropic","model":"..."}
data: {"type":"token","text":"..."}       ← per token
data: {"type":"usage","inputTokens":...,"outputTokens":...}
data: {"type":"done","status":"success"}
```

Use `Response` with `TransformStream` for streaming — do not pull in Vercel AI SDK.

## Good

```ts
// Server-side route calling the proxy for model output
const res = await fetch("/api/proxy/model", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ accountId, projectId, phase: "idea_review", userInput }),
});
```

## Bad

```ts
// Never call Anthropic directly from a route handler
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
const message = await client.messages.create({ ... }); // bypasses proxy entirely
```
