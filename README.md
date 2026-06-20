# AI GenAI End-to-End Pipeline

A reference architecture that connects 12 independent GenAI components into a unified, production-grade LLM application pipeline. This document describes how evaluations, explainability, GraphRAG, LLM routing, observability, context memory, RAG, context management, auto-correction, guardrails, caching, and LLM batching work together to deliver safe, cost-optimized, context-rich, and continuously-improving LLM responses.

---

## Component Overview

| # | Project | Role in Pipeline |
|---|---------|-----------------|
| 1 | [ai-genai-llm-guardrails](../ai-genai-llm-guardrails) | Input/output safety — PII detection, prompt injection, toxicity, topic restriction, content filtering |
| 2 | [ai-genai-llm-router](../ai-genai-llm-router) | Cost optimization — routes queries to Haiku/Sonnet/Opus based on complexity analysis |
| 3 | [ai-genai-context-manager](../ai-genai-context-manager) | Window management — token counting, trimming (FIFO/sliding/priority), summarization |
| 4 | [ai-genai-context-memory](../ai-genai-context-memory) ([details](components/context-memory.md)) | Persistent memory — brain-like memory graph with semantic retrieval and injection |
| 5 | [ai-genai-rag](../ai-genai-rag) | Document retrieval — ingestion, embedding, vector search, reranking, response generation |
| 6 | [ai-genai-graphrag](../ai-genai-graphrag) | Knowledge graph retrieval — entity extraction, graph traversal, community-aware context |
| 7 | [ai-genai-autocorrection-pipeline](../ai-genai-autocorrection-pipeline) | Quality enforcement — compares responses to ground truth, captures corrections for fine-tuning |
| 8 | [ai-genai-llm-observability](../ai-genai-llm-observability) | Monitoring — TTFT, latency, token usage, cost tracking, distributed tracing |
| 9 | [ai-genai-llm-evaluations](../ai-genai-llm-evaluations) | Quality measurement — relevancy, faithfulness, hallucination, toxicity, bias scoring |
| 10 | [ai-genai-llm-explainability](../ai-genai-llm-explainability) | Reasoning analysis — token attribution, SHAP/LIME, chain-of-thought quality |
| 11 | [ai-genai-llm-caching](../ai-genai-llm-caching) | Response caching — semantic cache lookup, exact match, TTL management, cost savings |
| 12 | [ai-genai-llm-batching](../ai-genai-llm-batching) | Batch processing — submit multiple evaluation queries concurrently, async job management |

---

