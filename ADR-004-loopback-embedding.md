# ADR-004: Embedding Strategy — Loopback HTTP to Rust over In-Process PyTorch

**Status:** Accepted  
**Date:** February 2026  
**Component:** `rag-main` (Python)

---

## Context

The RAG pipeline needs to produce text embeddings for two operations:

1. **At write time:** When a note is saved, its content is embedded and upserted into Supabase pgvector
2. **At query time:** The user's query is embedded to perform cosine similarity search against stored note vectors

Both operations require the same embedding model (a sentence transformer producing 384-dimensional vectors). The question is where that model runs and who calls it.

The system constraint is 8 GB total host RAM. The Rust inference engine is already running at startup and already has the embedding model loaded via tract-onnx. The RAG service needs embeddings — but it does not necessarily need to load the model itself.

---

## Decision

The RAG service **does not load the embedding model**. Instead, it calls the Rust engine over loopback HTTP at `localhost:8080/api/v1/predict/embedding`. The Rust engine handles tokenization, ONNX inference, and mean pooling. The RAG service receives the raw float vector and applies L2 normalization before storing or comparing it.

```python
# rag-main/retrieval/embeddings.py
async def embed(text: str) -> list[float]:
    response = await client.post(
        "http://localhost:8080/api/v1/predict/embedding",
        json={"model": "embedding", "input": text}
    )
    vector = response.json()["output"]
    return l2_normalize(vector)   # normalize here, not in Rust
```

---

## The RAM Calculation

This is the core of the decision. Loading `sentence-transformers` in Python requires PyTorch as a dependency. PyTorch's baseline memory footprint — before any model is loaded — is approximately 400-500 MB. The embedding model itself adds another ~90 MB.

```
Without loopback (loading PyTorch in RAG service):
  RAG service baseline:   ~101 MB
  PyTorch baseline:       ~450 MB
  Embedding model:        ~90 MB
  Total RAG footprint:    ~641 MB

With loopback (Rust handles embedding):
  RAG service baseline:   ~101 MB
  PyTorch:                not loaded
  Total RAG footprint:    ~101 MB

Saving:                   ~540 MB
```

On an 8 GB host with Phi-3.5-mini consuming ~4.4 GB at peak, every available megabyte matters. Saving 540 MB is the difference between the system fitting comfortably and running at the edge of available memory.

The Rust engine's memory cost for the embedding model is already paid — it loads the model at startup regardless of whether the RAG service also loads it. There is no additional memory cost for the Rust engine to serve embedding requests. The loopback call delegates work to a resource that is already resident.

---

## Alternatives Considered

### Option A: Load sentence-transformers in the RAG service (standard approach)

The conventional implementation. Import `sentence_transformers`, instantiate `SentenceTransformer("all-MiniLM-L6-v2")`, call `model.encode(text)`.

**Why rejected:** Adds ~540 MB to the RAG service's footprint. On an 8 GB host, this consumes memory that the LLM needs. During peak inference with Phi-3.5-mini loaded, the system would be operating at approximately 5.0 GB used of 8 GB available — still within budget, but only barely. Under any memory pressure (OS, other processes, model warmup), the system could hit the host OOM killer. The loopback approach gives ~540 MB of headroom at no functional cost.

Beyond the RAM cost, loading PyTorch in the RAG service creates a second ONNX/compute runtime running alongside the Rust engine's tract-onnx. Two separate runtimes loading the same model logic increases startup time and makes memory profiling harder.

### Option B: Shared memory / Unix socket between Rust and Python

Instead of HTTP loopback, use a Unix domain socket or shared memory to pass text and receive vectors without TCP stack overhead.

**Why rejected:** The latency of loopback HTTP for a ~384-dimensional vector payload is approximately 1-2 ms. The inference itself is ~1-2 ms. The total round-trip is ~3-4 ms. This is acceptable for both write-time (async, not user-facing) and query-time (batched with vector search, which takes ~85 ms P50). The complexity of implementing shared memory or a custom IPC protocol between Rust and Python is not justified by the latency savings. The loopback HTTP approach reuses the existing REST API contract.

