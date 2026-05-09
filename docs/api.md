# BorgOS API Reference

## Write Endpoint

### `POST /insert`

Write a memory to the shared fleet memory system. All writes pass through the 12-stage validation pipeline.

**Headers:**
```
Content-Type: application/json
X-API-Key: <per-agent API key>
```

**Request Body:**
```json
{
    "content": "Redis on state-host times out under concurrent load",
    "content_type": "observation",
    "session_id": "sess_20260509_143022",
    "derived_from": ["uuid-1", "uuid-2"],
    "confidence": "hypothesis",
    "confidence_score": 0.5,
    "valid_from": "2026-05-09T14:30:22Z",
    "valid_to": null,
    "tags": ["redis", "timeout"]
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `content` | string | YES | — | Memory content text |
| `content_type` | string | no | auto-classified | One of: `observation`, `decision`, `fact`, `belief`, `skill`, `identity`, `relationship`, `transient` |
| `session_id` | string | no | — | Source session identifier |
| `derived_from` | string[] | no | — | Parent memory IDs for provenance tracking |
| `confidence` | string | no | `hypothesis` | `hypothesis`, `experimental`, or `verified` |
| `confidence_score` | float | no | 0.5 | 0.0 - 1.0 |
| `valid_from` | ISO 8601 | no | now | When the fact became true |
| `valid_to` | ISO 8601 | no | null | When the fact stopped being true (null = current) |
| `tags` | string[] | no | — | Searchable tags |

**Response 200 (Success):**
```json
{
    "action": "ADD",
    "qdrant_id": "uuid",
    "pg_id": "uuid",
    "anomaly_score": 0.12,
    "warnings": []
}
```

| `action` | Meaning |
|----------|---------|
| `ADD` | New memory stored |
| `MERGE` | Merged with existing similar memory (cosine 0.85-0.92) |
| `SKIP` | Duplicate detected (cosine > 0.92), not stored |
| `REJECT` | Blocked by validation pipeline |

**Error Responses:**

| Code | Error | Meaning |
|------|-------|---------|
| 401 | `UNAUTHORIZED` | Invalid API key |
| 429 | `RATE_LIMITED` | Agent exceeded write rate limit |
| 422 | `L4_COLLISION` | Content too similar to read-only reference knowledge |
| 422 | `LAUNDERING_DETECTED` | Circular provenance chain detected |
| 422 | `ANOMALY_CRITICAL` | Anomaly score > 0.8, write blocked |

---

## Query Endpoint

### `GET /query`

Retrieve memories relevant to a query. Returns confidence-weighted results from L2 (semantic) and optionally L1 (provenance).

**Headers:**
```
Content-Type: application/json
X-API-Key: <per-agent API key>
```

**Request Body:**
```json
{
    "query": "What port does the memory service run on?",
    "limit": 10,
    "min_confidence": 0.3,
    "content_types": ["fact", "decision"],
    "source_agents": ["agent-alpha", "agent-beta"],
    "include_l1": false
}
```

**Response 200:**
```json
{
    "results": [
        {
            "id": "uuid",
            "content": "Memory service runs on port 5100",
            "content_type": "fact",
            "source_agent": "agent-alpha",
            "confidence": "verified",
            "confidence_score": 0.95,
            "composite_score": 0.87,
            "vitality": 0.92,
            "valid_from": "2026-05-01T00:00:00Z",
            "valid_to": null,
            "self_referential": false,
            "layer": 2
        }
    ],
    "query_intent": "factual",
    "layers_searched": ["L4", "L2", "L3"]
}
```

---

## Anomaly Detectors

The write pipeline includes 10 anomaly detectors. Each contributes a weighted score:

| # | Detector | Weight | Trigger |
|---|----------|--------|---------|
| 1 | Volume spike | 0.15 | Writes > 3σ above hourly average |
| 2 | Confidence manipulation | 0.20 | Requested confidence exceeds agent tier |
| 3 | Semantic drift | 0.10 | Embedding far from agent's recent centroid |
| 4 | Self-referential flood | 0.20 | > 5 self-referential writes in 1 hour |
| 5 | Corroboration laundering | 0.25 | derived_from traces back to same agent |
| 6 | Content-type abuse | 0.30 | Writing blocked content types |
| 7 | Rapid-fire | 0.10 | > 10 writes in 5 minutes |
| 8 | Identity injection | 0.25 | Identity-altering phrases detected |
| 9 | Migration mismatch | 0.15 | Phase tag doesn't match environment |
| 10 | Merge storm | 0.15 | > 5 merges in 1 hour |

**Score thresholds:**
- 0.0 - 0.3: Silent (write proceeds)
- 0.3 - 0.5: Flagged (write proceeds, logged)
- 0.5 - 0.8: Warned (write proceeds, immediate alert)
- 0.8 - 1.0: Rejected (write blocked)
