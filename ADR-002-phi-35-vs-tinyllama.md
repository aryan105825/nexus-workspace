# ADR-002: LLM Selection — Phi-3.5-mini over TinyLlama 1.1B

**Status:** Accepted  
**Date:** February 2026  
**Component:** `rag-main` (Python)

---

## Context

The RAG pipeline runs a local LLM for query answering. The system constraint is strict: 8 GB total host RAM, CPU only, no GPU. The LLM must fit in memory alongside the Rust engine (~200 MB) and the RAG service baseline (~101 MB), leaving approximately 7.7 GB available for the model and its KV cache.

Two quantized models were evaluated: TinyLlama 1.1B and Phi-3.5-mini-instruct 3.8B.

The evaluation criteria were:
1. Does it fit in the RAM budget?
2. Does it answer correctly when multiple named entities appear in the same context window?
3. Does it stay grounded — refusing to answer when context is insufficient rather than fabricating?

---

## Decision

Use **Phi-3.5-mini-instruct at Q4_K_M quantization** (3.8B parameters).

```
Model:         Phi-3.5-mini-instruct
Quantization:  Q4_K_M
Parameters:    3.8B
Peak RSS:      ~4.4 GB (model + KV cache at n_ctx=2048)
Inference:     ~115 tok/s, TTFT <100 ms (warm)
```

---

## Alternatives Considered

### Option A: TinyLlama 1.1B Chat (Q4_K_M)

TinyLlama was evaluated first because it fits in approximately 1.5 GB, leaving ~6 GB free for other services and future growth.

**Why rejected:** TinyLlama failed consistently on the primary use case — answering questions about notes when multiple named entities are present in the retrieved context.

**Observed failure pattern:** When two notes about distinct topics were retrieved together (e.g. a note about React hooks and a note about database indexing), TinyLlama would fabricate relationships between the entities that did not exist in either source document. Example:

> *Prompt:* "What did I write about React hooks?"  
> *Context:* [Note A: React hooks explanation] + [Note B: B-tree index explanation]  
> *TinyLlama output:* "React hooks can be used to manage B-tree index operations in your component state, similar to how database indices work..."

The model was not hallucinating randomly — it was confabulating a coherent-sounding relationship between the two retrieved topics. This is structurally different from a factual error. The model cannot maintain entity boundary separation when both entities are present in the same attention window.

**Prompt engineering attempts and results:**

| Attempt | Result |
|---|---|
| "Answer using only the provided notes" | No improvement — model ignores grounding instruction |
| Explicit negative instruction: "Do not connect unrelated topics" | Marginal improvement on obvious cases, failure persists on subtle ones |
| Reduced temperature to 0.1 | Reduced variance but did not eliminate cross-entity confabulation |
| Retrieved only the top-1 chunk instead of top-3 | Eliminated the failure by removing multi-entity context — but also eliminated correct answers that required multiple chunks |

None of the prompt-based mitigations resolved the root problem without sacrificing retrieval quality. The conclusion after testing: this is a model scale problem. At 1.1B parameters, TinyLlama does not have sufficient attention capacity to track entity provenance across a multi-document context window. The failure is deterministic and model-intrinsic.

### Option B: Phi-3.5-mini-instruct 3.8B Q4_K_M (chosen)

**Why chosen:** In all test cases that caused TinyLlama to hallucinate, Phi-3.5-mini answered correctly, attributing information to the correct source entity and explicitly noting when topics were unrelated.

> *Same prompt, same context:*  
> *Phi-3.5 output:* "Based on your notes, React hooks are used for managing component state and side effects. Your note on B-tree indices is a separate topic about database optimization and is not related to your React notes."

The model correctly identifies the entity boundary and refuses to fabricate a connection.

**RAM cost:** Phi-3.5-mini at Q4_K_M occupies ~4.4 GB peak RSS at n_ctx=2048. This is within the 7.7 GB available on the 8 GB host but leaves only ~3.3 GB free. This is tight. The decision to use n_ctx=2048 rather than the model's full 131k context window was made specifically to keep RSS manageable — see ADR-003 (n_ctx sizing) for that calculation.

---

## The Core Tradeoff

| Dimension | TinyLlama 1.1B | Phi-3.5-mini 3.8B |
|---|---|---|
| RAM (peak RSS) | ~1.5 GB | ~4.4 GB |
| Cross-entity accuracy | Fails | Correct |
| Grounding (stays in context) | Weak | Strong |
| Token throughput | ~180 tok/s | ~115 tok/s |
| Cold start | ~3 s | ~7.4 s |
| Fit on 8 GB host | Yes (comfortable) | Yes (tight) |

The performance gap is real. Phi-3.5-mini is slower and uses three times the memory. But the quality gap is also real and non-recoverable via prompting. A faster model that fabricates relationships between the user's own notes is not useful — it's actively misleading.

---

## Consequences

**Positive:**
- Multi-entity RAG answers are correct — the primary use case works as intended
- The model stays grounded when context is insufficient — returns "I don't have notes on that" rather than fabricating
- Response quality is consistent across query types (factual, relational, comparative)

**Negative:**
- Peak RSS ~4.4 GB leaves ~3.3 GB headroom on an 8 GB host — tight for a production system
- Cold start ~7.4 s is long for an interactive application. Mitigated by loading the model at service startup and keeping it resident
- Token throughput ~115 tok/s is slower than TinyLlama's ~180 tok/s. Acceptable for a single-user system; would require horizontal scaling for concurrent users

---

## What Would Change This Decision

If RAM budget increased (larger host or GPU VRAM), Phi-3.5-mini could be replaced with a larger model for better quality. The selection criterion — correct multi-entity grounding — would remain the same.

If a 1-2B parameter model emerged with significantly improved instruction following and multi-document grounding, TinyLlama could be reconsidered. The failure observed was specific to that model's capacity, not an intrinsic property of sub-2B models.

If the use case shifted to single-document Q&A (one note retrieved at a time), TinyLlama's failure mode would not manifest and the RAM savings would justify using it.
