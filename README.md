# BorgOS

Fleet memory architecture for multi-agent AI systems. Resistance is futile — your memories will be assimilated.

## What Is This

A shared memory system for AI agent fleets. Four layers, one validated write path, confidence-weighted recall, anomaly detection, and offline maintenance. Agent-agnostic — works with any agent framework (Agent Zero, CrewAI, LangGraph, custom).

## The Problem

Every team running multiple AI agents hits the same wall: agents don't share what they learn. Agent A discovers a fact, Agent B rediscovers it three days later. Memories are siloed, unvalidated, and ungoverned. There's no confidence system, no anomaly detection, no lifecycle management.

BorgOS solves this with a four-layer memory architecture that gives every agent in your fleet shared, validated, confidence-weighted memory.

## Architecture

```
L4: REFERENCE    — Curated ground truth (read-only, human-managed)
L1: PROVENANCE   — Forensic audit trail (append-only, Postgres)
L2: SEMANTIC     — Shared recall (Qdrant, cross-agent, confidence-weighted)
L3: WORKING      — Agent-local scratchpad (fast, ephemeral, per-agent)
```

### Key Features

- **12-stage write validation pipeline** — every memory passes through auth, rate limiting, novelty detection, confidence validation, provenance checking, and anomaly scoring before it's stored
- **Confidence system** — memories have confidence tiers (hypothesis/experimental/verified). Confidence can only increase via external validation, not self-promotion
- **Anti-laundering** — detects circular confidence boosting across agents (Agent A writes claim, Agent B retrieves and "corroborates" it)
- **Anomaly alerter** — 10 detectors monitoring write patterns for volume spikes, confidence manipulation, self-referential floods, identity injection, and more
- **Intent-aware recall** — queries route to different layers based on intent (factual, procedural, causal, exploratory)
- **Dream Cycle** — offline batch maintenance: vitality decay, dedup, archival, health grading
- **Layer priority on collision** — when reference knowledge and agent-written knowledge cover the same topic, reference wins

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture Design](docs/architecture.md) | Full technical design (1,700+ lines) |
| [API Reference](docs/api.md) | `/insert` and `/query` endpoint contracts |

## Tech Stack

| Component | Technology |
|-----------|-----------|
| L1 Provenance | PostgreSQL (append-only, bitemporal) |
| L2 Semantic | Qdrant (hybrid BM25 + dense, API key auth) |
| L3 Working | Markdown + FAISS (agent-local) |
| L4 Reference | Host-mounted read-only files + dedicated Qdrant collection |
| Embeddings | Snowflake arctic-embed-l-v2.0 (1024-dim) |
| Write endpoint | FastAPI (memory-query service) |
| Maintenance | Python systemd timer (Dream Cycle) |

## Status

**Design phase.** The architecture is specified and council-reviewed (4-agent adversarial debate, 3 rounds). Implementation sequence defined. Not yet deployed.

## Implementation Sequence

| Phase | What | Gate |
|-------|------|------|
| Day 0 | Firewall vector DB port | Network isolation verified |
| v1 | Schema + write endpoint + reference tier + anomaly alerter | Pipeline unit tests passing |
| v1 soak | 30 days single-agent writes, threshold tuning | Zero undetected anomalies |
| v1.1 | Multi-agent write paths + memory consolidation | Quality gate with negative tests |
| v2 | Dream Cycle + enhanced recall + knowledge graph | v1.1 stable 30 days |

## License

MIT

## Origin

Designed for a 5-node AI agent mesh (3 agents across dedicated hosts). Battle-tested architecture decisions informed by evaluation of 6 open-source memory systems (MemKraft, ClawMem, Memory Palace, Claude-Context, LeanCtx, ContextKeep) and 40+ cherry-picked patterns.