### Option C: Call Supabase's built-in pgvector embedding (delegate to DB)

Supabase supports automatic embedding generation via edge functions or the `pgvector` extension with a pre-configured model.

**Why rejected:** This would require a round-trip to Supabase's infrastructure for every embedding, adding network latency and a hard dependency on external service availability. The system is designed to run offline-capable on a local host. Delegating embedding to a remote service breaks that property. It also means the query embedding and the stored embeddings are not guaranteed to use the same model version — a silent semantic mismatch risk.

### Option D: Loopback HTTP to Rust (chosen)

**Why chosen:** The Rust engine is already running. The embedding model is already loaded. The HTTP API is already defined and tested. Calling it from Python adds ~2 ms per embedding request and ~5 lines of code in `embeddings.py`. The RAM saving is ~540 MB. The implementation and operational cost is minimal; the benefit is concrete and measured.

---

## L2 Normalization

The Rust engine returns the raw mean-pooled embedding vector. L2 normalization is applied in the RAG service before storing or comparing vectors:

```python
def l2_normalize(vector: list[float]) -> list[float]:
    norm = sum(x * x for x in vector) ** 0.5
    if norm == 0:
        return vector
    return [x / norm for x in vector]
```

**Why normalize in Python rather than Rust:** The Rust engine is a general-purpose inference server. It returns the model output as-is without post-processing. L2 normalization is a retrieval-specific semantic choice — it converts dot product similarity to cosine similarity. This belongs in the retrieval layer (Python), not the inference layer (Rust). If a future use case requires unnormalized vectors, the Rust engine does not need to change.

Supabase pgvector's `<=>` operator computes cosine distance on stored vectors. Storing L2-normalized vectors means `<=>` is equivalent to negative dot product, which is slightly faster to compute. The normalization is applied consistently at both write time (when storing note embeddings) and query time (when embedding the search query), ensuring the similarity metric is correct.

---

## Performance Profile

```
Embedding call (loopback):   ~2 ms P50, ~4 ms P95
Rust engine internal:        ~1 ms tokenization + inference + mean pool
HTTP overhead:               ~1 ms (loopback TCP, serialization)
```

For query-time retrieval, the full pipeline is:

```
1. Embed query:              ~2 ms  (loopback to Rust)
2. Vector search:            ~85 ms (Supabase pgvector)
3. LLM inference (TTFT):    <100 ms
Total to first token:        ~190 ms
```

The loopback embedding adds ~2 ms to a ~185 ms pipeline — a 1% overhead. This is not a performance concern.

For write-time embedding (note save), the operation is async and not on the critical path for the editor response. Latency is irrelevant.

---

## Consequences

**Positive:**
- ~540 MB RAM saved in the RAG service — the most impactful single optimization for the memory budget
- No PyTorch dependency in the RAG service — faster startup, simpler dependency tree, smaller container image if deployed
- Embedding model is loaded once at startup by the Rust engine — no duplication of model weights in memory
- L2 normalization applied consistently at the retrieval layer — correct cosine similarity semantics

**Negative:**
- The RAG service has a runtime dependency on the Rust engine. If the Rust engine is not running, embedding calls fail. Mitigated by the graceful degradation path: embedding failures are caught, the editor shows "AI Engine Offline", note saving continues without embedding
- The loopback contract (endpoint URL, response schema) must stay in sync between the Rust engine and the RAG service. Currently maintained manually — a shared OpenAPI spec would be better
- ~2 ms overhead per embedding call versus in-process. Acceptable at current scale

---

## What Would Change This Decision

If the system moved to a deployment where the Rust engine and RAG service run in separate containers without loopback access (e.g. Kubernetes pods on different nodes), the loopback approach would not work. In that case, the RAG service would need its own embedding model. The RAM budget would need to be re-evaluated against the available pod memory.

If the embedding model changed to one not supported by tract-onnx, the Rust engine could not run it and the loopback approach would break. The current model (a MiniLM-family sentence transformer exported to ONNX) is well within tract-onnx's supported operator set.
