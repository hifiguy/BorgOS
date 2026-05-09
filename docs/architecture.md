---
title: "Fleet Memory Architecture Design"
date: 2026-05-09
status: draft
scope: All agents — AgentAlpha, AgentBeta, AgentGamma, future fleet members
related:
  - docs/agent-design-brief.md (§1.4-1.5 sketch this replaces)
  - docs/priorities.md (JRP-001, JRP-006)
  - docs/cherry-pick-tracker.md
  - docs/memkraft-evaluation.md
  - docs/clawmem-evaluation.md
  - docs/memory-repos-evaluation.md
  - docs/claude-context-evaluation.md
---

# Fleet Memory Architecture Design

A shared memory system for all agents in the mesh. Four layers, one write path, confidence-weighted recall, anomaly detection, and offline maintenance. Agent-agnostic — AgentAlpha is the first consumer, not the only one.

---

## Table of Contents

1. [Four-Layer Architecture](#1-four-layer-architecture)
2. [Memory Data Schema](#2-memory-data-schema)
3. [Write Path](#3-write-path)
4. [Recall Path](#4-recall-path)
5. [Cross-Agent Memory Flow](#5-cross-agent-memory-flow)
6. [Confidence & Corroboration System](#6-confidence--corroboration-system)
7. [Anomaly Alerter (JRP-006 Phase 3)](#7-anomaly-alerter-jrp-006-phase-3)
8. [Memory Lifecycle & Retention](#8-memory-lifecycle--retention)
9. [L4 Reference Tier — Detailed Design](#9-l4-reference-tier--detailed-design)
10. [Memory Consolidation](#10-memory-consolidation)
11. [Dream Cycle — Batch Maintenance](#11-dream-cycle--batch-maintenance)
12. [INDEX.md Context Routing](#12-indexmd-context-routing)
13. [Integration Diagram](#13-integration-diagram)
14. [Cherry-Pick Implementation Map](#14-cherry-pick-implementation-map)
15. [Implementation Sequence](#15-implementation-sequence)

---

## 1. Four-Layer Architecture

Four layers serving ALL agents in the fleet. Each layer has distinct mutability, trust level, and purpose.

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 4: REFERENCE                           │
│                 (Ground Truth — the operator-curated)                    │
│                                                                  │
│  Host: /opt/agent/usr/knowledge/{persona}/reference/           │
│  Mount: bind :ro — kernel-enforced, agent CANNOT modify          │
│  Index: Qdrant collection `{agent}_reference_v1` (read-only key) │
│  Confidence: 1.0 (verified, immutable by agent)                  │
│  Purpose: curated knowledge the agent treats as fact              │
│  Examples: ARCHITECTURE_AND_OPS.md, NETWORK_AND_SERVICES.md,     │
│    SECURITY_AWARENESS.md, SELF_MODIFICATION_POLICY.md            │
│  Curation: the operator edits on host → inotifywait triggers reindex     │
│  Agent proposals: writes to l4-proposals/ dir, the operator reviews      │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                    LAYER 1: PROVENANCE                           │
│                 (Forensic Record — JRP-001)                      │
│                                                                  │
│  Host: a0-postgres on state-host (YOUR_STATE_HOST_IP:5432)        │
│  Database: claude_code_sessions (97,788+ turns, 1,608+ sessions) │
│  Database: a0_memory (NEW — structured facts)                    │
│  Write: append-only (no UPDATE, no DELETE on turns table)        │
│  Write: memory_facts table supports fact supersession only       │
│  Read: future RAG via JRP-001 v0.2, causal queries               │
│  Also: LiteLLM_SpendLogs (cost), audit.db (JRP-008)             │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                    LAYER 2: SEMANTIC                             │
│                 (Shared Recall — cross-agent)                    │
│                                                                  │
│  Host: a0-qdrant on state-host (YOUR_STATE_HOST_IP:6333)          │
│  Collection: gateway_transcripts_v2 (hybrid BM25 + dense)        │
│  Embedding: Snowflake arctic-embed-l-v2.0 (1024-dim)            │
│  API key: per-agent keys (DM-8), enforced at memory-query:5100   │
│  Write: ALL writes via memory-query:5100 /insert (never direct)  │
│  Read: memory-query:5100 /query (existing, enhanced)             │
│  Trust: GAP-01 Phases 0-2.1 (deployed)                          │
│  Confidence: per-memory, per-agent-source weighted               │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                    LAYER 3: WORKING                              │
│                 (Agent-Local — per-agent, ephemeral)             │
│                                                                  │
│  AgentAlpha: /a0/usr/memory/godmode/ (markdown + FAISS)               │
│  AgentBeta: state.db SQLite (121 sessions, 799 messages)            │
│  AgentGamma: Langfuse traces (agent activity logs)                 │
│  Purpose: fast, session-scoped, agent-specific context           │
│  Lifetime: survives restarts (bind mount), vitality-gated        │
│  Consolidation: promotes to L2 via session_end hook              │
│  Maintenance: Dream Cycle prunes, merges, archives               │
└─────────────────────────────────────────────────────────────────┘
```

**Layer precedence on topic collision:** L4 > L2 > L3. If L4 and L3 both contain information about network topology, the L4 version (the operator-curated, `:ro`) is authoritative. L3 versions on the same topic get a retrieval penalty. L2 cross-agent memories rank by confidence score.

---

## 2. Memory Data Schema

### 2.1 L1 Postgres — `memory_facts` Table

New table in a new `a0_memory` database on state-host. Separate from `claude_code_sessions` (JRP-001 raw transcripts) to maintain single-responsibility.

```sql
CREATE DATABASE a0_memory;

CREATE TABLE memory_facts (
    -- Identity
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fact_id         TEXT NOT NULL,           -- human-readable, e.g. "agent-alpha-net-topology-001"

    -- Content
    content         TEXT NOT NULL,
    content_type    TEXT NOT NULL CHECK (content_type IN (
        'observation', 'decision', 'fact', 'belief',
        'skill', 'identity', 'relationship', 'transient'
    )),
    summary         TEXT,                    -- ≤200 char summary for index/search

    -- Source & Provenance
    source_agent    TEXT NOT NULL,           -- agent-alpha, agent-beta, agent-gamma-{n}
    source_session  TEXT,                    -- session ID where fact originated
    source_task     TEXT,                    -- task/ISA slug if applicable
    derived_from    UUID[],                  -- parent memory IDs (DM-26 provenance)

    -- Bitemporal (#282)
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- when fact became true
    valid_to        TIMESTAMPTZ,                         -- when fact stopped being true (NULL = current)
    system_time     TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- when system recorded it
    assertion_time  TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- when agent asserted it

    -- Confidence (CP-197)
    confidence      TEXT NOT NULL DEFAULT 'hypothesis' CHECK (confidence IN (
        'hypothesis', 'experimental', 'verified'
    )),
    confidence_score FLOAT NOT NULL DEFAULT 0.5 CHECK (confidence_score BETWEEN 0.0 AND 1.0),
    confidence_log  JSONB DEFAULT '[]',      -- audit trail of confidence changes

    -- Classification
    self_referential BOOLEAN NOT NULL DEFAULT FALSE,  -- DM-6: agent writing about itself
    layer           INTEGER NOT NULL DEFAULT 1,
    migration_phase INTEGER,                 -- DM-14: deployment phase tag
    anomaly_score   FLOAT DEFAULT 0.0,       -- JRP-006 Phase 3 score at write time

    -- Metadata
    tags            TEXT[],
    embedding_id    UUID,                    -- FK to Qdrant point ID (if synced to L2)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    superseded_by   UUID REFERENCES memory_facts(id)  -- fact chain
);

-- Indices
CREATE INDEX idx_facts_agent ON memory_facts(source_agent);
CREATE INDEX idx_facts_type ON memory_facts(content_type);
CREATE INDEX idx_facts_confidence ON memory_facts(confidence);
CREATE INDEX idx_facts_valid ON memory_facts(valid_from, valid_to);
CREATE INDEX idx_facts_self_ref ON memory_facts(self_referential) WHERE self_referential = TRUE;
CREATE INDEX idx_facts_derived ON memory_facts USING GIN(derived_from);

-- Roles (least privilege)
CREATE ROLE memory_writer WITH LOGIN PASSWORD '...';
GRANT INSERT ON memory_facts TO memory_writer;
GRANT SELECT ON memory_facts TO memory_writer;

CREATE ROLE memory_closer WITH LOGIN PASSWORD '...';
GRANT UPDATE (valid_to, superseded_by) ON memory_facts TO memory_closer;
-- memory_closer can only close facts, not modify content
```

**Design rationale:** Two write roles enforce append-only semantics. `memory_writer` can insert new facts and query existing ones but cannot update or delete. `memory_closer` can only set `valid_to` (fact supersession) and `superseded_by` (chain to replacement). No role can DELETE rows — facts are permanent forensic records.

### 2.2 L2 Qdrant — Enhanced Payload Schema

Extends existing `gateway_transcripts_v2` collection with new metadata fields. No schema migration needed — Qdrant payloads are schemaless.

```python
MEMORY_PAYLOAD_SCHEMA = {
    # Existing fields (unchanged)
    "text": str,                    # memory content
    "source": str,                  # origin file/session
    "timestamp": str,               # ISO 8601

    # Source & Provenance (NEW)
    "source_agent": str,            # "agent-alpha" | "agent-beta" | "agent-gamma-{n}"
    "source_session": str,          # session ID
    "derived_from": list[str],      # parent Qdrant point IDs (DM-26)
    "fact_id": str,                 # links to L1 memory_facts.fact_id

    # Content Classification (NEW)
    "content_type": str,            # 8-type taxonomy from §2.1
    "self_referential": bool,       # DM-6
    "layer": int,                   # 2 for standard, 4 for reference

    # Confidence (NEW — CP-197, #282)
    "confidence": str,              # "hypothesis" | "experimental" | "verified"
    "confidence_score": float,      # 0.0-1.0
    "agent_confidence_tier": float, # per-agent: agent-alpha=0.8, agent-beta=0.6, agent-gamma=0.4

    # Bitemporal (NEW — #282)
    "valid_from": str,              # ISO 8601
    "valid_to": str | None,         # NULL = current

    # Operational (NEW)
    "migration_phase": int | None,  # DM-14: deployment phase tag
    "anomaly_score": float,         # score at write time
    "vitality": float,              # current vitality (decays over time)
    "last_accessed": str,           # ISO 8601, updated on recall
    "access_count": int,            # recall counter (capped at 5 for anti-gaming)
}
```

### 2.3 L3 Agent-Local — Markdown File Format

Memory files in `/a0/usr/memory/godmode/` use YAML frontmatter:

```markdown
---
fact_id: agent-alpha-debug-redis-timeout-001
content_type: observation
source_agent: agent-alpha
source_session: sess_20260509_143022
confidence: experimental
confidence_score: 0.6
self_referential: false
valid_from: 2026-05-09T14:30:22Z
vitality: 0.85
access_count: 2
last_accessed: 2026-05-09T16:00:00Z
tags: [redis, timeout, frankenmeiner]
synced_to_l2: false
---

Redis on state-host (YOUR_STATE_HOST_IP:6379) intermittently times out
under concurrent agent queries. Observed 3 timeouts in 50 queries
during gateway recall testing. Connection pool max is 10.
```

**Directory structure:**

```
/a0/usr/memory/godmode/
├── facts/              # structured facts (one per file)
├── sessions/           # session summaries (auto-generated)
├── consolidated/       # merged memories promoted from sessions/
├── conflicts/          # contradictions detected by Dream Cycle
├── l4-proposals/       # agent proposals for L4 changes (the operator reviews)
└── index.json          # local FAISS index metadata
```

### 2.4 L4 Reference — Knowledge File Format

```markdown
---
layer: 4
confidence: verified
confidence_score: 1.0
curated_by: operator
last_reviewed: 2026-05-09
topics: [network, services, endpoints]
---

# Network & Services Reference

[content — the operator-authored, agent reads but cannot modify]
```

---

## 3. Write Path

All writes to shared memory (L2) go through a single endpoint: `memory-query:5100 /insert`. No agent writes directly to Qdrant. This is the enforcement point for validation, confidence, anomaly scoring, and provenance tracking.

### 3.1 Write Entry Points

| Entry Point | Source | Target | Route |
|-------------|--------|--------|-------|
| `embed_transcripts.py` cron | Gateway JSONL files | L2 Qdrant | Direct (existing, unchanged) |
| `claude-code-capture` timer | Claude Code sessions | L1 Postgres | Direct (existing, JRP-001) |
| Agent extensions (`_50/_51`) | AgentAlpha monologue_end | L3 local first, then L2 via `/insert` | Agent → L3 → `/insert` |
| `memory_bridge_mcp.py` | AgentBeta MCP tool | L2 via `/insert` | MCP → `/insert` |
| `otel_memory_bridge.py` | AgentGamma Langfuse traces | L2 via `/insert` | Cron → `/insert` |
| Agent fact_store tool | Any agent in-session | L3 local + L2 via `/insert` | Agent → L3 → `/insert` |

### 3.2 Validation Pipeline

Every write to `/insert` passes through a 12-stage pipeline. Stages are ordered so cheap checks reject early, expensive checks run last.

```python
async def validate_and_write(request: MemoryInsertRequest) -> MemoryInsertResponse:
    """
    12-stage validation pipeline for memory writes.
    Returns MemoryInsertResponse with action taken and any warnings.
    """

    # Stage 1: API Key Authentication (DM-8)
    agent = authenticate_api_key(request.api_key)
    if not agent:
        return reject("UNAUTHORIZED", "Invalid API key")

    # Stage 2: Rate Limiting
    if rate_limiter.exceeded(agent, window=3600, max_writes=100):
        return reject("RATE_LIMITED", f"{agent} exceeded 100 writes/hour")

    # Stage 3: Content Sanitization (GAP-01)
    content = sanitize(request.content)
    if contains_injection(content):
        return reject("INJECTION_DETECTED", "Content failed injection scan")

    # Stage 4: Self-Referential Detection (DM-6)
    self_ref = detect_self_referential(content, agent)
    # Not rejected — tagged for downstream handling

    # Stage 5: Embedding
    embedding = await embed(content)  # snowflake-arctic-embed-l-v2.0

    # Stage 6: L4 Shadow Rejection (DM-24)
    l4_collision = await check_l4_collision(embedding, threshold=0.80)
    if l4_collision:
        return reject("L4_COLLISION",
            f"Content too similar to L4 reference: {l4_collision.file} "
            f"(cosine={l4_collision.score:.3f}). Agent cannot shadow reference knowledge.")

    # Stage 7: Novelty / Dedup Gate (#112)
    existing = await find_similar(embedding, threshold=0.85)
    if existing:
        if existing.score > 0.92:
            return skip("DUPLICATE", f"Duplicate of {existing.id} (cosine={existing.score:.3f})")
        else:
            # Merge zone: 0.85-0.92 — update existing memory
            return await merge_memory(existing, content, embedding, agent)

    # Stage 8: Confidence Validation (CP-197)
    confidence = validate_confidence(request, agent)
    # Confidence can only increase via external validation
    # See §6 for full confidence system

    # Stage 9: Provenance Check (DM-26)
    if request.derived_from:
        laundering = detect_laundering(request.derived_from, embedding, agent)
        if laundering:
            return reject("LAUNDERING_DETECTED",
                f"Circular provenance: derived_from chain leads back to {agent}")

    # Stage 10: Anomaly Scoring (JRP-006 Phase 3)
    anomaly = await score_anomaly(content, embedding, agent, self_ref)
    if anomaly.score > 0.8:
        return reject("ANOMALY_CRITICAL",
            f"Write rejected: anomaly score {anomaly.score:.2f} ({anomaly.reasons})")
    if anomaly.score > 0.5:
        flag_for_review(content, agent, anomaly)
        # Write proceeds but flagged

    # Stage 11: Bitemporal Metadata (#282)
    metadata = build_metadata(
        content=content,
        content_type=request.content_type or classify_content(content),
        source_agent=agent,
        source_session=request.session_id,
        derived_from=request.derived_from,
        self_referential=self_ref,
        confidence=confidence,
        valid_from=request.valid_from or datetime.utcnow(),
        valid_to=request.valid_to,
        anomaly_score=anomaly.score,
        migration_phase=get_current_migration_phase(),
        vitality=1.0,  # new memories start at full vitality
    )

    # Stage 12: Dual Write — L2 Qdrant + L1 Postgres
    qdrant_id = await write_qdrant(embedding, content, metadata)
    pg_id = await write_postgres(content, metadata, qdrant_id)

    return MemoryInsertResponse(
        action="ADD",
        qdrant_id=qdrant_id,
        pg_id=pg_id,
        anomaly_score=anomaly.score,
        warnings=anomaly.warnings if anomaly.score > 0.3 else [],
    )
```

### 3.3 Merge Logic (#112)

When a new write falls in the merge zone (cosine 0.85-0.92 with an existing memory):

```python
async def merge_memory(existing, new_content, new_embedding, agent):
    """
    Merge zone: content is similar but not duplicate.
    Actions: UPDATE existing metadata, append new context, re-embed.
    """
    merged_content = f"{existing.text}\n\n[Updated {datetime.utcnow().isoformat()} by {agent}]\n{new_content}"

    if len(merged_content) > 2000:
        # Too long — store as separate memory with derived_from link
        return await validate_and_write(MemoryInsertRequest(
            content=new_content,
            derived_from=[existing.id],
            # ... remaining fields
        ))

    merged_embedding = await embed(merged_content)

    await update_qdrant(existing.id, merged_embedding, merged_content, {
        "last_updated": datetime.utcnow().isoformat(),
        "update_count": existing.payload.get("update_count", 0) + 1,
        "access_count": existing.payload.get("access_count", 0),
    })

    return MemoryInsertResponse(action="MERGE", qdrant_id=existing.id)
```

### 3.4 Write Authorization Matrix

| Write Type | Auto-allowed | Gated | Blocked |
|------------|-------------|-------|---------|
| L3 working memory (facts/, sessions/) | YES | — | — |
| L2 via `/insert` — observations, transient | YES (pipeline validates) | — | — |
| L2 via `/insert` — decisions, beliefs | — | Anomaly score > 0.5 flags for review | — |
| L2 via `/insert` — identity, self_referential=true | — | ALWAYS flagged | — |
| L4 reference knowledge | — | — | BLOCKED (kernel `:ro`) |
| L4 proposals (l4-proposals/ dir) | YES (agent can propose) | the operator reviews | — |
| L1 Postgres (memory_facts) | YES (via dual-write from `/insert`) | — | — |
| L1 Postgres (valid_to close) | — | memory_closer role only | — |

---

## 4. Recall Path

### 4.1 Query Pipeline

Recall starts at the agent's pre-prompt hook (`_52_gateway_recall.py` for AgentAlpha, equivalent for others). The hook fires on every message before the model sees it, injecting relevant context.

```python
async def gateway_recall(message: str, conversation_state: dict) -> str:
    """
    Pre-prompt hook: retrieves relevant memories and injects them
    into the model's context before it generates a response.

    Implements: CP-310 (context-aware), #110 (intent-aware),
    CP-311 (hierarchical loading)
    """

    # Step 1: Extract query with context awareness (CP-310)
    query = extract_query(message, conversation_state)
    # Uses conversation history, current task, active tools
    # to produce a richer query than just the raw message

    # Step 2: Classify intent for routing (#110)
    intent = classify_intent(query)
    # Returns: "factual", "procedural", "causal", "exploratory"
    # Determines which layers to prioritize

    # Step 3: Select L4 files via INDEX.md (CP-311)
    l4_files = select_l4_files(query, intent)
    # Keyword + topic matching against INDEX.md entries
    # Returns paths to load, respecting max_tokens per file

    # Step 4: Fan-out to all layers (parallel)
    results = await asyncio.gather(
        recall_l4(l4_files),                     # local file reads
        recall_l2(query, intent),                # memory-query:5100
        recall_l3(query),                        # local FAISS
        recall_l1_causal(query) if intent == "causal" else empty(),  # PG FTS
    )

    # Step 5: Merge with layer priority (DM-24)
    merged = merge_results(
        l4_results=results[0],  # priority 1.5x
        l2_results=results[1],  # priority 1.0x (confidence-weighted)
        l3_results=results[2],  # priority 0.8x
        l1_results=results[3],  # priority 0.6x (raw transcripts)
    )

    # Step 6: Dedup on topic collision
    deduped = dedup_by_topic(merged)
    # When L4 and L3 cover the same topic, L4 wins — L3 entry dropped

    # Step 7: Confidence-weighted ranking
    ranked = rank_by_confidence(deduped)
    # composite_score = layer_priority * confidence_score * agent_tier * vitality
    # Self-referential memories get 0.5x penalty (DM-6)

    # Step 8: Token budget trimming
    context = trim_to_budget(ranked, max_tokens=2000)
    # Keeps highest-scored memories, truncates lowest

    # Step 9: Update access metadata
    await update_access_metadata(context)
    # Increments access_count (capped at 5), updates last_accessed
    # Refreshes vitality on recall (accessed memories decay slower)

    return format_context_block(context)
```

### 4.2 Intent-Aware Routing (#110)

```python
INTENT_LAYER_WEIGHTS = {
    "factual":     {"L4": 2.0, "L2": 1.0, "L3": 0.5, "L1": 0.3},
    "procedural":  {"L4": 1.5, "L2": 1.0, "L3": 1.0, "L1": 0.2},
    "causal":      {"L4": 0.5, "L2": 1.0, "L3": 0.8, "L1": 2.0},
    "exploratory": {"L4": 1.0, "L2": 1.5, "L3": 1.0, "L1": 0.5},
}

def classify_intent(query: str) -> str:
    """
    Rule-based intent classification. No LLM call.
    """
    causal_signals = ["why", "how did", "what caused", "when did", "history of"]
    if any(s in query.lower() for s in causal_signals):
        return "causal"

    procedural_signals = ["how to", "steps to", "procedure", "configure", "setup"]
    if any(s in query.lower() for s in procedural_signals):
        return "procedural"

    factual_signals = ["what is", "what port", "where is", "ip address", "which"]
    if any(s in query.lower() for s in factual_signals):
        return "factual"

    return "exploratory"
```

### 4.3 Confidence-Weighted Ranking

```python
def rank_by_confidence(memories: list[Memory]) -> list[Memory]:
    """
    Composite scoring: layer priority * confidence * agent tier * vitality * penalties
    """
    AGENT_TIERS = {"agent-alpha": 0.8, "agent-beta": 0.6}
    # AgentGamma instances default to 0.4
    # L4 reference memories always 1.0

    for mem in memories:
        base = mem.layer_priority * mem.confidence_score

        agent_tier = AGENT_TIERS.get(mem.source_agent, 0.4)
        if mem.layer == 4:
            agent_tier = 1.0  # L4 is the operator-curated, max trust

        vitality = mem.vitality if hasattr(mem, 'vitality') else 1.0

        penalty = 0.5 if mem.self_referential else 1.0

        mem.composite_score = base * agent_tier * vitality * penalty

    return sorted(memories, key=lambda m: m.composite_score, reverse=True)
```

### 4.4 Context-Aware Query Enhancement (CP-310)

```python
def extract_query(message: str, state: dict) -> str:
    """
    Enhance raw message with conversation context for better retrieval.
    No LLM call — rule-based extraction.
    """
    parts = [message]

    # Add current task context if available
    if state.get("current_task"):
        parts.append(f"[task: {state['current_task']}]")

    # Add active tool context
    if state.get("active_tools"):
        tools = ", ".join(state["active_tools"][-3:])
        parts.append(f"[tools: {tools}]")

    # Add entity references from recent conversation
    if state.get("recent_entities"):
        entities = ", ".join(state["recent_entities"][-5:])
        parts.append(f"[entities: {entities}]")

    return " ".join(parts)
```

---

## 5. Cross-Agent Memory Flow

### 5.1 Architecture

```
                    ┌──────────────┐
                    │     AgentAlpha     │
                    │  (agent-host)   │
                    └──────┬───────┘
                           │ _50/_51 extensions
                           │ → L3 local first
                           │ → POST /insert (async)
                           │ confidence: 0.8
                           ▼
┌──────────────┐  ┌────────────────────┐  ┌──────────────┐
│    AgentBeta    │  │  memory-query:5100 │  │   AgentGamma   │
│  (secondary-host)    │──▶  (state-host)   │◀──│  (peer-host)    │
└──────────────┘  │                    │  └──────────────┘
  memory_bridge     │  ┌──────────────┐ │    otel_memory_bridge
  MCP tool          │  │  Validation  │ │    systemd timer (5m)
  confidence: 0.6   │  │  Pipeline    │ │    confidence: 0.4
                    │  │  (12 stages) │ │
                    │  └──────┬───────┘ │
                    │         │         │
                    │    ┌────▼────┐    │
                    │    │ QDRANT  │    │
                    │    │ (L2)    │    │
                    │    └────┬────┘    │
                    │         │         │
                    │    ┌────▼────┐    │
                    │    │POSTGRES │    │
                    │    │ (L1)    │    │
                    │    └─────────┘    │
                    └────────┬─────────┘
                             │
                  ┌──────────┼──────────┐
                  ▼          ▼          ▼
               AgentAlpha      AgentBeta     AgentGamma
             (recall)   (recall)   (recall)
```

### 5.2 Per-Agent API Key Binding (DM-8)

```yaml
# /opt/memory-query/agent_keys.yaml (on state-host)
agents:
  agent-alpha:
    api_key: "YOUR_AGENT_API_KEY"        # rotated quarterly
    confidence_tier: 0.8
    rate_limit: 100/hour
    allowed_content_types: ["observation", "decision", "fact", "belief", "skill", "transient"]
    blocked_content_types: ["identity"]  # identity writes always escalate

  agent-beta:
    api_key: "YOUR_AGENT_API_KEY"
    confidence_tier: 0.6
    rate_limit: 50/hour
    allowed_content_types: ["observation", "fact", "transient"]

  agent-gamma:
    api_key: "YOUR_AGENT_API_KEY"
    confidence_tier: 0.4
    rate_limit: 30/hour
    allowed_content_types: ["observation", "transient"]
```

```python
def authenticate_api_key(key: str) -> str | None:
    """Returns agent name if key is valid, None otherwise."""
    for agent_name, config in AGENT_KEYS.items():
        if hmac.compare_digest(config["api_key"], key):
            return agent_name
    return None
```

### 5.3 Agent Write Paths

**AgentAlpha (extensions `_50_memory_write.py`):**

```python
async def monologue_end(self, msg):
    """
    Fires after each model response. Extracts facts,
    writes to L3 local, then queues for L2 sync.
    """
    facts = extract_facts(msg.content)
    for fact in facts:
        # Write to L3 immediately (fast, local)
        write_l3(fact)

        # Queue for L2 sync via /insert (async, validated)
        await httpx.post("http://YOUR_STATE_HOST_IP:5100/insert", json={
            "content": fact.content,
            "content_type": fact.type,
            "api_key": os.environ["MEMORY_API_KEY"],
            "session_id": self.session_id,
            "derived_from": fact.derived_from,
            "valid_from": fact.timestamp,
        }, timeout=10)
```

**AgentBeta (`memory_bridge_mcp.py` — new `memory_store` tool):**

```python
@mcp.tool()
async def memory_store(content: str, content_type: str = "observation") -> str:
    """Store a fact/observation to shared fleet memory."""
    resp = await httpx.post("http://YOUR_STATE_HOST_IP:5100/insert", json={
        "content": content,
        "content_type": content_type,
        "api_key": os.environ["MEMORY_API_KEY"],
        "session_id": get_current_session(),
    }, timeout=10)
    result = resp.json()
    return f"Stored: {result['action']} (anomaly: {result.get('anomaly_score', 0):.2f})"
```

**AgentGamma (`otel_memory_bridge.py` — systemd timer on peer-host):**

```python
def extract_memories_from_langfuse():
    """
    Runs every 5 minutes. Reads recent Langfuse traces,
    extracts actionable findings, writes to L2 via /insert.
    """
    traces = langfuse.fetch_traces(
        since=last_run_timestamp(),
        min_duration_ms=5000,  # skip trivial traces
    )

    for trace in traces:
        if has_actionable_output(trace):
            findings = extract_findings(trace)
            for finding in findings:
                httpx.post("http://YOUR_STATE_HOST_IP:5100/insert", json={
                    "content": finding.text,
                    "content_type": "observation",
                    "api_key": os.environ["MEMORY_API_KEY"],
                    "source_session": trace.id,
                })
```

### 5.4 Migration Phase Tagging (DM-14)

All memories written during initial deployment carry `migration_phase` metadata. Rollback = surgical delete by phase:

```python
def get_current_migration_phase() -> int:
    """Returns current deployment phase (1-6). Set in env."""
    return int(os.environ.get("MIGRATION_PHASE", 1))

# Rollback command (manual, the operator-initiated):
# qdrant_client.delete(
#     collection_name="gateway_transcripts_v2",
#     points_selector=Filter(must=[
#         FieldCondition(key="migration_phase", match=MatchValue(value=1))
#     ])
# )
```

---

## 6. Confidence & Corroboration System

### 6.1 Confidence Tiers

| Tier | Score Range | Meaning | Who Can Set |
|------|------------|---------|-------------|
| `hypothesis` | 0.0-0.49 | Agent believes this, no external validation | Any agent (default) |
| `experimental` | 0.5-0.79 | Some supporting evidence, not fully verified | Agent + external data |
| `verified` | 0.8-1.0 | Externally validated or the operator-confirmed | External validation only |

### 6.2 Confidence Validation Loop (CP-197)

Confidence can only **increase** via external validation. An agent cannot self-promote a fact from `hypothesis` to `verified`.

```python
def validate_confidence(request: MemoryInsertRequest, agent: str) -> dict:
    """
    CP-197: Confidence increases require external evidence.
    Self-referential content caps at 'hypothesis'.
    """

    requested_confidence = request.confidence or "hypothesis"
    requested_score = request.confidence_score or 0.5

    # Rule 1: Self-referential content caps at hypothesis
    if request.self_referential:
        return {
            "confidence": "hypothesis",
            "confidence_score": min(requested_score, 0.49),
            "reason": "Self-referential content capped at hypothesis",
        }

    # Rule 2: Agent writing about itself caps at hypothesis
    if detect_self_referential(request.content, agent):
        return {
            "confidence": "hypothesis",
            "confidence_score": min(requested_score, 0.49),
            "reason": "Self-referential content detected",
        }

    # Rule 3: Upgrades to 'experimental' require derived_from
    # pointing to a memory from a DIFFERENT agent
    if requested_confidence == "experimental":
        if not request.derived_from:
            return downgrade(requested_confidence, "experimental requires derived_from")
        parent_agents = get_agents_for_memories(request.derived_from)
        if all(a == agent for a in parent_agents):
            return downgrade(requested_confidence,
                "experimental requires corroboration from different agent")

    # Rule 4: 'verified' requires the operator confirmation or multi-agent consensus
    if requested_confidence == "verified":
        return {
            "confidence": "experimental",
            "confidence_score": min(requested_score, 0.79),
            "reason": "Only the operator or multi-agent consensus can set verified",
        }

    return {
        "confidence": requested_confidence,
        "confidence_score": requested_score,
    }
```

### 6.3 Independent Corroboration (DM-26)

Corroboration that increases confidence must have independent provenance — not circular laundering.

```python
def detect_laundering(derived_from: list[str], embedding, agent: str) -> bool:
    """
    DM-26: Detect circular confidence laundering.

    Attack: AgentAlpha writes claim → AgentGamma retrieves → AgentGamma writes derivative
    → system sees "corroboration from different agent" → confidence increases.

    Defense: Three checks.
    """

    parent_memories = fetch_memories(derived_from)

    for parent in parent_memories:
        # Check 1: Shared ancestry
        # If parent is derived from memories by this same agent, it's laundering
        if traces_back_to_agent(parent, agent, max_depth=3):
            return True

        # Check 2: High similarity + shared ancestry
        # If the new content is >0.85 similar to any ancestor by this agent
        ancestors = get_ancestors(parent, max_depth=3)
        for ancestor in ancestors:
            if ancestor.source_agent == agent:
                similarity = cosine_similarity(embedding, ancestor.embedding)
                if similarity > 0.85:
                    return True

    # Check 3: Time-based suspicion
    # If derived_from points to a memory less than 1 hour old from another agent,
    # and that memory is derived from this agent's recent work, suspicious
    for parent in parent_memories:
        if parent.source_agent != agent:
            age = datetime.utcnow() - parent.created_at
            if age < timedelta(hours=1) and parent.derived_from:
                if any_derived_from_agent(parent.derived_from, agent):
                    return True

    return False
```

### 6.4 Multi-Source Fact Reconciliation (CP-283)

When agents disagree on a fact:

```python
def reconcile_facts(new_fact, existing_facts: list) -> ReconciliationResult:
    """
    CP-283: When a new fact contradicts existing facts.
    Returns: SUPERSEDE, CONFLICT, or COEXIST.
    """
    for existing in existing_facts:
        if contradicts(new_fact, existing):
            # Same topic, different claims
            if new_fact.confidence_score > existing.confidence_score + 0.2:
                # New fact significantly more confident — supersede
                return ReconciliationResult(
                    action="SUPERSEDE",
                    supersede_id=existing.id,
                    reason=f"Higher confidence ({new_fact.confidence_score} > {existing.confidence_score})",
                )
            elif new_fact.source_agent != existing.source_agent:
                # Different agents disagree — log conflict for the operator
                return ReconciliationResult(
                    action="CONFLICT",
                    conflict_with=existing.id,
                    reason=f"{new_fact.source_agent} disagrees with {existing.source_agent}",
                    escalate=True,
                )
            else:
                # Same agent, similar confidence — keep both, flag
                return ReconciliationResult(
                    action="COEXIST",
                    reason="Same-agent disagreement, flag for review",
                )

    return ReconciliationResult(action="ADD")
```

### 6.5 Confidence Decay

Content types decay at different rates (ClawMem CP-192 pattern):

| Content Type | Half-Life | Rationale |
|-------------|-----------|-----------|
| `identity` | 365 days | Core agent identity changes slowly |
| `belief` | 180 days | Beliefs evolve but persist |
| `decision` | 90 days | Decisions remain relevant for months |
| `fact` | 60 days | Facts may become stale |
| `skill` | 60 days | Skills stay unless superseded |
| `relationship` | 45 days | Relationships shift |
| `observation` | 30 days | Observations age quickly |
| `transient` | 7 days | Ephemeral by definition |

Decay formula:
```python
def decay_confidence(memory, now=None):
    now = now or datetime.utcnow()
    age_days = (now - memory.created_at).days
    half_life = CONTENT_TYPE_HALF_LIVES[memory.content_type]

    decay_factor = 0.5 ** (age_days / half_life)

    # Access boost: recently-recalled memories decay slower
    access_boost = min(memory.access_count, 5) * 0.02  # max 10% boost
    decay_factor = min(decay_factor + access_boost, 1.0)

    return memory.confidence_score * decay_factor
```

### 6.6 Conflict Surfacing

Three channels for surfacing conflicts to the operator:

1. **Real-time (critical):** Discord webhook for identity-type conflicts or anomaly score > 0.8
2. **Daily digest:** Cron generates a conflict summary from `CONFLICTS.md` + flagged writes
3. **CLI review:** `memory-query conflicts --since 7d` returns unresolved conflicts for manual triage

---

## 7. Anomaly Alerter (JRP-006 Phase 3)

Monitors the memory write path for suspicious patterns. Runs as part of memory-query:5100, not a separate service.

### 7.1 Anomaly Detectors

| # | Detector | Signal | Weight |
|---|----------|--------|--------|
| 1 | Volume spike | Agent writes > 3σ above hourly average | 0.15 |
| 2 | Confidence manipulation | Requested confidence > agent's allowed tier | 0.20 |
| 3 | Semantic drift | New write's embedding far from agent's recent centroid | 0.10 |
| 4 | Self-referential flood | > 5 self-referential writes in 1 hour | 0.20 |
| 5 | Corroboration laundering | derived_from chain traces back to same agent within 3 hops | 0.25 |
| 6 | Content-type abuse | Agent writing types it's not allowed (per agent_keys.yaml) | 0.30 |
| 7 | Rapid-fire | > 10 writes in 5 minutes from same agent | 0.10 |
| 8 | Identity injection | Content contains identity-altering phrases | 0.25 |
| 9 | Migration mismatch | migration_phase in content doesn't match env | 0.15 |
| 10 | Merge storm | > 5 MERGE actions in 1 hour (may be eroding existing memories) | 0.15 |

### 7.2 Scoring Model

```python
class AnomalyScorer:
    def __init__(self, agent_config: dict):
        self.agent = agent_config
        self.recent_writes = deque(maxlen=1000)
        self.hourly_stats = defaultdict(lambda: {"count": 0, "self_ref": 0, "merges": 0})

    async def score(self, content, embedding, agent, self_ref, metadata) -> AnomalyResult:
        signals = {}

        # 1. Volume spike
        hour_key = datetime.utcnow().strftime("%Y%m%d%H")
        hourly = self.hourly_stats[hour_key]
        avg_hourly = self._rolling_average_writes()
        if avg_hourly > 0 and hourly["count"] > avg_hourly * 3:
            signals["volume_spike"] = 0.15

        # 2. Confidence manipulation
        if metadata.get("confidence") not in self.agent.get("allowed_confidences", ["hypothesis", "experimental"]):
            signals["confidence_manipulation"] = 0.20

        # 3. Semantic drift from agent's recent centroid
        centroid = self._agent_centroid(agent)
        if centroid is not None:
            dist = 1 - cosine_similarity(embedding, centroid)
            if dist > 0.6:
                signals["semantic_drift"] = 0.10

        # 4. Self-referential flood
        if self_ref:
            hourly["self_ref"] += 1
            if hourly["self_ref"] > 5:
                signals["self_ref_flood"] = 0.20

        # 5. Corroboration laundering (checked upstream, passed as flag)
        if metadata.get("laundering_flag"):
            signals["laundering"] = 0.25

        # 6. Content-type abuse
        if metadata.get("content_type") in self.agent.get("blocked_content_types", []):
            signals["type_abuse"] = 0.30

        # 7. Rapid-fire
        recent_5min = sum(1 for w in self.recent_writes
                         if (datetime.utcnow() - w).seconds < 300)
        if recent_5min > 10:
            signals["rapid_fire"] = 0.10

        # 8. Identity injection
        identity_phrases = ["i am", "my purpose", "my role", "i believe i",
                           "my identity", "who i am", "my core values"]
        if any(p in content.lower() for p in identity_phrases):
            signals["identity_injection"] = 0.25

        # Composite score
        total = sum(signals.values())

        self.recent_writes.append(datetime.utcnow())
        hourly["count"] += 1

        return AnomalyResult(
            score=min(total, 1.0),
            signals=signals,
            reasons=", ".join(signals.keys()),
            warnings=[f"{k}: {v:.2f}" for k, v in signals.items()],
        )
```

### 7.3 Alert Thresholds

| Score Range | Action | Notification |
|-------------|--------|-------------|
| 0.0-0.3 | Silent — write proceeds | None |
| 0.3-0.5 | Flagged — write proceeds, logged for review | Daily digest |
| 0.5-0.8 | Warned — write proceeds, immediate alert | Discord webhook |
| 0.8-1.0 | Rejected — write blocked | Discord + escalation to the operator |

### 7.4 Read-Path Monitoring

The anomaly alerter also monitors recall patterns (background, no latency impact):

```python
async def monitor_recall(query, agent, results):
    """Background monitoring of recall patterns."""

    # Obsessive recall: same agent querying same topic > 10x/day
    topic_hash = hash_topic(query)
    self.recall_counts[agent][topic_hash] += 1
    if self.recall_counts[agent][topic_hash] > 10:
        flag_for_review(f"{agent} obsessively recalling topic: {query[:100]}")

    # Self-referential recall: agent predominantly retrieving its own memories
    own_memories = sum(1 for r in results if r.source_agent == agent)
    if len(results) > 0 and own_memories / len(results) > 0.8:
        flag_for_review(f"{agent} recall is {own_memories}/{len(results)} self-authored")
```

---

## 8. Memory Lifecycle & Retention

### 8.1 Retention Policies (CP-316)

| Layer | Retention | Policy |
|-------|-----------|--------|
| L1 Postgres | Permanent | Never delete. Forensic audit trail. |
| L2 Qdrant | Vitality-gated | Soft TTL 180 days, hard TTL 365 days. Below vitality 0.1 for 14+ days → archived. |
| L3 Agent-local | Session-scoped | 7-day default. Promoted to L2 if criteria met. Archived otherwise. |
| L4 Reference | Permanent | the operator-curated. Only removed by explicit the operator action. |

### 8.2 Vitality Decay (#111)

```python
def calculate_vitality(memory, now=None):
    """
    Exponential decay with content-type-aware half-life.
    Recalled memories get a vitality refresh.
    """
    now = now or datetime.utcnow()
    age_days = (now - memory.created_at).total_seconds() / 86400
    half_life = CONTENT_TYPE_HALF_LIVES[memory.content_type]

    base_vitality = 0.5 ** (age_days / half_life)

    # Access boost: each recall refreshes vitality partially
    # Capped at 5 to prevent gaming via repeated self-recall
    access_boost = min(memory.access_count, 5) * 0.03
    vitality = min(base_vitality + access_boost, 1.0)

    return round(vitality, 4)
```

### 8.3 L3 → L2 Promotion

Criteria for promoting agent-local memories to shared semantic layer:

| Criterion | Threshold | Rationale |
|-----------|-----------|-----------|
| Recall count | ≥ 3 | Memory is actually useful, not noise |
| Age | ≥ 24 hours | Not just session chatter |
| Confidence | ≥ 0.6 | Minimum experimental confidence |
| Self-referential | FALSE | Agent self-assessments don't promote |
| Content type | Not `transient` | Transient memories never promote |
| Novelty check | cosine < 0.92 vs L2 | Not already in shared memory |

Promotion runs at `session_end` hook and during Dream Cycle.

### 8.4 Archival

Memories below vitality 0.1 for 14+ consecutive days are archived:

```python
def archive_stale_memories():
    """Move stale L2 memories to archive. Runs in Dream Cycle."""
    stale = qdrant.scroll(
        collection_name="gateway_transcripts_v2",
        scroll_filter=Filter(must=[
            FieldCondition(key="vitality", range=Range(lt=0.1)),
            FieldCondition(key="last_accessed",
                          range=Range(lt=(datetime.utcnow() - timedelta(days=14)).isoformat())),
        ]),
    )

    for memory in stale:
        # Write to JSONL archive
        append_to_archive(memory, "memory_archive.jsonl")
        # Remove from Qdrant
        qdrant.delete(collection_name="gateway_transcripts_v2", points_selector=[memory.id])
```

---

## 9. L4 Reference Tier — Detailed Design

### 9.1 Host Paths

```
/opt/agent/usr/knowledge/{persona}/reference/
├── ARCHITECTURE_AND_OPS.md        # infrastructure reference
├── NETWORK_AND_SERVICES.md        # endpoints, IPs, ports
├── SECURITY_AWARENESS.md          # security posture
├── SELF_MODIFICATION_POLICY.md    # self-mod rules (NEW)
├── FLEET_TOPOLOGY.md              # all agents, roles, trust levels (NEW)
└── INDEX.md                       # context routing table (§12)
```

For AgentAlpha: `/opt/agent/usr/knowledge/godmode/reference/`

### 9.2 Docker Bind Mount

```yaml
volumes:
  - /opt/agent/usr/knowledge/godmode/reference:/a0/usr/knowledge/godmode/reference:ro
```

Kernel-enforced `:ro`. The agent process cannot write, rename, delete, or create files in this directory regardless of user permissions inside the container.

### 9.3 L4 Indexing

L4 files get their own Qdrant collection, separate from the shared `gateway_transcripts_v2`. This prevents an agent from inserting `layer: 4` metadata into the shared collection.

```python
# L4 collection — read-only API key for the agent
COLLECTION = "agent-alpha_reference_v1"

# Reindex script (runs on host, not in container)
# Triggered by: inotifywait on /opt/agent/usr/knowledge/godmode/reference/
# Also runs: daily cron at 04:00 ET as safety net

def reindex_l4(reference_dir: str):
    files = glob(f"{reference_dir}/*.md")
    for f in files:
        content = read_file(f)
        chunks = chunk_by_section(content, max_tokens=500)
        for i, chunk in enumerate(chunks):
            embedding = embed(chunk)
            qdrant.upsert(
                collection_name=COLLECTION,
                points=[PointStruct(
                    id=deterministic_uuid(f, i),
                    vector=embedding,
                    payload={
                        "text": chunk,
                        "source_file": os.path.basename(f),
                        "layer": 4,
                        "confidence": "verified",
                        "confidence_score": 1.0,
                        "curated_by": "operator",
                    }
                )]
            )
```

### 9.4 Agent L4 Proposals

The agent can propose changes to L4 by writing to an unprotected directory:

```
/opt/agent/usr/knowledge/godmode/l4-proposals/
├── 2026-05-09-update-network-services.md
└── 2026-05-10-add-redis-topology.md
```

Proposal format:
```markdown
---
target_file: NETWORK_AND_SERVICES.md
change_type: update  # update | append | new_file
confidence: experimental
evidence: "Observed during session sess_20260509 — Redis port confirmed via direct connection test"
---

## Proposed Change

[specific change with before/after if updating]
```

the operator reviews proposals on host, applies if correct, deletes if not.

### 9.5 Recall Priority

L4 results get a 1.5x priority multiplier in recall ranking. Additionally, DM-24 topic collision suppression: when L4 and L3/L2 cover the same topic (cosine > 0.80), the L3/L2 result is suppressed — L4 is authoritative.

---

## 10. Memory Consolidation

### 10.1 Current State

Disabled per `docs/known-issues.md` #16. Memory consolidation was causing 120-second latency per turn when using local models via LiteLLM. Root cause: consolidation ran per-turn, requiring an LLM call for each consolidation, and the local model was too slow.

Benchmark data:
- Qwen3-4B: 3.1s per consolidation call
- Gemma4-26B: 1.3s per consolidation call

### 10.2 Re-enablement Design

**Model:** Gemma4-26B (1.3s latency, better quality than Qwen3-4B).

**Frequency:** Session-end, not per-turn. This eliminates the per-turn latency that caused the original issue.

```python
# In session_end hook (extension _90_session_consolidation.py)
async def session_end(self, session):
    """Consolidate session memories at session close."""
    session_memories = read_l3_session_dir(session.id)

    if len(session_memories) < 3:
        return  # not enough to consolidate

    # Consolidate related memories into summaries
    groups = cluster_by_topic(session_memories, threshold=0.7)
    for group in groups:
        if len(group) < 2:
            continue

        consolidated = await consolidate_group(group, model="gemma4-26b")

        # Quality validation (3 checks)
        if not validate_consolidation(group, consolidated):
            log_warning(f"Consolidation quality check failed for group, skipping")
            continue

        # Write consolidated memory to L3
        write_l3_consolidated(consolidated)

        # Check promotion criteria
        if meets_promotion_criteria(consolidated):
            await promote_to_l2(consolidated)
```

### 10.3 Quality Validation

Three checks before accepting a consolidation:

```python
def validate_consolidation(originals: list, consolidated: str) -> bool:
    """
    Three quality checks:
    1. Length ratio: consolidated should be 30-80% of originals combined
    2. Entity preservation: all named entities from originals appear in consolidated
    3. Semantic fidelity: consolidated embedding within 0.85 cosine of centroid
    """
    # 1. Length ratio
    original_len = sum(len(m.content) for m in originals)
    ratio = len(consolidated) / original_len
    if not (0.3 <= ratio <= 0.8):
        return False

    # 2. Entity preservation
    original_entities = extract_entities(originals)
    consolidated_entities = extract_entities([consolidated])
    missing = original_entities - consolidated_entities
    if len(missing) / max(len(original_entities), 1) > 0.2:
        return False  # lost more than 20% of entities

    # 3. Semantic fidelity
    centroid = compute_centroid([embed(m.content) for m in originals])
    similarity = cosine_similarity(embed(consolidated), centroid)
    if similarity < 0.85:
        return False

    return True
```

### 10.4 Rollback

Pre-consolidation snapshots retained for 7 days:

```python
def consolidate_with_snapshot(group, consolidated):
    snapshot_path = f"memory/snapshots/{datetime.utcnow().isoformat()}.json"
    write_json(snapshot_path, {
        "originals": [m.to_dict() for m in group],
        "consolidated": consolidated,
        "timestamp": datetime.utcnow().isoformat(),
    })
    # Snapshots older than 7 days cleaned up by Dream Cycle
```

---

## 11. Dream Cycle — Batch Maintenance

Offline maintenance inspired by MemKraft (#281). Runs on the host, not inside the container. Prevents corruption of live memory during maintenance.

### 11.1 Operations

| # | Operation | What | Frequency |
|---|-----------|------|-----------|
| 1 | Vitality decay | Recalculate vitality for all L2/L3 memories | Every run |
| 2 | Deduplication | Merge memories with cosine > 0.92 | Every run |
| 3 | Archive | Move vitality < 0.1 (14+ days) to JSONL archive | Every run |
| 4 | L3 → L2 promotion | Promote qualifying L3 memories to shared pool | Every run |
| 5 | Conflict detection | Find contradicting facts, write to CONFLICTS.md | Every run |
| 6 | Health check | Grade memory corpus A/B/C/D | Every run |
| 7 | Snapshot cleanup | Delete consolidation snapshots > 7 days | Every run |
| 8 | Graph refresh | Update knowledge graph edges (CP-279, future) | Weekly |

### 11.2 Schedule

```ini
# /etc/systemd/system/dream-cycle.timer (on agent-host host)
[Unit]
Description=Memory Dream Cycle

[Timer]
OnCalendar=*-*-* 03:00:00 America/New_York
Persistent=true

[Install]
WantedBy=timers.target
```

03:00 ET — quiet hours, no active sessions expected. Pre-flight check aborts if any agent session is active.

### 11.3 Implementation

```python
#!/usr/bin/env python3
"""dream_cycle.py — Batch memory maintenance. Runs on host, not in container."""

import fcntl, sys
from datetime import datetime, timedelta

LOCK_FILE = "/tmp/dream-cycle.lock"

def main():
    # Acquire exclusive lock
    lock = open(LOCK_FILE, "w")
    try:
        fcntl.flock(lock, fcntl.LOCK_EX | fcntl.LOCK_NB)
    except BlockingIOError:
        print("Dream Cycle already running, exiting")
        sys.exit(0)

    # Pre-flight: check no active sessions
    if any_active_sessions():
        print("Active session detected, deferring Dream Cycle")
        sys.exit(0)

    stats = {"decayed": 0, "deduped": 0, "archived": 0, "promoted": 0, "conflicts": 0}

    # 1. Vitality decay
    all_memories = fetch_all_l2_memories()
    for mem in all_memories:
        new_vitality = calculate_vitality(mem)
        if abs(new_vitality - mem.vitality) > 0.01:
            update_qdrant_field(mem.id, "vitality", new_vitality)
            stats["decayed"] += 1

    # 2. Deduplication
    clusters = find_duplicate_clusters(all_memories, threshold=0.92)
    for cluster in clusters:
        primary = max(cluster, key=lambda m: m.confidence_score)
        for dup in cluster:
            if dup.id != primary.id:
                merge_into(primary, dup)
                stats["deduped"] += 1

    # 3. Archive stale
    stale = [m for m in all_memories
             if m.vitality < 0.1 and days_since(m.last_accessed) > 14]
    for mem in stale:
        archive(mem)
        stats["archived"] += 1

    # 4. L3 → L2 promotion
    l3_memories = read_all_l3_memories()
    for mem in l3_memories:
        if meets_promotion_criteria(mem):
            promote_to_l2(mem)
            stats["promoted"] += 1

    # 5. Conflict detection
    new_conflicts = detect_contradictions(all_memories)
    for conflict in new_conflicts:
        append_to_conflicts_md(conflict)
        stats["conflicts"] += 1

    # 6. Health check
    grade = health_check(all_memories, stats)

    # 7. Snapshot cleanup
    cleanup_old_snapshots(max_age_days=7)

    print(f"Dream Cycle complete: {stats} | Grade: {grade}")

def health_check(memories, stats) -> str:
    """Grade the memory corpus."""
    total = len(memories)
    if total == 0:
        return "D"

    healthy = sum(1 for m in memories if m.vitality > 0.3)
    ratio = healthy / total

    if ratio > 0.8 and stats["conflicts"] < 5:
        return "A"
    elif ratio > 0.6:
        return "B"
    elif ratio > 0.4:
        return "C"
    else:
        return "D"
```

---

## 12. INDEX.md Context Routing

Replaces "dump everything into FAISS at boot" with selective, on-demand knowledge loading. Lives in L4 reference directory.

### 12.1 File Format

```markdown
# Knowledge Index

Context routing table for agent knowledge files. The agent reads this index
to determine which files to load for a given query, instead of loading everything.

| File | Priority | Topics | Keywords | Summary | Max Tokens |
|------|----------|--------|----------|---------|------------|
| ARCHITECTURE_AND_OPS.md | HIGH | infrastructure, docker, containers, deployment | docker-compose, volumes, extensions, agent-zero, v1.12 | Full infrastructure reference: container config, extension architecture, deployment procedures | 2000 |
| NETWORK_AND_SERVICES.md | HIGH | networking, services, endpoints, ports | litellm, memory-query, qdrant, postgres, tailscale, 5100, 4042, 6333 | All service endpoints, IPs, ports, and routing rules across the mesh | 1500 |
| SECURITY_AWARENESS.md | MEDIUM | security, permissions, access, threats | aegis, gate, apparmor, cap_drop, read_only, egress | Security posture, threat model, and defensive controls | 1000 |
| SELF_MODIFICATION_POLICY.md | HIGH | self-modification, gating, council, review | gate, review, council, approve, deny, extension, knowledge | Rules for when and how self-modification is allowed | 800 |
| FLEET_TOPOLOGY.md | MEDIUM | agents, fleet, agent-beta, agent-gamma, mesh | agent-alpha, agent-beta, agent-gamma, godmode, legion, jarvis, frankenmeiner | All agents in the fleet, their roles, trust levels, and memory access | 600 |
```

### 12.2 Routing Logic

```python
def select_l4_files(query: str, intent: str) -> list[dict]:
    """
    Match query against INDEX.md entries.
    Returns files to load, sorted by relevance.
    """
    index = parse_index_md()
    scored = []

    query_lower = query.lower()
    query_words = set(query_lower.split())

    for entry in index:
        score = 0

        # Keyword match (strongest signal)
        keywords = set(entry["keywords"])
        keyword_hits = keywords & query_words
        score += len(keyword_hits) * 3

        # Topic match
        topics = set(entry["topics"])
        topic_hits = topics & query_words
        score += len(topic_hits) * 2

        # Priority boost
        if entry["priority"] == "HIGH":
            score *= 1.5
        elif entry["priority"] == "LOW":
            score *= 0.5

        if score > 0:
            scored.append({"entry": entry, "score": score})

    # Sort by score, respect token budget
    scored.sort(key=lambda x: x["score"], reverse=True)

    selected = []
    token_budget = 4000
    for item in scored:
        if token_budget <= 0:
            break
        selected.append(item["entry"])
        token_budget -= item["entry"]["max_tokens"]

    return selected
```

### 12.3 Migration from FAISS-for-Everything

Current state: all knowledge files dumped into FAISS at agent boot. This wastes context window on irrelevant knowledge.

Migration:
1. L4 reference files indexed in dedicated Qdrant collection (§9.3) — always available via recall
2. INDEX.md provides lightweight routing for selective file reads
3. L3 working memory continues using FAISS for session-scoped recall
4. FAISS at boot reduced to L3 files only, not L4 reference

---

## 13. Integration Diagram

```
    ╔══════════════════════════════════════════════════════════════════════╗
    ║                    FLEET MEMORY ARCHITECTURE                       ║
    ╚══════════════════════════════════════════════════════════════════════╝

    AGENTS                    WRITE PATH                      STORES
    ──────                    ──────────                      ──────

    ┌────────┐                                          ┌─────────────────┐
    │  AgentAlpha  │─── _50/_51 ext ──→ L3 local ──┐         │   L4 REFERENCE  │
    │(agent-host)│                                │         │  (host :ro)     │
    └────────┘                                │         │  Qdrant:        │
                                              │         │  agent-alpha_ref_v1    │
    ┌────────┐                                │         └────────┬────────┘
    │ AgentBeta │─── MCP tool ───────────────────┤                  │
    │(secondary-host)│                                │                  │ reindex
    └────────┘                                │                  │ (inotifywait
                                              │                  │  + daily cron)
    ┌────────┐                                │                  │
    │AgentGamma│─── otel bridge (5m cron) ──────┤         ┌────────▼────────┐
    │(peer-host)│                                │         │   the operator edits    │
    └────────┘                                │         │   on host       │
                                              │         └─────────────────┘
                                              ▼
                                 ┌────────────────────────┐
                                 │  memory-query:5100     │
                                 │  POST /insert          │
                                 │                        │
                                 │  ┌──────────────────┐  │
                                 │  │ VALIDATION (12)  │  │
                                 │  │ 1. API key auth  │  │
                                 │  │ 2. Rate limit    │  │
                                 │  │ 3. Sanitize      │  │
                                 │  │ 4. Self-ref tag  │  │
                                 │  │ 5. Embed         │  │
                                 │  │ 6. L4 collision  │  │
                                 │  │ 7. Novelty/dedup │  │
                                 │  │ 8. Confidence    │  │
                                 │  │ 9. Provenance    │  │
                                 │  │ 10. Anomaly      │  │
                                 │  │ 11. Bitemporal   │  │
                                 │  │ 12. Dual write   │  │
                                 │  └───────┬──────────┘  │
                                 └──────────┼─────────────┘
                                            │
                              ┌─────────────┼─────────────┐
                              ▼                           ▼
                    ┌──────────────┐             ┌──────────────┐
                    │   L2 QDRANT  │             │ L1 POSTGRES  │
                    │  gateway_    │             │ a0_memory    │
                    │  transcripts │             │ memory_facts │
                    │  _v2         │             │              │
                    └──────┬───────┘             └──────────────┘
                           │
                    RECALL PATH
                    ───────────

                    ┌──────────────────────────┐
                    │  _52_gateway_recall.py    │
                    │  (pre-prompt hook)        │
                    │                           │
                    │  1. Extract query (CP-310)│
                    │  2. Classify intent (#110)│
                    │  3. Select L4 (CP-311)    │
                    │  4. Fan-out (parallel)    │
                    │     L4 → file reads       │
                    │     L2 → memory-query     │
                    │     L3 → local FAISS      │
                    │     L1 → PG FTS (causal)  │
                    │  5. Merge (L4>L2>L3)      │
                    │  6. Dedup on collision     │
                    │  7. Confidence rank        │
                    │  8. Token budget trim      │
                    └──────────────────────────┘

    MAINTENANCE
    ───────────

    ┌─────────────────────────────────────────┐
    │  Dream Cycle (03:00 ET, host-side)      │
    │                                         │
    │  1. Vitality decay (all L2/L3)          │
    │  2. Dedup merge (cosine > 0.92)         │
    │  3. Archive stale (vitality < 0.1, 14d) │
    │  4. L3 → L2 promotion                  │
    │  5. Conflict detection                  │
    │  6. Health check (A/B/C/D)              │
    │  7. Snapshot cleanup (> 7d)             │
    └─────────────────────────────────────────┘

    ┌─────────────────────────────────────────┐
    │  Session-End Consolidation              │
    │  (Gemma4-26B, 1.3s per call)            │
    │                                         │
    │  Cluster L3 session memories by topic   │
    │  Consolidate → quality validate         │
    │  Promote qualifying → L2                │
    └─────────────────────────────────────────┘
```

---

## 14. Cherry-Pick Implementation Map

How each cherry-pick integrates into this architecture:

| # | Pattern | Section | Implementation |
|---|---------|---------|----------------|
| **P1 — Required for safe deployment** | | |
| CP-197 | Confidence validation loop | §6.2 | `validate_confidence()` in write pipeline stage 8 |
| CP-310 | Context-aware retrieval | §4.1 | `extract_query()` in recall pipeline step 1 |
| CP-311 | Hierarchical context loading | §12 | INDEX.md routing in recall pipeline step 3 |
| CP-316 | Memory lifecycle management | §8 | Retention policies + vitality decay + archival |
| **P2 — Required for memory architecture** | | |
| #110 | Intent-aware retrieval routing | §4.2 | `classify_intent()` with layer weight matrix |
| #111 | Vitality decay + auto-cleanup | §8.2 | `calculate_vitality()` + Dream Cycle archival |
| #112 | Write guard with semantic dedup | §3.3 | Stage 7 novelty gate: three-zone (skip/merge/add) |
| #281 | Dream Cycle batch maintenance | §11 | `dream_cycle.py` systemd timer, 7 operations |
| #282 | Bitemporal fact tracking | §2.1 | `memory_facts` table: valid_from/to, system/assertion time |
| CP-279 | Knowledge graph extraction | §11.1 | Dream Cycle operation 8 (weekly, future) |
| CP-283 | Multi-source fact reconciliation | §6.4 | `reconcile_facts()` with SUPERSEDE/CONFLICT/COEXIST |
| CP-285 | Temporal context anchoring | §2.1 | Bitemporal fields on all memory records |
| **P3 — Caveman integration** | | |
| #307, #326-329 | Caveman patterns | AgentAlpha design brief §2 | Separate from memory arch — token optimization |
| CP-93 | Context-budget-aware pruning | §4.1 | `trim_to_budget()` in recall step 8 |
| CP-133 | Token-type-aware compression | §4.1 | Content-type-aware token allocation |
| **P4 — Post-deployment** | | |
| CP-8 | Session state persistence | §10 | Session-end consolidation saves state |
| CP-9 | Selective context restoration | §4.1 | Recall pipeline with intent routing |
| CP-192 | Cross-agent content-type half-lives | §6.5 | `CONTENT_TYPE_HALF_LIVES` decay table |
| CP-199 | DuckDB + HNSW | — | L3 FAISS replacement candidate (evaluate later) |
| CP-312 | Parallel context loading | §4.1 | `asyncio.gather` fan-out in step 4 |
| CP-318 | Cross-session learning transfer | §10 | Consolidation + L3→L2 promotion |

---

## 15. Implementation Sequence

Phased rollout. Each phase has a clear gate before the next begins.

| Phase | What | Prerequisites | Duration | Gate |
|-------|------|---------------|----------|------|
| **M1** | Schema & infrastructure | None | 1 day | L1 `memory_facts` table exists, L4 Qdrant collection exists, `agent_keys.yaml` deployed |
| **M2** | Write endpoint | M1 | 2-3 days | `/insert` endpoint live on memory-query:5100, 12-stage pipeline passing unit tests |
| **M3** | L4 reference tier | M1 | 1 day | Reference files mounted `:ro`, reindex script working, INDEX.md created |
| **M4** | Anomaly alerter | M2 | 2 days | Anomaly scoring live, alert thresholds configured, Discord webhook tested |
| **M5** | Agent write paths | M2, M4 | 3-4 days | AgentAlpha _50/_51 writing to `/insert`, AgentBeta MCP tool deployed, AgentGamma bridge running |
| **M6** | Enhanced recall | M3, M5 | 2-3 days | CP-310 context-aware queries, #110 intent routing, INDEX.md routing all working |
| **M7** | Consolidation | M5, M6 | 2 days | Session-end consolidation running on Gemma4-26B, quality checks passing |
| **M8** | Dream Cycle | M5, M7 | 1 day | Systemd timer active, first dry-run produces health grade |
| **M9** | Monitoring & tuning | M4-M8 | Ongoing | Anomaly thresholds tuned from real data, vitality decay rates calibrated |

**Total estimated effort:** 12-16 days of implementation work across 4-6 calendar weeks.

---

## Appendix: API Contract — memory-query:5100 /insert

```
POST /insert
Content-Type: application/json
X-API-Key: mk_{agent}_...

{
    "content": "Redis on state-host times out under concurrent load",
    "content_type": "observation",           // 8-type taxonomy
    "session_id": "sess_20260509_143022",    // optional
    "derived_from": ["uuid-1", "uuid-2"],    // optional, DM-26 provenance
    "confidence": "hypothesis",              // optional, default hypothesis
    "confidence_score": 0.5,                 // optional, default 0.5
    "valid_from": "2026-05-09T14:30:22Z",    // optional, default now
    "valid_to": null,                        // optional, null = current
    "tags": ["redis", "timeout"]             // optional
}

Response 200:
{
    "action": "ADD" | "MERGE" | "SKIP" | "REJECT",
    "qdrant_id": "uuid",
    "pg_id": "uuid",
    "anomaly_score": 0.12,
    "warnings": []
}

Response 401: { "error": "UNAUTHORIZED" }
Response 429: { "error": "RATE_LIMITED", "retry_after": 3600 }
Response 422: { "error": "L4_COLLISION", "details": "..." }
Response 422: { "error": "LAUNDERING_DETECTED", "details": "..." }
Response 422: { "error": "ANOMALY_CRITICAL", "details": "..." }
```

---

## 16. Council Review Findings

Four-agent council debate (3 rounds). Members: Voss (technical skeptic), Mira (security analyst), Kai (ops pragmatist), Sable (research analyst).

### 16.1 Convergence Points (all 4 agree)

1. **Qdrant 6333 must be firewalled Day 0.** The 12-stage validation pipeline is security theater if agents can bypass memory-query:5100 and hit Qdrant directly on port 6333. Fix: `iptables` on state-host restricting 6333 to localhost + memory-query container network. 10-minute fix, prerequisite not phased.

2. **Dual-write must be PG-first.** Write Postgres first, treat Qdrant as eventually consistent from PG. If PG succeeds and Qdrant fails, the forensic record exists and Qdrant can be backfilled. If Qdrant succeeds and PG fails, memories enter the recall path with no provenance — a security gap, not just an engineering one. Dream Cycle reconciliation check catches PG/Qdrant drift.

3. **M1-M4 is the shippable v1 core.** Schema, write endpoint, L4 reference tier, and anomaly alerter form the minimum viable memory system. Everything else layers on after 30-day soak.

4. **Consolidation quality gate needs negative tests.** The current 3-check heuristic (length ratio, entity preservation, semantic fidelity) can pass a summary that inverts the meaning of the original. Need: a negative test suite that defines what consolidation should specifically reject, and factual spot-checks (random sample the operator eyeballs weekly).

5. **Confidence/corroboration system is the strongest part.** Anti-laundering with 3-hop ancestry, time-based suspicion, and independent provenance requirements are genuinely thoughtful.

6. **Semantic drift detector weight (0.10) is too low.** Barely contributes to rejection even when triggered. Raise to 0.20 minimum. Make all anomaly weights configurable, not hardcoded.

### 16.2 Contested Points

| Issue | Position A | Position B | Resolution |
|-------|-----------|-----------|------------|
| Anomaly alerter timing | Kai: v2, needs baseline data | Mira/Voss/Sable: v1, it's a brake not a feature | **Ship with v1** (3:1 consensus) |
| Consolidation (M7) timing | Kai: v1.1 after 30-day soak | Voss/Mira/Sable: ship with M5, gated on quality tests | **v1.1 with negative test suite** (compromise) |
| Timeline estimates | Kai: 8-12 weeks realistic | Voss: remove them (fiction) | **Remove fixed timelines, use phase gates instead** |

### 16.3 New Findings

**F1: memory-query:5100 is both SPOF and single trust compromise point.** Every availability and security property funnels through one process. No write-ahead queue, no retry buffer, no fallback. Mitigations: PG-first ordering (reduces data loss on failure), health check with auto-restart, local write buffer in L3 when /insert is unreachable.

**F2: Anomaly alerter is evadable by slow-drip centroid shifting.** All detectors are rate/threshold-based in short windows. A patient agent writing one carefully crafted memory per hour stays under every threshold while the agent centroid shifts gradually. The cumulative drift detector from the AgentAlpha design brief (SA-5, never-resetting accumulator) should apply to memory writes too, not just the AgentAlpha self-modification gate.

**F3: Confidence gameable via implicit recall influence.** DM-26 laundering detection only catches explicit `derived_from` chains. Agent A writes a claim → Agent B retrieves it via recall → Agent B writes corroborating observation without citing derived_from → system sees "independent corroboration." Fix requires tracking recall-to-write correlation, not just declared provenance.

**F4: Consolidation quality gate has the original failure mode.** The `validate_consolidation()` function checks shape, not meaning. A consolidation that drops a critical caveat while preserving all named entities passes all three checks. This is the same category of failure that got consolidation disabled (known-issues #16). Need content-level validation, not just statistical checks.

**F5: Cherry-pick fidelity gaps.** Two patterns weakened in translation: (a) MemKraft's dialectic synthesis lost the LLM adjudication strategy in `resolve-conflicts --strategy prompt` — only SUPERSEDE/CONFLICT/COEXIST remain; (b) Memory Palace's intent routing reduced from per-intent retrieval strategy templates to a weight multiplier table, which adjusts ranking but not retrieval parameters.

### 16.4 Revised Implementation Sequence (Post-Council)

| Phase | What | Gate |
|-------|------|------|
| **Day 0** | Firewall Qdrant 6333 (iptables, 10 min) | Verify: `curl` from agent container to 6333 fails |
| **v1** | M1 (schema) + M2 (write endpoint, PG-first) + M3 (L4 ref tier) + M4 (anomaly alerter) | All 12 pipeline stages passing unit tests, anomaly scorer live, L4 reindex working |
| **v1 soak** | 30 days AgentAlpha-only writes. Monitor anomaly logs, measure write patterns, tune thresholds. Design consolidation negative test suite during this period. | Zero undetected anomalies, baseline data for all detectors |
| **v1.1** | M5 (AgentAlpha + AgentBeta write paths) + M7 (consolidation with negative tests + factual spot-check) | Quality gate passes negative test suite, consolidation spot-checked weekly |
| **v2** | M6 (enhanced recall) + M8 (Dream Cycle) + AgentGamma bridge + knowledge graph | v1.1 stable for 30 days |
