---
name: API SSE Streaming
description: Use Response + TransformStream for SSE; do not pull in Vercel AI SDK; normalize event shape across providers.
category: api
applicable_phases: [code_gen]
applicable_stacks: [nextjs-clerk-supabase, expo-clerk-supabase]
version: 1
---

Server-Sent Events (SSE) streaming from Next.js Route Handlers. Source: `bytetalent/docs/guide-api.md`.

## Rules

- Use the Web Streams API (`Response` + `TransformStream`) — do not add the Vercel AI SDK as a dependency
- Normalize the event shape across all providers (Anthropic, OpenAI, etc.) before sending to the client
- Set the correct SSE headers on the `Response`
- Stream chunks immediately; do not buffer the full response

## Standard SSE event shape

```ts
// Normalized event shape — same for all providers
type SseEvent =
  | { type: "chunk"; text: string }
  | { type: "done"; usage?: { inputTokens: number; outputTokens: number } }
  | { type: "error"; message: string };

function ssePayload(event: SseEvent): string {
  return `data: ${JSON.stringify(event)}\n\n`;
}
```

## Route Handler pattern

```ts
// src/app/api/generate/route.ts
export async function POST(req: Request) {
  const { prompt } = await req.json();
  const { accountId } = await resolveAccount();

  const { readable, writable } = new TransformStream();
  const writer = writable.getWriter();
  const encoder = new TextEncoder();

  // Start streaming in the background
  (async () => {
    try {
      const stream = await callProviderStream(prompt, accountId);
      for await (const chunk of stream) {
        await writer.write(encoder.encode(ssePayload({ type: "chunk", text: chunk.text })));
      }
      await writer.write(encoder.encode(ssePayload({ type: "done" })));
    } catch (err) {
      console.error("[POST /api/generate] stream error:", err);
      await writer.write(encoder.encode(ssePayload({ type: "error", message: "Generation failed" })));
    } finally {
      await writer.close();
    }
  })();

  return new Response(readable, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive",
    },
  });
}
```

## Client-side consumption

```ts
// hooks/use-generate-stream.ts
async function startStream(prompt: string, onChunk: (text: string) => void) {
  const res = await fetch("/api/generate", {
    method: "POST",
    body: JSON.stringify({ prompt }),
    headers: { "Content-Type": "application/json" },
  });

  const reader = res.body!.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const lines = decoder.decode(value).split("\n\n").filter(Boolean);
    for (const line of lines) {
      if (!line.startsWith("data: ")) continue;
      const event: SseEvent = JSON.parse(line.slice(6));
      if (event.type === "chunk") onChunk(event.text);
      if (event.type === "done") return;
      if (event.type === "error") throw new Error(event.message);
    }
  }
}
```

## Bad

```ts
// Never: Vercel AI SDK (adds unnecessary dependency)
import { streamText } from "ai"; // don't use this in route handlers

// Never: buffer entire response
const fullResponse = await callProvider(prompt);
return Response.json({ text: fullResponse }); // not streaming

// Never: provider-specific event shape leaked to client
return encoder.encode(`data: ${JSON.stringify(anthropicChunk)}\n\n`); // exposes provider shape
```
