# ADR-003: SSE Delivery — Route Handler Bypass over Next.js Rewrites

**Status:** Accepted  
**Date:** February 2026  
**Component:** `nexus-frontend` (TypeScript / Next.js 14)

---

## Context

The frontend needs to stream Server-Sent Events from the Python RAG backend (`localhost:8000`) to the browser. The standard Next.js approach for proxying backend traffic is the `rewrites()` configuration in `next.config.ts`, which maps a frontend path to a backend URL:

```typescript
// next.config.ts
async rewrites() {
  return [{ source: '/api/python/:path*', destination: 'http://localhost:8000/:path*' }]
}
```

This works correctly for standard request-response endpoints. The question is whether it works for long-lived SSE streams.

---

## Decision

Use a **Next.js Route Handler** at `app/api/python/query/stream/route.ts` that calls `fetch()` directly to the backend and pipes the upstream ReadableStream to the browser. The Next.js rewrite is **not used for the streaming endpoint**.

Non-streaming endpoints (embedding, NER, note CRUD) continue to use the rewrite. Only the SSE streaming path uses the Route Handler.

---

## What Failed and Why

### Initial approach: Next.js rewrite

The `rewrites()` configuration was set up first because it requires no additional code. All `/api/python/*` traffic routes to the backend automatically.

**Failure:** Streaming queries failed with `ECONNRESET` every time. The stream would open (the browser received the initial SSE headers), then die immediately before any event arrived.

**Diagnosis process:**

1. Tested the Python backend directly: `curl -N http://localhost:8000/api/v1/query/stream` — stream worked correctly, events arrived in real time.
2. Tested through the Next.js rewrite: stream opened then reset. Confirmed the rewrite was the failure point.
3. Checked Next.js documentation on streaming — rewrites are described as working for "all requests" but SSE is not mentioned specifically.
4. Added logging to the Route Handler to inspect headers mid-stream — found the upstream connection was being held open by Next.js but events were not being forwarded until the stream closed.

**Root cause:** Next.js `rewrites()` uses an internal HTTP proxy (`http-proxy` or equivalent) that **buffers the complete response body before forwarding it to the client**. For a standard request-response, this is invisible — the response arrives in full before the client sees it. For SSE, the proxy waits for the stream to close so it can forward the complete body. The stream never closes during an active query, so the proxy eventually times out and resets the connection.

This behaviour is consistent with how HTTP/1.1 proxies work by default. Next.js does not document that rewrites buffer responses, nor does it provide a configuration option to disable buffering for specific routes.

---

## Alternatives Considered

### Option A: Next.js rewrite with streaming headers

Attempted to hint to the proxy that this was a streaming response by setting response headers on the backend:

```python
# rag-main/api/routes.py
headers = {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    "X-Accel-Buffering": "no",       # nginx directive
    "Transfer-Encoding": "chunked",
}
```

**Result:** No effect on the Next.js rewrite layer. The `X-Accel-Buffering: no` header is a nginx directive — nginx respects it, but the Next.js internal proxy does not. The ECONNRESET persisted.

### Option B: Direct backend URL from the browser

Skip the proxy entirely. Call `http://localhost:8000/api/v1/query/stream` directly from the browser.

**Why rejected:** This exposes the backend port directly to the browser. In development this works, but in any deployed context the backend is not accessible from the client — only the Next.js server has access to it. The architecture assumes the frontend server is the only entry point. Bypassing the frontend server breaks this assumption and would require CORS configuration on the backend for every deployment environment.

### Option C: WebSockets

Replace SSE with a WebSocket connection, which is explicitly stream-oriented and well-supported by Next.js.

**Why rejected:** The streaming protocol is unidirectional — the server sends events, the client only listens (plus an abort signal). SSE is the correct primitive for this. WebSockets add bidirectional complexity for a unidirectional use case. The backend already implements SSE. Replacing it would require changes to the Python service and the frontend event consumer with no functional gain.

### Option D: Route Handler with direct fetch (chosen)

A Next.js Route Handler runs server-side and has direct network access to the backend. It can call `fetch()`, receive the upstream ReadableStream, and return a `Response` object that pipes the stream to the client.

```typescript
// app/api/python/query/stream/route.ts
export async function POST(request: Request) {
  const body = await request.json();

  const upstream = await fetch('http://localhost:8000/api/v1/query/stream', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
    // @ts-ignore — required for Node 18 streaming
    duplex: 'half',
  });

  return new Response(upstream.body, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'X-Accel-Buffering': 'no',
    },
  });
}
```

`new Response(upstream.body)` passes the ReadableStream directly to the Next.js response writer. No buffering occurs. Events flow from the Python backend through the Route Handler to the browser in real time.

**Why this works where the rewrite does not:** The Route Handler is application code, not a proxy layer. When it returns `new Response(stream)`, Next.js writes the stream incrementally to the HTTP response as bytes arrive. The rewrite layer's buffering behaviour does not apply — the Route Handler owns the response lifecycle.

---

## Consequences

**Positive:**
- SSE events stream correctly from backend to browser with no buffering
- The frontend server remains the single entry point — no direct backend exposure
- `X-Accel-Buffering: no` in the Route Handler response prevents nginx from re-introducing buffering at the reverse-proxy layer
- The fix is isolated to the streaming path — non-streaming endpoints still use the simpler rewrite

**Negative:**
- Two code paths exist for backend communication: rewrites for non-streaming, Route Handler for streaming. This is additional surface area that must be kept consistent if the backend URL changes
- `duplex: 'half'` is required for Node 18 and is undocumented in the official Next.js docs — it is a Node.js Fetch API flag required when the request body is also a stream. It must be type-cast to suppress TypeScript errors
- The Route Handler adds a small amount of overhead versus a true transparent proxy — acceptable for this use case

---

## AbortController Integration

The Route Handler approach enables correct client-side cancellation. The browser sends an `AbortSignal` through `retrievalService.ts`, which is forwarded to the `fetch()` call in the Route Handler:

```typescript
// Three abort triggers wired in retrievalService.ts
// 1. Query text changes while streaming
// 2. Search palette closes
// 3. User clicks Stop button
```

When the signal fires, the `fetch()` in the Route Handler aborts, the upstream connection to the Python backend closes, and the Python SSE generator stops executing. The abort propagates correctly through the full stack.

---

## What Would Change This Decision

If Next.js adds first-class support for streaming rewrites (configurable buffering), the Route Handler could be replaced with a simpler rewrite entry. The behaviour would be identical; the Route Handler exists only because the rewrite layer cannot be configured to stop buffering.

If the backend moved to WebTransport or HTTP/3, the streaming mechanism would need to be revisited. SSE over HTTP/1.1 is the appropriate choice for the current deployment target.