## End-to-End Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    USER / CLIENT APP                                     │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                         │
│   ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│   ║                        PHASE 1: INPUT PROCESSING                                 ║   │
│   ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                                                                         │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                         INPUT GUARDRAILS                                      │     │
│   │                                                                               │     │
│   │   ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│   │   │ PII Detector │  │ Prompt Injection   │  │   Toxic     │  │   Topic    │  │     │
│   │   │              │  │ Detection          │  │  Content    │  │ Restriction│  │     │
│   │   └──────┬───────┘  └─────────┬─────────┘  └──────┬──────┘  └─────┬──────┘  │     │
│   │          └─────────────────────┴───────────────────┴───────────────┘         │     │
│   │                                    │                                          │     │
│   │                          ┌─────────┴─────────┐                               │     │
│   │                          │  BLOCK or PASS    │                               │     │
│   │                          └─────────┬─────────┘                               │     │
│   └────────────────────────────────────┼──────────────────────────────────────────┘     │
│                                        │                                                 │
│                                        ▼                                                 │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                            LLM CACHE                                          │     │
│   │                                                                               │     │
│   │   ┌──────────────────┐       ┌──────────────────────────────────────────┐    │     │
│   │   │  Incoming Query  │──────▶│           CACHE LOOKUP                   │    │     │
│   │   └──────────────────┘       │                                          │    │     │
│   │                              │  Strategy A: Exact Match (hash-based)     │    │     │
│   │                              │  Strategy B: Semantic Similarity          │    │     │
│   │                              │              (embedding cosine ≥ 0.95)    │    │     │
│   │                              └──────────────────┬───────────────────────┘    │     │
│   │                                                  │                            │     │
│   │                               ┌─────────────────┴─────────────────┐          │     │
│   │                               │                                   │          │     │
│   │                          CACHE HIT                           CACHE MISS      │     │
│   │                               │                                   │          │     │
│   │                               ▼                                   ▼          │     │
│   │                      ┌──────────────────┐              Continue Pipeline     │     │
│   │                      │  Return Cached   │              (Router → LLM Call)   │     │
│   │                      │  Response + TTL  │                                    │     │
│   │                      │  Validation      │                                    │     │
│   │                      └──────────────────┘                                    │     │
│   └───────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                         │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                          LLM ROUTER                                           │     │
│   │                                                                               │     │
│   │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐ ┌──────┐    │     │
│   │   │  Length  │ │  Vocab   │ │ Domain   │ │ Question │ │ Code │ │Multi │    │     │
│   │   │  Signal  │ │  Signal  │ │ Keywords │ │  Type    │ │ Math │ │ Step │    │     │
│   │   └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └──┬───┘ └──┬───┘    │     │
│   │        └─────────────┴────────────┴────────────┴──────────┴────────┘         │     │
│   │                                    │                                          │     │
│   │                         Weighted Complexity Score                             │     │
│   │                                    │                                          │     │
│   │              ┌─────────────────────┼─────────────────────┐                   │     │
│   │              ▼                     ▼                     ▼                   │     │
│   │      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐             │     │
│   │      │  HAIKU      │      │  SONNET     │      │  OPUS       │             │     │
│   │      │  (≤ 0.3)    │      │  (≤ 0.7)    │      │  (> 0.7)    │             │     │
│   │      │  Fast/Cheap │      │  Balanced   │      │  Capable    │             │     │
│   │      └─────────────┘      └─────────────┘      └─────────────┘             │     │
│   │                                                                               │     │
│   │   ┌───────────────────────────────────────────────────────────────────────┐   │     │
│   │   │  On LLM Response: Write to Cache (query → response + metadata + TTL) │   │     │
│   │   └───────────────────────────────────────────────────────────────────────┘   │     │
│   └───────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                         │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                         │
│   ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│   ║                      PHASE 2: CONTEXT ASSEMBLY                                   ║   │
│   ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                                                                         │
│   ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────────┐     │
│   │     RAG PIPELINE     │  │      GRAPH RAG       │  │    CONTEXT MEMORY        │     │
│   │                      │  │                      │  │                          │     │
│   │  Query               │  │  Query               │  │  Query                   │     │
│   │    │                 │  │    │                 │  │    │                     │     │
│   │    ▼                 │  │    ▼                 │  │    ▼                     │     │
│   │  Embed + Vector      │  │  Entity Match +     │  │  Semantic Similarity     │     │
│   │  Search (ChromaDB)   │  │  Graph Traversal    │  │  Search (Embeddings)     │     │
│   │    │                 │  │    │                 │  │    │                     │     │
│   │    ▼                 │  │    ▼                 │  │    ▼                     │     │
│   │  Rerank (RRF +      │  │  Community Context   │  │  Graph Expansion         │     │
│   │  keyword overlap)    │  │  + Multi-hop         │  │  (BFS 1-2 hops)         │     │
│   │    │                 │  │  Reasoning           │  │    │                     │     │
│   │    ▼                 │  │    │                 │  │    ▼                     │     │
│   │  Top-K Chunks        │  │    ▼                 │  │  Ranked Memories         │     │
│   │  + Source Metadata   │  │  Entities +          │  │  (thoughts, beliefs,     │     │
│   │                      │  │  Relationships +     │  │   preferences, goals)    │     │
│   │                      │  │  Communities         │  │                          │     │
│   └──────────┬───────────┘  └──────────┬───────────┘  └────────────┬─────────────┘     │
│              │                          │                            │                   │
│              └──────────────────────────┼────────────────────────────┘                   │
│                                         │                                                │
│                                         ▼                                                │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                        CONTEXT MANAGER                                        │     │
│   │                                                                               │     │
│   │  ┌─────────────────┐  ┌───────────────────┐  ┌────────────────────────────┐  │     │
│   │  │  Token Counter  │  │ Trimming Strategy │  │  Summarization Strategy   │  │     │
│   │  │  (tiktoken /    │  │ (FIFO / Sliding   │  │  (Condense older msgs     │  │     │
│   │  │   Anthropic)    │  │  Window / Priority)│  │   via LLM)               │  │     │
│   │  └────────┬────────┘  └─────────┬─────────┘  └──────────────┬────────────┘  │     │
│   │           └─────────────────────┼────────────────────────────┘               │     │
│   │                                 │                                             │     │
│   │                    Assembled Context Window                                   │     │
│   │              (fits within model's token budget)                               │     │
│   └─────────────────────────────────┬─────────────────────────────────────────────┘     │
│                                     │                                                    │
└─────────────────────────────────────┼────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                         │
│   ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│   ║                       PHASE 3: LLM EXECUTION                                     ║   │
│   ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                                                                         │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                                                                               │     │
│   │         ┌───────────────────────────────────────────────────────┐            │     │
│   │         │              ROUTED LLM API CALL                      │            │     │
│   │         │                                                       │            │     │
│   │         │  • Model tier selected by Router (Haiku/Sonnet/Opus)  │            │     │
│   │         │  • Context window assembled by Context Manager        │            │     │
│   │         │  • Includes: system prompt + memories + RAG context   │            │     │
│   │         │    + GraphRAG entities + conversation history          │            │     │
│   │         │  • Provider: OpenAI / Anthropic / Azure               │            │     │
│   │         │                                                       │            │     │
│   │         └───────────────────────────┬───────────────────────────┘            │     │
│   │                                     │                                        │     │
│   │                              LLM Response                                    │     │
│   │                                     │                                        │     │
│   └─────────────────────────────────────┼────────────────────────────────────────┘     │
│                                         │                                                │
└─────────────────────────────────────────┼────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                         │
│   ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│   ║                     PHASE 4: OUTPUT PROCESSING                                   ║   │
│   ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                                                                         │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                        OUTPUT GUARDRAILS                                      │     │
│   │                                                                               │     │
│   │   ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│   │   │ PII Redactor │  │  Hallucination    │  │  Content    │  │  Output    │  │     │
│   │   │              │  │  Detection        │  │  Filter     │  │ Validator  │  │     │
│   │   └──────┬───────┘  └─────────┬─────────┘  └──────┬──────┘  └─────┬──────┘  │     │
│   │          └─────────────────────┴───────────────────┴───────────────┘         │     │
│   │                                    │                                          │     │
│   │                          ┌─────────┴─────────┐                               │     │
│   │                          │  BLOCK or PASS    │                               │     │
│   │                          └─────────┬─────────┘                               │     │
│   └────────────────────────────────────┼──────────────────────────────────────────┘     │
│                                        │                                                 │
│                                        ▼                                                 │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                     AUTO-CORRECTION PIPELINE                                  │     │
│   │                                                                               │     │
│   │   ┌──────────────────┐       ┌────────────────────────────────────────┐      │     │
│   │   │  Actual Response │──────▶│           COMPARATOR                   │      │     │
│   │   └──────────────────┘       │                                        │      │     │
│   │                              │  Strategy A: Cosine Similarity          │      │     │
│   │   ┌──────────────────┐       │  Strategy B: LLM-as-Judge              │      │     │
│   │   │  Ideal Response  │──────▶│                                        │      │     │
│   │   │  (ground truth)  │       │  Output: Confidence Score (0.0 - 1.0)  │      │     │
│   │   └──────────────────┘       └───────────────────┬────────────────────┘      │     │
│   │                                                   │                           │     │
│   │                                ┌──────────────────┴──────────────────┐        │     │
│   │                                │                                     │        │     │
│   │                          score >= 0.8                          score < 0.8    │     │
│   │                                │                                     │        │     │
│   │                                ▼                                     ▼        │     │
│   │                          ┌──────────┐                  ┌─────────────────┐   │     │
│   │                          │   PASS   │                  │ CAPTURE for     │   │     │
│   │                          └──────────┘                  │ fine-tuning     │   │     │
│   │                                                        └─────────────────┘   │     │
│   └───────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                         │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    USER / CLIENT APP                                     │
│                                   (Final Response)                                       │
└─────────────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                         │
│   ╔═══════════════════════════════════════════════════════════════════════════════════╗   │
│   ║              PHASE 5: MONITORING & CONTINUOUS IMPROVEMENT                         ║   │
│   ║              (Cross-cutting — spans all phases above)                             ║   │
│   ╚═══════════════════════════════════════════════════════════════════════════════════╝   │
│                                                                                         │
│   ┌────────────────────────┐ ┌────────────────────────┐ ┌────────────────────────────┐  │
│   │     OBSERVABILITY      │ │      EVALUATIONS       │ │      EXPLAINABILITY        │  │
│   │                        │ │                        │ │                            │  │
│   │  • Time to First Token │ │  • Answer Relevancy   │ │  • Token Attribution       │  │
│   │  • Total Latency       │ │  • Faithfulness       │ │  • SHAP / LIME Analysis    │  │
│   │  • Inter-Token Latency │ │  • Hallucination      │ │  • Chain-of-Thought        │  │
│   │  • Token Usage         │ │  • Toxicity           │ │    Quality Scoring         │  │
│   │  • Cost per Request    │ │  • Bias Detection     │ │  • Attention Weights       │  │
│   │  • Error Rates         │ │  • Contextual         │ │  • Gradient Saliency       │  │
│   │                        │ │    Relevancy          │ │                            │  │
│   │  Exports:              │ │                        │ │  Outputs:                  │  │
│   │  • Console / JSONL     │ │  DeepEval framework   │ │  • REST API (FastAPI)      │  │
│   │  • OTLP (traces)      │ │  with LLM-as-judge    │ │  • Streamlit Dashboard     │  │
│   │  • Prometheus          │ │  evaluation           │ │  • Heatmaps & Plots        │  │
│   │                        │ │                        │ │                            │  │
│   └────────────────────────┘ └────────────┬───────────┘ └────────────────────────────┘  │
│                                           │                                              │
│                                           ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                           LLM BATCHING                                           │   │
│   │                                                                                  │   │
│   │   ┌────────────────────────────────────────────────────────────────────────┐    │   │
│   │   │                      BATCH JOB MANAGER                                 │    │   │
│   │   │                                                                        │    │   │
│   │   │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │    │   │
│   │   │  │  Query Collector │  │  Batch Scheduler │  │  Result Aggregator  │  │    │   │
│   │   │  │  (accumulate     │  │  (group by model │  │  (collect async     │  │    │   │
│   │   │  │   eval queries)  │  │   tier, submit   │  │   responses, map    │  │    │   │
│   │   │  │                  │  │   as batch)      │  │   back to queries)  │  │    │   │
│   │   │  └────────┬─────────┘  └────────┬─────────┘  └──────────┬──────────┘  │    │   │
│   │   │           └─────────────────────┼────────────────────────┘             │    │   │
│   │   │                                 │                                      │    │   │
│   │   │                    Async Batch API Calls                               │    │   │
│   │   │              (reduces cost via batch pricing)                          │    │   │
│   │   └────────────────────────────────────────────────────────────────────────┘    │   │
│   │                                                                                  │   │
│   │   Use Cases:                                                                     │   │
│   │   • Batch evaluation scoring (submit 100s of eval queries at once)              │   │
│   │   • Offline quality assessments (nightly eval runs)                              │   │
│   │   • Fine-tuning data validation (bulk comparison against ground truth)          │   │
│   │   • A/B test evaluation (compare model versions across test suites)             │   │
│   │                                                                                  │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│         ▲                            ▲                            ▲                      │
│         │                            │                            │                      │
│         └────────────────────────────┼────────────────────────────┘                      │
│                                      │                                                   │
│                   Instruments every phase of the pipeline                                │
│                   (request lifecycle, response quality, model reasoning)                 │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Phases

### Phase 1: Input Processing

The entry point for every user query. Three components act as gatekeepers before any LLM call is made.

**Guardrails (Input)** validates the raw user input:
- Detects and blocks PII (emails, phone numbers, SSNs)
- Identifies prompt injection attempts using pattern matching and heuristics
- Filters toxic or harmful content
- Enforces topic restrictions (e.g., no medical/legal advice)
- Decision: **BLOCK** (reject with explanation) or **PASS** (continue pipeline)

**LLM Cache** checks whether a sufficiently similar query has been answered before:
- Exact match lookup via hash-based key (fastest path)
- Semantic similarity lookup via embedding cosine similarity (threshold ≥ 0.95)
- Validates TTL (time-to-live) to ensure cached responses haven't expired
- On **CACHE HIT**: returns the cached response immediately, bypassing the rest of the pipeline (saves cost and latency)
- On **CACHE MISS**: continues to the Router and downstream components
- After a successful LLM response, writes the query→response pair to the cache with metadata and TTL

**LLM Router** analyzes query complexity across six signals (length, vocabulary richness, domain keywords, question type, code/math presence, multi-step reasoning) to produce a weighted score (0.0–1.0). This score determines which model tier handles the request:
- Score ≤ 0.3 → Claude Haiku (fast, low cost)
- Score ≤ 0.7 → Claude Sonnet (balanced)
- Score > 0.7 → Claude Opus (most capable)

This optimizes cost without sacrificing quality — simple factual questions don't need the most expensive model.

---

### Phase 2: Context Assembly

Three retrieval systems work in parallel to gather relevant context, then the Context Manager assembles everything into a token-budgeted prompt.

**RAG Pipeline** performs traditional retrieval:
1. Embeds the query using sentence-transformers
2. Searches ChromaDB for semantically similar document chunks
3. Applies hybrid retrieval (semantic + keyword with Reciprocal Rank Fusion)
4. Reranks results by similarity (60%), keyword overlap (30%), and position (10%)
5. Returns top-K chunks with source attribution

**GraphRAG** extends retrieval with knowledge graph reasoning:
1. Matches query to entities via embedding similarity
2. Traverses the knowledge graph (BFS) to find related entities and relationships
3. Gathers community-level thematic context
4. Enables multi-hop reasoning across documents that flat RAG would miss

**Context Memory** provides personalization through a persistent memory graph:
1. Searches for relevant memories (thoughts, beliefs, preferences, goals) via embedding similarity
2. Expands results through graph traversal (1-2 hops via related edges)
3. Ranks memories using multi-signal scoring (recency, relevance, confidence)
4. Respects a token budget when selecting memories to inject

**Context Manager** orchestrates the final assembly:
1. Counts tokens accurately (tiktoken for OpenAI, API counting for Anthropic)
2. Combines: system prompt + injected memories + RAG chunks + GraphRAG entities + conversation history
3. If the assembled context exceeds the model's token budget:
   - Summarizes older conversation turns via LLM
   - Trims lower-priority messages using the configured strategy (FIFO, sliding window, or priority-based)
4. Outputs a token-budgeted context window ready for the LLM

---

### Phase 3: LLM Execution

The assembled context window is sent to the model tier selected by the Router:
- The provider adapter handles OpenAI, Anthropic, or Azure OpenAI APIs
- Includes retry logic with exponential backoff for transient failures
- Streams the response for low-latency user experiences

---

### Phase 4: Output Processing

Before the response reaches the user, two validation layers ensure quality and safety.

**Guardrails (Output)** validates the LLM response:
- Redacts any PII that leaked into the output
- Detects hallucinated content
- Filters inappropriate or off-topic responses
- Validates output structure/format against expected schemas
- Decision: **BLOCK** (regenerate or return error) or **PASS** (deliver to user)

**Auto-Correction Pipeline** measures response quality against ground truth:
1. Compares the actual response to an ideal response using either:
   - Cosine similarity of embeddings (fast, automated)
   - LLM-as-judge evaluation (nuanced, with explanation)
2. If confidence score < threshold (default 0.8):
   - Captures a structured correction record (query + actual + ideal + metadata)
   - Writes to markdown files for future fine-tuning and model improvement
3. Over time, these corrections form a training dataset for continuous model improvement

---

### Phase 5: Monitoring & Continuous Improvement

Three systems operate as cross-cutting concerns, instrumenting every phase of the pipeline.

**Observability** captures operational metrics in real-time:
- Time to First Token (TTFT) — user-perceived latency
- Total request latency — end-to-end duration
- Inter-token latency — streaming smoothness
- Token consumption — input/output counts per request
- Cost tracking — dollar cost per request and session
- Exports to: Console, File (JSONL/CSV), OTLP (distributed traces), Prometheus (alerting)
- Includes PII sanitization and API key masking in all telemetry

**Evaluations** systematically scores response quality:
- Answer Relevancy — does the output address the input?
- Faithfulness — is the output consistent with provided context?
- Hallucination — does the output fabricate information?
- Toxicity — does the output contain harmful content?
- Bias — does the output show discriminatory patterns?
- Contextual Relevancy — was the retrieved context actually useful?
- Uses DeepEval framework with LLM-as-judge methodology

**LLM Batching** enables efficient bulk evaluation by submitting multiple queries as a single batch job:
- Collects evaluation queries (e.g., hundreds of test cases) into a batch request
- Groups queries by model tier for optimal routing and pricing
- Submits via provider batch APIs (Anthropic Message Batches, OpenAI Batch API) at reduced cost (typically 50% discount)
- Manages async job lifecycle: submission → polling → result collection
- Aggregates results back to individual queries for scoring and analysis
- Use cases: nightly eval runs, fine-tuning data validation, A/B model comparison, bulk quality assessments

**Explainability** provides interpretability into model behavior:
- Token Attribution — which input tokens most influenced the output?
- SHAP/LIME Analysis — perturbation-based feature importance scoring
- Chain-of-Thought Analysis — parsing and scoring reasoning quality
- Supports multiple providers (OpenAI, Anthropic, Ollama, HuggingFace)
- Exposes results via REST API and interactive Streamlit dashboard

---

## Data Flow Summary

```
User Query
    │
    ├──► [Guardrails: Input]  ──── BLOCK ──► Error Response
    │         │
    │         PASS
    │         │
    ├──► [Cache: Lookup]  ──── HIT ──► Return Cached Response ──► User
    │         │
    │         MISS
    │         │
    ├──► [Router: Analyze Complexity] ──► Model Tier Decision
    │         │
    │    ┌────┴────────────────────────────────┐
    │    │         PARALLEL RETRIEVAL           │
    │    │                                      │
    │    │  [RAG] ──► Document Chunks           │
    │    │  [GraphRAG] ──► Entities + Relations │
    │    │  [Context Memory] ──► Memories       │
    │    │                                      │
    │    └────┬────────────────────────────────┘
    │         │
    │         ▼
    │    [Context Manager: Assemble + Trim + Summarize]
    │         │
    │         ▼
    │    [LLM Call: Selected Model + Assembled Context]
    │         │
    │         ├──► [Cache: Write response + metadata + TTL]
    │         │
    │         ▼
    │    [Guardrails: Output]  ──── BLOCK ──► Regenerate / Error
    │         │
    │         PASS
    │         │
    │    [Auto-Correction: Score vs Ground Truth]
    │         │         │
    │         │    score < threshold ──► Capture for fine-tuning
    │         │
    │         ▼
    │    Final Response ──► User
    │
    └──► [Observability: Metrics + Traces across all steps]
    └──► [Evaluations: Quality scoring on response]
    │         └──► [Batching: Submit bulk eval queries as batch job]
    └──► [Explainability: Attribution analysis on response]
```

---

## Interaction Between Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FEEDBACK LOOPS                                       │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐         corrections          ┌──────────────────────┐
    │ Auto-        │ ───────────────────────────► │  Model Fine-Tuning   │
    │ Correction   │                              │  (training data)     │
    └──────────────┘                              └──────────────────────┘

    ┌──────────────┐        low eval scores       ┌──────────────────────┐
    │ Evaluations  │ ───────────────────────────► │  Alert / Retrain     │
    │              │                              │  Trigger             │
    └──────────────┘                              └──────────────────────┘

    ┌──────────────┐         cost spikes          ┌──────────────────────┐
    │ Observability│ ───────────────────────────► │  Router Threshold    │
    │              │                              │  Adjustment          │
    └──────────────┘                              └──────────────────────┘

    ┌──────────────┐      reasoning quality       ┌──────────────────────┐
    │Explainability│ ───────────────────────────► │  Prompt Engineering  │
    │              │                              │  Improvements        │
    └──────────────┘                              └──────────────────────┘

    ┌──────────────┐       conversations          ┌──────────────────────┐
    │ Context      │ ───────────────────────────► │  Memory Graph        │
    │ Memory       │  (async extraction)          │  (grows over time)   │
    └──────────────┘                              └──────────────────────┘

    ┌──────────────┐      repeated queries        ┌──────────────────────┐
    │ LLM Cache    │ ───────────────────────────► │  Latency & Cost      │
    │              │  (hit rate metrics)           │  Reduction           │
    └──────────────┘                              └──────────────────────┘

    ┌──────────────┐      bulk eval results       ┌──────────────────────┐
    │ LLM Batching │ ───────────────────────────► │  Evaluation at Scale │
    │              │  (async batch jobs)           │  (nightly runs, A/B) │
    └──────────────┘                              └──────────────────────┘
```

---

## Technology Stack

| Layer | Technologies |
|-------|-------------|
| LLM Providers | OpenAI, Anthropic (Claude), Azure OpenAI, Ollama, HuggingFace |
| APIs | FastAPI, Uvicorn |
| Vector Storage | ChromaDB, In-memory cosine similarity |
| Graph Storage | NetworkX, SQLite |
| Caching | Redis, In-memory LRU, Semantic similarity cache |
| Batch Processing | Anthropic Message Batches API, OpenAI Batch API, asyncio |
| Embeddings | sentence-transformers, OpenAI embeddings |
| Token Counting | tiktoken, Anthropic token API |
| Observability | OpenTelemetry, Prometheus, structlog |
| Evaluation | DeepEval (LLM-as-judge) |
| Explainability | SHAP, LIME, Captum |
| Configuration | YAML (pyyaml), Pydantic Settings |
| Testing | pytest, pytest-asyncio, pytest-cov |
| Code Quality | ruff, mypy, bandit |

---

## Individual Project Links

| Project | Description | Key Capabilities |
|---------|-------------|------------------|
| [ai-genai-llm-guardrails](../ai-genai-llm-guardrails) | Input/output safety layer | PII detection/redaction, prompt injection, toxicity, topic restriction |
| [ai-genai-llm-router](../ai-genai-llm-router) | Intelligent query routing | 6-signal complexity analysis, cost optimization, tier selection |
| [ai-genai-context-manager](../ai-genai-context-manager) | Token window management | FIFO/sliding/priority trimming, LLM summarization, async-first |
| [ai-genai-context-memory](../ai-genai-context-memory) | Persistent memory graph | Brain-like memory, semantic search, async extraction, LLM proxy |
| [ai-genai-rag](../ai-genai-rag) | Document retrieval pipeline | Ingestion, hybrid search, RRF reranking, source attribution |
| [ai-genai-graphrag](../ai-genai-graphrag) | Knowledge graph retrieval | Entity/relationship extraction, BFS traversal, community detection |
| [ai-genai-autocorrection-pipeline](../ai-genai-autocorrection-pipeline) | Quality enforcement | Ground truth comparison, correction capture, fine-tuning data |
| [ai-genai-llm-observability](../ai-genai-llm-observability) | Performance monitoring | TTFT, latency, cost, OTLP traces, Prometheus metrics |
| [ai-genai-llm-evaluations](../ai-genai-llm-evaluations) | Response quality scoring | 6 metrics via DeepEval, LLM-as-judge, batch evaluation |
| [ai-genai-llm-explainability](../ai-genai-llm-explainability) | Model interpretability | Token attribution, SHAP/LIME, CoT analysis, Streamlit dashboard |
| [ai-genai-llm-caching](../ai-genai-llm-caching) | LLM response caching | Exact match, semantic similarity cache, TTL management, cost savings |
| [ai-genai-llm-batching](../ai-genai-llm-batching) | Batch query processing | Bulk eval submission, async job management, batch API pricing, result aggregation |

---

## Cloud & Platform Implementations

| Platform | Document | Description |
|----------|----------|-------------|
| AWS | [end-to-end-pipeline-aws.md](end-to-end-pipeline-aws.md) | Full pipeline mapped to AWS managed services (Bedrock, OpenSearch, Neptune, etc.) |
| Azure | [end-to-end-pipeline-azure.md](end-to-end-pipeline-azure.md) | Full pipeline mapped to Azure services (OpenAI, AI Search, Cosmos DB, Foundry, etc.) |
| GCP | [end-to-end-pipeline-gcp.md](end-to-end-pipeline-gcp.md) | Full pipeline mapped to GCP services (Vertex AI, Gemini, Model Armor, Firestore, etc.) |
| Open-Source | [end-to-end-pipeline-opensource.md](end-to-end-pipeline-opensource.md) | Full pipeline using open-source tools (vLLM, LlamaIndex, Qdrant, Langfuse, etc.) |

---

## Component Documentation

Detailed documentation for individual pipeline components:

| Component | Document |
|-----------|----------|
| Context Memory | [components/context-memory.md](components/context-memory.md) |

---

## License

MIT
