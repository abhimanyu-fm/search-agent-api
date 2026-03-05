# Search Agent API — Design Plan

## Problem Statement

When a user chats with an agent, the agent produces a stream of messages (user turns, assistant turns, tool calls/results). This API gives the agent a **search tool** it can invoke to query across those accumulated messages — enabling memory recall, context retrieval, and conversation history mining.

---

## Architecture Overview

```
User ──► Agent (Claude / LLM)
             │  streams messages
             ▼
     Message Ingestion API        ◄── webhook / SDK hook on each stream chunk/event
             │
             ▼
     Message Store (DB + Index)
             │
             ▼
     Search API  ◄────────────────── Agent calls this as a Tool during inference
```

---

## Core Concepts

| Concept | Description |
|---|---|
| **Session** | A single end-to-end user ↔ agent interaction |
| **Conversation** | A named thread grouping multiple sessions (optional) |
| **Message** | One discrete turn: user input, assistant response, tool call, or tool result |
| **Chunk** | A streaming fragment of a message; assembled server-side into a complete message |

---

## Data Model

### `conversations`
```
id            UUID  PK
title         TEXT
created_at    TIMESTAMPTZ
metadata      JSONB       -- arbitrary key-value (user_id, tags, etc.)
```

### `sessions`
```
id              UUID  PK
conversation_id UUID  FK → conversations.id
started_at      TIMESTAMPTZ
ended_at        TIMESTAMPTZ  (nullable, set when stream closes)
metadata        JSONB
```

### `messages`
```
id              UUID  PK
session_id      UUID  FK → sessions.id
conversation_id UUID  FK → conversations.id
sequence_no     INT          -- order within session
role            ENUM(user, assistant, tool_call, tool_result)
content         TEXT         -- full assembled text
tool_name       TEXT         -- populated when role = tool_call / tool_result
tool_call_id    TEXT         -- links tool_call ↔ tool_result pairs
created_at      TIMESTAMPTZ
metadata        JSONB        -- model, stop_reason, usage tokens, etc.
embedding       VECTOR(1536) -- optional; computed async for semantic search
search_vector   TSVECTOR     -- auto-generated; for full-text search
```

---

## API Endpoints

### 1. Session Management

```
POST   /sessions                     Create a new session (call at stream start)
PATCH  /sessions/{session_id}        Close session / update metadata (call at stream end)
GET    /sessions/{session_id}        Get session details
```

### 2. Message Ingestion

```
POST   /sessions/{session_id}/messages       Append a complete message
POST   /sessions/{session_id}/messages/batch Append multiple messages at once
```

**Message body:**
```json
{
  "role": "assistant",
  "content": "The capital of France is Paris.",
  "sequence_no": 3,
  "metadata": {
    "model": "claude-sonnet-4-6",
    "stop_reason": "end_turn",
    "input_tokens": 120,
    "output_tokens": 12
  }
}
```

SDK integration note: call this endpoint once the stream for a message completes (on `message_stop` event for Anthropic SDK), not per-chunk. Chunks are assembled client-side before ingestion.

### 3. Search API — the agent tool

```
POST /search
```

**Request:**
```json
{
  "query": "what did the user say about their deadline",
  "filters": {
    "conversation_id": "uuid-optional",
    "session_id": "uuid-optional",
    "role": ["user", "assistant"],
    "date_from": "2026-01-01T00:00:00Z",
    "date_to":   "2026-03-05T23:59:59Z",
    "tool_name": null
  },
  "search_mode": "hybrid",  // "keyword" | "semantic" | "hybrid"
  "limit": 10,
  "offset": 0
}
```

**Response:**
```json
{
  "results": [
    {
      "message_id": "uuid",
      "session_id": "uuid",
      "conversation_id": "uuid",
      "role": "user",
      "content": "I need this done by Friday EOD",
      "created_at": "2026-03-04T14:22:00Z",
      "score": 0.91,
      "highlights": ["need this done by **Friday EOD**"],
      "context": {
        "before": { "role": "assistant", "content": "What is your deadline?" },
        "after":  { "role": "assistant", "content": "Got it, Friday EOD." }
      }
    }
  ],
  "total": 1,
  "search_mode_used": "hybrid"
}
```

### 4. Convenience Read Endpoints

```
GET  /sessions/{session_id}/messages         List all messages in a session (ordered)
GET  /conversations/{id}/messages            List all messages across sessions in a conversation
GET  /messages/{message_id}                  Get a single message + surrounding context
```

---

## Search Implementation

### Full-Text Search (keyword mode)
- **PostgreSQL FTS** using `tsvector` / `tsquery` with `ts_rank_cd`
- Supports phrase matching, stemming, boolean operators
- Index: `GIN` on `search_vector`

