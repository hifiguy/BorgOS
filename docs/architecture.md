---
title: "BorgOS Architecture Overview"
status: draft
---

# BorgOS Architecture Overview

Fleet memory architecture for multi-agent AI systems. Four layers, one validated write path, confidence-weighted recall, anomaly detection, and offline maintenance.

This document describes the *what* and *why*. The full implementation design — schemas, code, pipeline internals, council review findings — is available under consulting engagement.

---

## Four-Layer Architecture

Four layers serving all agents in a fleet. Each layer has distinct mutability, trust level, and purpose.

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 4: REFERENCE                           │
│                 (Ground Truth — operator-curated)               │
│                                                                 │
│  Host-mounted read-only files. Kernel-enforced immutability.    │
│  Confidence: 1.0 (verified, agents cannot modify)               │
│  Purpose: curated knowledge the fleet treats as fact             │
│  Agent proposals: agents can suggest changes; operator reviews   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                    LAYER 1: PROVENANCE                           │
│                 (Forensic Record — append-only)                  │
│                                                                 │
│  PostgreSQL with bitemporal columns.                            │
│  Append-only: no UPDATE, no DELETE on the facts table.          │
│  Two write roles enforce least-privilege (writer vs closer).     │
│  Purpose: permanent forensic audit trail for every memory        │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                    LAYER 2: SEMANTIC                             │
│                 (Shared Recall — cross-agent)                    │
│                                                                 │
│  Qdrant with hybrid BM25 + dense vector search.                 │
│  Per-agent API keys. All writes via validated endpoint only.     │
│  Confidence-weighted ranking with per-agent trust tiers.         │
│  Purpose: shared fleet memory — query, recall, cross-pollinate   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                    LAYER 3: WORKING                              │
│                 (Agent-Local — per-agent, ephemeral)             │
│                                                                 │
│  Markdown + FAISS per agent. Fast, session-scoped.              │
│  Promotes to L2 via session-end consolidation.                   │
│  Maintained by Dream Cycle batch operations.                     │
│  Purpose: agent-local scratchpad and working context             │
└─────────────────────────────────────────────────────────────────┘
```

**Layer precedence on collision:** L4 > L2 > L3. When reference knowledge and agent-written knowledge cover the same topic, reference wins. Agent versions get a retrieval penalty.

---

## 12-Stage Write Validation Pipeline

Every write to shared memory passes through a 12-stage pipeline. Stages are ordered so cheap checks reject early, expensive checks run last.

| Stage | Check | Action on Failure |
|-------|-------|-------------------|
| 1 | API key authentication | Reject: UNAUTHORIZED |
| 2 | Rate limiting (per-agent) | Reject: RATE_LIMITED |
| 3 | Content sanitization | Reject: injection detected |
| 4 | Self-referential detection | Tag (not reject) — downstream handling |
| 5 | Embedding generation | — |
| 6 | L4 shadow rejection | Reject: content shadows reference knowledge |
| 7 | Novelty / dedup gate | Skip (>0.92 cosine), Merge (0.85-0.92), or Add |
| 8 | Confidence validation | Downgrade if agent exceeds allowed tier |
| 9 | Provenance / anti-laundering check | Reject: circular provenance detected |
| 10 | Anomaly scoring (10 detectors) | Reject if score > 0.8 |
| 11 | Bitemporal metadata assembly | — |
| 12 | Dual write (Postgres-first, then Qdrant) | PG is authoritative; Qdrant eventually consistent |

**Write outcomes:** `ADD` (new memory stored), `MERGE` (combined with similar existing memory), `SKIP` (duplicate detected), `REJECT` (blocked by pipeline).

---

## Confidence & Corroboration System

Memories have confidence tiers that can only increase via external validation. An agent cannot self-promote its own claims.

| Tier | Score Range | Meaning | Who Can Set |
|------|------------|---------|-------------|
| `hypothesis` | 0.0-0.49 | Agent believes this, no external validation | Any agent (default) |
| `experimental` | 0.5-0.79 | Supporting evidence from different agent | Requires cross-agent corroboration |
| `verified` | 0.8-1.0 | Externally validated or operator-confirmed | Operator or multi-agent consensus only |

**Key properties:**

- **Self-referential cap:** Agents writing about themselves are capped at `hypothesis`
- **Cross-agent corroboration required:** Upgrading to `experimental` requires `derived_from` pointing to a memory from a *different* agent
- **Anti-laundering (3-check):** Detects circular confidence boosting — Agent A writes claim, Agent B retrieves and "corroborates" it. Three checks: shared ancestry (3-hop), high-similarity ancestor detection, and time-based suspicion for rapid round-trip patterns
- **Content-type decay:** Different memory types decay at different rates (identity: 365 days, observation: 30 days, transient: 7 days)

---

## Anomaly Detection

10 anomaly detectors monitor the write path for suspicious patterns, each contributing a weighted score:

| # | Detector | What It Catches |
|---|----------|----------------|
| 1 | Volume spike | Writes > 3σ above hourly average |
| 2 | Confidence manipulation | Requested confidence exceeds agent's allowed tier |
| 3 | Semantic drift | Embedding far from agent's recent centroid |
| 4 | Self-referential flood | Excessive self-referential writes in short window |
| 5 | Corroboration laundering | `derived_from` traces back to same agent |
| 6 | Content-type abuse | Writing content types the agent is not authorized for |
| 7 | Rapid-fire | Burst writes from same agent |
| 8 | Identity injection | Content contains identity-altering phrases |
| 9 | Migration mismatch | Phase tag doesn't match environment |
| 10 | Merge storm | Excessive merge actions (may be eroding existing memories) |

**Score thresholds:**

| Score | Action |
|-------|--------|
| 0.0-0.3 | Silent — write proceeds |
| 0.3-0.5 | Flagged — write proceeds, logged for review |
| 0.5-0.8 | Warned — write proceeds, immediate alert |
| 0.8-1.0 | Rejected — write blocked |

---

## Intent-Aware Recall

Queries are classified by intent and routed to different layer combinations with per-intent weight matrices:

| Intent | Primary Layers | Strategy |
|--------|---------------|----------|
| Factual | L4, L2 | High L4 weight, verified-first ranking |
| Procedural | L2, L3 | Skill and decision memories prioritized |
| Causal | L1, L2 | Provenance chain traversal |
| Exploratory | L2, L3 | Broad recall, lower confidence threshold |

Context-aware query enhancement reformulates raw queries using the agent's current session context, task, and recent conversation — improving recall relevance without requiring agents to craft perfect queries.

---

## Dream Cycle — Batch Maintenance

Offline maintenance operations inspired by biological memory consolidation. Runs during quiet hours with pre-flight checks to avoid interfering with active sessions.

| # | Operation | Purpose |
|---|-----------|---------|
| 1 | Vitality decay | Recalculate freshness scores for all shared memories |
| 2 | Deduplication | Merge memories above similarity threshold |
| 3 | Archive | Move stale, low-vitality memories to cold storage |
| 4 | L3 → L2 promotion | Promote qualifying local memories to shared pool |
| 5 | Conflict detection | Find contradicting facts, surface for operator review |
| 6 | Health check | Grade overall memory corpus quality (A/B/C/D) |
| 7 | Snapshot cleanup | Remove old consolidation snapshots |

---

## Memory Consolidation

Session-end consolidation merges an agent's working memory into structured facts for the shared pool. Quality-gated with rollback via pre-consolidation snapshots.

**Quality checks:** Length ratio (no aggressive summarization), entity preservation (no dropped names/values), semantic fidelity (consolidated meaning matches originals).

---

## Cross-Agent Memory Flow

Each agent in the fleet has its own API key with defined permissions:

- **Allowed content types** per agent (not all agents can write all types)
- **Confidence tier caps** per agent (trust is earned, not assumed)
- **Rate limits** per agent
- **Migration phase tagging** for staged rollouts

Agents write to their local L3 first, then promote to shared L2 through the validated write endpoint. No agent writes directly to the vector database.

---

## Implementation Approach

Phased rollout with explicit gates between phases:

| Phase | What | Gate |
|-------|------|------|
| **Day 0** | Firewall vector DB port | Network isolation verified |
| **v1** | Schema + write endpoint + reference tier + anomaly alerter | Pipeline unit tests passing |
| **v1 soak** | 30-day single-agent writes, threshold tuning | Zero undetected anomalies |
| **v1.1** | Multi-agent write paths + memory consolidation | Quality gate with negative tests |
| **v2** | Dream Cycle + enhanced recall + knowledge graph | v1.1 stable 30 days |

---

## What's Not in This Document

The full implementation design includes:

- Complete SQL schemas with bitemporal columns and role-based access
- Python implementation of all 12 pipeline stages
- Qdrant payload schema (18 fields)
- Merge logic with cosine-zone handling
- Confidence validation and anti-laundering code
- Multi-source fact reconciliation algorithms
- Anomaly scoring engine implementation
- Dream Cycle batch processing code
- Context-aware query enhancement pipeline
- Per-agent API key configuration
- Council review findings (4-agent adversarial debate, 3 rounds)
- Integration diagrams and cherry-pick implementation maps

Available under consulting engagement. Contact: [AI Ascent](mailto:joshua.r.kimmel@gmail.com)

---

## Design Provenance

Architecture decisions informed by evaluation of 6 open-source memory systems and 40+ cherry-picked patterns. Validated through a 4-agent adversarial council review (3 rounds) covering security, operations, research, and technical skepticism perspectives. Key patterns borrowed and improved: bitemporal fact storage, confidence-weighted recall, anti-laundering provenance tracking, content-type-aware vitality decay, and intent-classified query routing.
