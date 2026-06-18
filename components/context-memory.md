# Context Memory

## Role in Pipeline

Persistent memory layer — a brain-like memory graph that stores cognitive memories (thoughts, beliefs, preferences, aversions, emotions, goals, experiences) and injects relevant context into LLM conversations via semantic similarity search and graph traversal.

**Pipeline Phase:** Phase 2 — Context Assembly (parallel retrieval)

---

## How It Works

The Context Memory system operates as a transparent LLM proxy with two paths:

### Request Path (Synchronous)

1. Client sends a request through the proxy
2. Query is embedded and matched against stored memories via cosine similarity
3. Graph expansion (BFS 1-2 hops) discovers related memories through edges
4. Multi-signal ranker scores candidates (recency, relevance, confidence)
5. Token budget selector greedily picks top memories within the limit
6. Memories are formatted (markdown/XML/plain) and injected into the system prompt
7. Enriched request is forwarded to the upstream LLM provider
8. Response is returned to the client

### Extraction Path (Asynchronous)

1. After responding, the conversation is queued for background processing
2. An extraction LLM analyzes the conversation (structured output, low temperature)
3. New memory nodes and edges are parsed from the extraction response
4. Duplicate detection (embedding similarity) prevents redundant memories
5. New memories are persisted to SQLite + NetworkX graph + vector store
6. The graph grows richer over time, improving future context injection

---

## Architecture Diagram

```
Client App  -->  Proxy (FastAPI)  -->  LLM Provider (OpenAI/Anthropic/any)
                     |                         |
                     | inject memories         | response
                     | from graph              |
                     v                         v
              Context Builder <---- Memory Graph (NetworkX + SQLite)
                                          ^
                                          | extract (async)
                                   Extraction Pipeline
                                          ^
                                          |
                                   Conversation logs
```

---

## Context Injection Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│              CONTEXT INJECTION PIPELINE                           │
│                                                                   │
│  ┌─────────────────┐    ┌──────────────────────┐                │
│  │ Similarity      │    │  Graph Expansion     │                │
│  │ Search          │───▶│  (BFS 1-2 hops)     │                │
│  │ (embeddings)    │    │  (related memories)  │                │
│  └─────────────────┘    └──────────┬───────────┘                │
│                                    │                             │
│                                    ▼                             │
│                         ┌──────────────────┐                    │
│                         │  Context Ranker   │                    │
│                         │  (multi-signal    │                    │
│                         │   scoring)        │                    │
│                         └────────┬─────────┘                    │
│                                  │                               │
│                                  ▼                               │
│                         ┌──────────────────┐                    │
│                         │  Token Budget     │                    │
│                         │  (greedy select   │                    │
│                         │   within limit)   │                    │
│                         └────────┬─────────┘                    │
│                                  │                               │
│                                  ▼                               │
│                         ┌──────────────────┐                    │
│                         │ Context Formatter │                    │
│                         │ (markdown/xml/    │                    │
│                         │  plain text)      │                    │
│                         └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Memory Types

| Type | Description |
|------|-------------|
| thought | General thoughts or observations |
| belief | Held beliefs or convictions |
| aversion | Things the person dislikes or avoids |
| preference | Things the person prefers or likes |
| emotion | Emotional states or feelings |
| goal | Desired outcomes or aspirations |
| experience | Past events or lived experiences |
| fact | Factual information about the person |
| habit | Recurring behavioral patterns |
| value | Core values or principles |
| relationship | Information about relationships |
| skill | Known skills or competencies |
| custom | User-defined types |

---

## Edge Types (Relationships Between Memories)

| Edge Type | Meaning |
|-----------|---------|
| contradicts | Memory A contradicts Memory B |
| supports | Memory A reinforces Memory B |
| caused_by | Memory A was caused by Memory B |
| leads_to | Memory A leads to Memory B |
| related_to | General association |
| evolved_from | Memory A is an evolution of Memory B |
| conflicts_with | Memory A conflicts with Memory B |
| depends_on | Memory A depends on Memory B |
| temporal_before | Memory A occurred before Memory B |
| temporal_after | Memory A occurred after Memory B |
| part_of | Memory A is part of Memory B |
| generalizes | Memory A is a generalization of Memory B |
| specifies | Memory A is a specialization of Memory B |

---

## Integration with Other Pipeline Components

| Component | Integration Point |
|-----------|------------------|
| **RAG Pipeline** | Both provide retrieval context; memories add personalization while RAG provides document knowledge |
| **GraphRAG** | Complementary graph traversal — GraphRAG covers document entities, Context Memory covers user cognitive state |
| **Context Manager** | Receives memory context and assembles it with other sources into the final token-budgeted prompt |
| **LLM Router** | Router selects the model tier; memory context is injected regardless of which model is chosen |
| **Observability** | Memory retrieval latency and hit rates are tracked as pipeline metrics |

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| API Framework | FastAPI + Uvicorn |
| Graph Storage | NetworkX (in-memory) + SQLite (persistence) |
| Embeddings | sentence-transformers |
| Vector Search | Cosine similarity (numpy) |
| Token Counting | tiktoken |
| HTTP Client | httpx (async) |
| Configuration | YAML (pyyaml) + Pydantic Settings |
| Logging | structlog |
| Rate Limiting | slowapi |

---

## Project Structure

```
ai-genai-context-memory/
├── config/                         # Environment-specific YAML configuration
│   ├── application.yaml            # Default settings
│   ├── application-dev.yaml        # Development overrides
│   ├── application-prod.yaml       # Production overrides
│   └── application-test.yaml       # Test overrides
├── src/memory_graph/               # Main application source
│   ├── main.py                     # FastAPI app factory + lifespan
│   ├── config/                     # Settings and logging
│   ├── models/                     # Pydantic data models and enums
│   ├── persistence/                # SQLite database layer
│   ├── graph/                      # NetworkX graph operations
│   ├── embeddings/                 # Vector similarity search
│   ├── extraction/                 # Async memory extraction pipeline
│   ├── context/                    # Context building for injection
│   ├── proxy/                      # LLM proxy layer + provider adapters
│   ├── api/                        # REST API endpoints
│   ├── security/                   # Rate limiting, validation, secrets
│   └── middleware/                 # Error handling, request logging
├── tests/                          # Unit and integration tests
├── scripts/                        # Dev server + seed data utilities
└── pyproject.toml                  # Project metadata + dependencies
```

---

## API Examples

### Proxy (transparent LLM forwarding with memory injection)

```bash
# OpenAI-compatible proxy
curl -X POST http://localhost:8000/v1/proxy/openai/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "What programming language should I use?"}]
  }'
```

### Memory CRUD

```bash
# Create a memory
curl -X POST http://localhost:8000/api/v1/memories/nodes \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "preference",
    "content": "User prefers dark mode in all applications",
    "summary": "Prefers dark mode",
    "confidence": 0.9,
    "tags": ["ui", "preferences"]
  }'

# Retrieve context for a query
curl -X POST http://localhost:8000/api/v1/context/retrieve \
  -H "Content-Type: application/json" \
  -d '{"query_text": "What UI theme should I recommend?"}'
```

---

## Key Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `server.port` | 8000 | HTTP port |
| `database.path` | `./data/memory.db` | SQLite file path |
| `embeddings.model_name` | `all-MiniLM-L6-v2` | Sentence transformer model |
| `extraction.enabled` | true | Toggle memory extraction |
| `extraction.provider` | openai | LLM for extraction |
| `context.max_token_budget` | 1024 | Max tokens for context injection |

---

## Source Repository

[ai-genai-context-memory](../ai-genai-context-memory)

---

## License

MIT
