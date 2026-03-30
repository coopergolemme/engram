# Engram — MVP Plan

> Your AI forgets everything. Engram fixes that.

---

## Overview

Engram is a self-hosted long-term memory engine for AI agents. Rather than storing raw document chunks like a traditional RAG system, Engram extracts **atomic facts** from conversations and documents, tracks how they change over time, resolves conflicts, and retrieves the most relevant context on demand.

The MVP is a Python REST API that any agent or app can call with a few HTTP requests.

---

## Goals

- Store memories that persist across sessions
- Extract atomic facts from raw input using an LLM
- Detect and resolve conflicting or outdated facts automatically
- Retrieve relevant memories via hybrid semantic + keyword search
- Return a ready-to-inject context block for use in agent prompts

---

## Non-Goals (Post-MVP)

- Web UI or dashboard
- File / PDF ingestion
- URL fetching
- Memory decay / expiry scoring
- Multi-user / auth layer
- Connectors (Notion, Google Drive, etc.)

---

## Tech Stack

| Layer | Tool |
|---|---|
| API framework | FastAPI |
| Database | SQLite |
| Vector search | sqlite-vec |
| Embeddings | sentence-transformers (local) |
| Keyword search | rank-bm25 |
| Fact extraction | Claude API (claude-sonnet-4-20250514) |

No external infrastructure required. Runs entirely on localhost.

---

## API Endpoints

### `POST /memories`
Manually add a single memory.

**Request**
```json
{
  "content": "User prefers concise responses",
  "agent_id": "assistant-1",
  "tags": ["preference"]
}
```

**Response**
```json
{
  "id": "mem_abc123",
  "content": "User prefers concise responses",
  "created_at": "2026-03-30T12:00:00Z"
}
```

---

### `POST /ingest`
Send raw text. Engram extracts atomic facts and stores them as individual memories.

**Request**
```json
{
  "text": "I just moved from New York to Boston. I'm working on a CLI tool in Python and prefer minimal dependencies.",
  "agent_id": "assistant-1"
}
```

**Response**
```json
{
  "extracted": [
    "User moved from New York to Boston",
    "User is building a CLI tool in Python",
    "User prefers minimal dependencies"
  ],
  "stored": 3
}
```

---

### `GET /search?q=&agent_id=`
Hybrid semantic + keyword search over stored memories.

**Response**
```json
{
  "results": [
    {
      "id": "mem_abc123",
      "content": "User is building a CLI tool in Python",
      "score": 0.91
    }
  ]
}
```

---

### `GET /context?q=&agent_id=`
Returns top-N memories formatted as a prompt-ready context block.

**Response**
```json
{
  "context": "Here is what you know about the user:\n- User moved from New York to Boston\n- User is building a CLI tool in Python\n- User prefers minimal dependencies"
}
```

---

### `DELETE /memories/{id}`
Delete a specific memory by ID.

---

### `GET /memories?agent_id=`
List all memories for an agent.

---

## Project Structure

```
engram/
├── main.py                    # FastAPI app + router registration
├── routers/
│   ├── memories.py            # CRUD endpoints
│   ├── ingest.py              # /ingest endpoint
│   └── search.py              # /search and /context endpoints
├── services/
│   ├── extractor.py           # LLM-based fact extraction
│   ├── memory_engine.py       # conflict detection + deduplication
│   ├── embedder.py            # sentence-transformers wrapper
│   └── retriever.py           # hybrid BM25 + vector search
├── db/
│   ├── database.py            # SQLite connection + schema setup
│   └── models.py              # memory table schema
├── requirements.txt
└── README.md
```

---

## Data Model

```sql
CREATE TABLE memories (
  id          TEXT PRIMARY KEY,
  agent_id    TEXT NOT NULL,
  content     TEXT NOT NULL,
  embedding   BLOB,              -- stored as sqlite-vec vector
  tags        TEXT,              -- JSON array
  source      TEXT,              -- 'manual' | 'ingest'
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## Conflict Detection (MVP)

When `/ingest` stores a new fact, the engine:

1. Embeds the new fact
2. Searches for existing memories with similarity > 0.85
3. Sends both to the LLM with a prompt: *"Does the new fact supersede the old one?"*
4. If yes, soft-deletes the old memory and stores the new one

Example:
- Existing: `"User lives in New York"`
- Incoming: `"User moved to Boston"`
- Action: old memory marked as superseded, new one stored

---

## Build Roadmap

### Session 1 — Storage foundation
- FastAPI skeleton
- SQLite schema + database.py
- `POST /memories`, `GET /memories`, `DELETE /memories/{id}`

### Session 2 — Embeddings + semantic search
- Integrate sentence-transformers
- Store embeddings in sqlite-vec
- `GET /search` with cosine similarity

### Session 3 — Fact extraction
- `/ingest` endpoint
- LLM prompt to extract atomic facts from raw text
- Store each fact as a separate memory

### Session 4 — Hybrid search + context endpoint
- Add BM25 keyword search layer
- Merge and re-rank semantic + keyword results
- `GET /context` returning a formatted prompt block

### Session 5 — Conflict resolution
- Compare incoming facts against existing memories
- LLM-based supersession check
- Soft-delete stale memories on conflict

### Session 6 — Polish
- `agent_id` scoping across all endpoints
- Tag filtering on search
- `/health` endpoint
- Docker support

---

## Example Agent Integration

```python
import httpx

ENGRAM = "http://localhost:8000"

# After a conversation turn, ingest what the user shared
httpx.post(f"{ENGRAM}/ingest", json={
    "text": user_message,
    "agent_id": "my-agent"
})

# Before generating a response, fetch relevant context
res = httpx.get(f"{ENGRAM}/context", params={
    "q": user_message,
    "agent_id": "my-agent"
})

context_block = res.json()["context"]

# Inject into your prompt
prompt = f"{context_block}\n\nUser: {user_message}"
```

---

## Success Criteria for MVP

- [ ] Can store and retrieve memories via HTTP
- [ ] `/ingest` correctly extracts multiple atomic facts from a paragraph
- [ ] Semantic search returns relevant results for paraphrased queries
- [ ] Conflict detection correctly supersedes outdated facts
- [ ] `/context` output can be dropped directly into an LLM prompt
- [ ] All data persists across server restarts
