# ADR-001: Bounded Job Queue — Depth 20, Immediate Rejection

**Status:** Accepted  
**Date:** January 2026  
**Component:** `infer-engine` (Rust)

---

## Context

The inference engine accepts HTTP requests and dispatches them to a fixed worker pool for ONNX model execution. The core scheduling question is: what happens when requests arrive faster than workers can process them?

There are three broad approaches:

1. **Unbounded queue** — accept all requests, let them pile up, process them in order
2. **Bounded queue with blocking** — accept requests up to a limit, then block the HTTP handler until a slot opens
3. **Bounded queue with immediate rejection** — accept requests up to a limit, then return an error immediately

The system runs on a single 8 GB host with no auto-scaling. Under a traffic spike, the host cannot add capacity. Any approach that accepts unbounded work will eventually exhaust memory or starve workers.

---

## Decision

Use a **crossbeam bounded channel with depth 20** and `try_send` for non-blocking submission. When the channel is full, the request is rejected immediately with HTTP 503. The queue never blocks the HTTP accept loop.

```
Queue depth:  20
Workers:      4
Send method:  try_send (non-blocking)
Full policy:  immediate 503
```

---

## Alternatives Considered

### Option A: Unbounded queue

The simplest implementation. Every incoming request gets queued regardless of worker availability.

**Why rejected:** Under sustained overload, the queue grows without bound. Memory consumption grows linearly with queue depth. On a fixed 8 GB host with no vertical scaling, this eventually OOMs the process or starves the model of memory mid-inference. The failure mode is non-deterministic and difficult to observe until it's catastrophic. An unbounded queue converts a traffic problem into a resource exhaustion problem, which is strictly harder to recover from.

### Option B: Bounded queue with blocking send

Cap the queue at some depth. When full, block the HTTP handler thread until a worker frees a slot.

**Why rejected:** Blocking the HTTP handler under an async runtime (Tokio) is unsafe without explicit thread offloading — it starves the executor. Even with offloading, a blocked handler means the server appears unresponsive to the caller. The caller's client typically has its own timeout, so the caller retries. Retries increase load on an already-overloaded system. A blocking queue under overload creates a feedback loop: more load → more blocking → slower processing → more retries → more load. The queue becomes a mechanism for amplifying overload rather than absorbing it.

### Option C: Bounded queue with immediate rejection (chosen)

Cap the queue. Reject overflow immediately with a well-defined error.

**Why chosen:** The server's behavior under overload is explicit and observable. Callers get a 503 immediately, not after a timeout. A 503 is a signal the caller can act on — back off, retry later, route elsewhere. The server's latency characteristics remain stable under overload because the queue never grows. Memory use is bounded. Worker throughput is unaffected by the queue depth.

---

## Sizing the Depth

The queue depth is not arbitrary. It is derived from the target maximum queue wait:

```
Workers:          4
Warm inference:   ~5 ms per request (P50, NoOpModel baseline)
Queue drain time: 4 workers × 5 ms = 20 ms to process one full queue
Target max wait:  ~100 ms before rejection is preferable to waiting

Depth = target_wait / drain_time = 100 ms / 20 ms = 5 slots per worker = 20 total
```

At depth 20, a request that enters the queue when it is full waits approximately 100 ms before being processed. This is the upper bound of acceptable latency for an interactive RAG query. Requests beyond depth 20 would wait longer than the user-facing timeout budget, so rejecting them is preferable to queuing them.

The depth was not tuned empirically — it was derived from the constraint and then verified.

---

## Verification

The `brick_wall` integration test was written specifically to verify this behavior under stress:

- 50 concurrent requests against an engine configured with 2 workers and depth 0
- Expected result: exactly 2 succeed (one per worker), 48 receive clean 503s
- Actual result: matches expected

The test is deterministic. It does not rely on timing or sleep — it uses a depth-0 queue to force immediate rejection of all but the in-flight requests.

```
test test_brick_wall_and_metrics ... ok

Results -> 200 OK: 2, 503 Service Unavailable: 48
```

---

## Consequences

**Positive:**
- Server latency is stable under overload — queue depth is bounded, so P99 latency does not balloon
- Caller receives actionable error (503) immediately rather than timing out silently
- Memory consumption is bounded — the queue cannot grow unboundedly
- Rejection behavior is observable via Prometheus counter `request_dropped_total{reason="queue_full"}`
- Worker throughput is unaffected — workers process at full speed regardless of rejection rate

**Negative:**
- No burst absorption — a short traffic spike (e.g. 25 concurrent requests for 50 ms) will cause rejections even if the system could handle the load spread over a longer window
- Callers must implement retry-with-backoff to handle 503s gracefully — the server cannot buffer requests on their behalf
- Queue depth of 20 is correct for current worker count and inference latency. If either changes, the depth should be recalculated

---

## What Would Change This Decision

If the system gained horizontal scaling (multiple engine instances behind a load balancer), the queue depth calculation would be per-instance but the rejection policy would remain the same — the load balancer handles distribution, each instance handles its own backpressure.

If inference latency dropped significantly (e.g. GPU acceleration bringing P50 to sub-millisecond), the optimal queue depth would increase proportionally. At 0.5 ms per inference, 4 workers drain 20 requests in 2 ms, allowing a much deeper queue for the same 100 ms wait budget.