### Semantic Search (semantic mode)
- Embeddings generated asynchronously after ingestion (worker queue)
- Model: `text-embedding-3-small` (OpenAI) or `voyage-3` (Anthropic partner)
- Storage: `pgvector` extension, `HNSW` index on `embedding` column
- Similarity: cosine distance

### Hybrid Search (default)
- Run keyword + semantic in parallel
- Combine scores with Reciprocal Rank Fusion (RRF):
  ```
  rrf_score = Σ  1 / (k + rank_i)   where k = 60
  ```
- Return top-N by combined score

---

## Agent Tool Definition (Anthropic SDK)

Register `search_messages` as a tool in the agent's tool list:

```python
search_tool = {
    "name": "search_messages",
    "description": (
        "Search through the conversation history to recall past messages, "
        "user statements, decisions, or facts. Use this when you need to "
        "reference something said earlier in the conversation or in prior sessions."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "Natural language search query"
            },
            "role": {
                "type": "array",
                "items": {"type": "string", "enum": ["user", "assistant", "tool_result"]},
                "description": "Filter by message role. Omit to search all roles."
            },
            "date_from": {
                "type": "string",
                "format": "date-time",
                "description": "ISO 8601 start date filter (optional)"
            },
            "date_to": {
                "type": "string",
                "format": "date-time",
                "description": "ISO 8601 end date filter (optional)"
            },
            "limit": {
                "type": "integer",
                "default": 5,
                "description": "Max results to return (1–20)"
            }
        },
        "required": ["query"]
    }
}
```

The agent runtime calls `POST /search` when the LLM emits a `tool_use` block for `search_messages`, injects the `tool_result` back into the conversation, and continues generation.

---

## Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Runtime | **Python / FastAPI** | Async-native, fast, OpenAPI auto-docs |
| Database | **PostgreSQL 16** | FTS + pgvector in one store |
| Vector index | **pgvector HNSW** | No extra infrastructure |
| Embeddings | Background worker (asyncio task queue or Celery) | Non-blocking ingestion |
| Auth | API key (`X-API-Key` header) | Simple; agent has one key |
| Deployment | Docker Compose (Postgres + API) | Easy local + cloud |

---

## Project Structure

```
search-agent-api/
├── PLAN.md
├── docker-compose.yml
├── pyproject.toml
├── alembic/                    # DB migrations
│   └── versions/
├── app/
│   ├── main.py                 # FastAPI app factory
│   ├── config.py               # Settings (env vars)
│   ├── auth.py                 # API key middleware
│   ├── models/
│   │   ├── db.py               # SQLAlchemy ORM models
│   │   └── schemas.py          # Pydantic request/response schemas
│   ├── routers/
│   │   ├── sessions.py         # POST /sessions, PATCH /sessions/{id}
│   │   ├── messages.py         # POST /sessions/{id}/messages
│   │   └── search.py           # POST /search
│   ├── services/
│   │   ├── search_service.py   # Orchestrates keyword + semantic search
│   │   ├── embed_service.py    # Async embedding generation
│   │   └── ingest_service.py   # Message assembly + storage
│   └── db.py                   # Async DB session factory
├── tests/
│   ├── test_ingest.py
│   └── test_search.py
└── scripts/
    └── seed.py                 # Load sample messages for dev
```

---

## Key Design Decisions

1. **Ingest complete messages, not raw chunks** — assembling chunks server-side would require stateful streaming proxies. Simpler: client SDK assembles then POSTs one message per turn.

2. **Embeddings are async** — ingestion is synchronous and fast; embedding generation runs in the background so the agent is never blocked waiting for a vector model.

3. **Hybrid search by default** — keyword search handles exact names/dates/jargon; semantic search handles paraphrase and intent. Neither alone is sufficient.

4. **Context window in results** — returning the message immediately before and after each hit lets the agent reconstruct meaning without fetching full sessions.

5. **One API key per agent** — messages are scoped to that key's namespace; multi-tenant isolation is handled at the DB query layer via a `tenant_id` derived from the API key.

---

## Implementation Phases

### Phase 1 — Ingest + Keyword Search
- FastAPI skeleton, Postgres, Alembic migrations
- `POST /sessions`, `POST /sessions/{id}/messages`
- `POST /search` with FTS only
- Agent tool definition + basic integration test

### Phase 2 — Semantic Search
- Embed service + background task
- pgvector setup, HNSW index
- Hybrid RRF ranking

### Phase 3 — Production Hardening
- Rate limiting, pagination cursors, soft-delete
- Streaming ingestion webhook (Server-Sent Events alternative)
- Observability (structured logs, Prometheus metrics)
- Docker Compose + deployment guide
