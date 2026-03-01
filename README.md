# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for the Nexus AI system. Each ADR documents a significant technical decision: the context that forced it, the alternatives considered and rejected, the decision made, and its consequences.

ADRs are written after the fact but reflect decisions made during development. They are intended to explain *why* the system is built the way it is, not just *what* it does.

---

## Index

| ADR | Component | Decision |
|-----|-----------|----------|
| [ADR-001](ADR-001-bounded-queue-depth.md) | `infer-engine` | Bounded job queue with depth 20 and immediate rejection over unbounded or blocking alternatives |
| [ADR-002](ADR-002-phi-35-vs-tinyllama.md) | `rag-main` | Phi-3.5-mini-instruct (3.8B) over TinyLlama (1.1B) — model scale required for multi-entity RAG grounding |
| [ADR-003](ADR-003-sse-route-handler-bypass.md) | `nexus-frontend` | Route Handler bypass for SSE streaming over Next.js rewrites — rewrites buffer response bodies |
| [ADR-004](ADR-004-loopback-embedding.md) | `rag-main` | Embedding via loopback HTTP to Rust over loading PyTorch in-process — saves ~540 MB baseline RAM |

---

## System Constraint

Every ADR in this repository is shaped by the same root constraint:

> **8 GB host RAM, CPU only, no GPU, no auto-scaling.**

Decisions that would be straightforward on a larger host (load every model in-process, accept all requests, use the largest available model) require explicit justification here. The constraint is not incidental — it is the design condition that makes the interesting decisions interesting.

---

## Format

Each ADR follows a consistent structure:

- **Context** — what problem existed and what the options were
- **Decision** — what was chosen and at what configuration
- **Alternatives Considered** — what was rejected and why
- **Consequences** — what the decision cost and what it gained
- **What Would Change This Decision** — under what conditions the decision would be revisited
