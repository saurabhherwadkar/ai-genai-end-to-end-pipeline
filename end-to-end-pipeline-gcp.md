# AI GenAI End-to-End Pipeline — GCP Implementation

A production-grade implementation of the GenAI LLM application pipeline using Google Cloud managed services. This document maps each pipeline component to its GCP equivalent, leveraging [Vertex AI](https://cloud.google.com/vertex-ai) as the unified AI platform and following Google Cloud's recommended architecture patterns for generative AI applications.

---

## GCP Service Mapping

| # | Pipeline Component | GCP Service(s) |
|---|-------------------|----------------|
| 1 | Input/Output Guardrails | Model Armor + Vertex AI Safety Settings + Sensitive Data Protection |
| 2 | LLM Router | Custom (Cloud Run + countTokens API) — no native managed router |
| 3 | Context Manager | Cloud Run / Cloud Functions (2nd Gen) |
| 4 | Context Memory | Firestore (vector) / AlloyDB AI + Agent Platform Memory Bank |
| 5 | RAG Pipeline | Vertex AI RAG Engine + Vertex AI Search |
| 6 | GraphRAG | Neo4j Aura on GCP + Custom Graph Pipeline |
| 7 | Auto-Correction | Cloud Workflows + Vertex AI + Cloud Storage |
| 8 | Observability | Cloud Monitoring + Cloud Trace + Cloud Logging |
| 9 | Evaluations | Vertex AI Gen AI Evaluation Service |
| 10 | Explainability | Grounding with Citations + Gen AI Evaluation (custom metrics) |
| 11 | LLM Caching | Memorystore for Redis + Vertex AI Context Caching |
| 12 | LLM Batching | Vertex AI Batch Prediction + Cloud Workflows |
| 13 | Orchestration | Gemini Enterprise Agent Platform + Agent Development Kit (ADK) |
| 14 | Infrastructure | Cloud Run + Cloud Functions + Pub/Sub + Eventarc + API Gateway |

---

## End-to-End Architecture Diagram (GCP)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    USER / CLIENT APP                                     │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              CLOUD ENDPOINTS / API GATEWAY                                │
│                    ┌──────────────────────────────────────────────────┐                   │
│                    │  Cloud Armor (WAF) + Rate Limiting + IAM Auth    │                   │
│                    └──────────────────────────────────────────────────┘                   │
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
│   │                         MODEL ARMOR (Input)                                    │     │
│   │                    [Standalone API — no model invocation needed]               │     │
│   │                                                                               │     │
│   │   ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│   │   │ PII          │  │ Responsible AI    │  │  Prompt     │  │  Malicious │  │     │
│   │   │ Detection &  │  │ Filters (Hate,    │  │  Injection  │  │  URL       │  │     │
│   │   │ Redaction    │  │  Harassment,      │  │  Detection  │  │  Detection │  │     │
│   │   │              │  │  Sexual, Danger)  │  │  & Jailbreak│  │            │  │     │
│   │   └──────┬───────┘  └─────────┬─────────┘  └──────┬──────┘  └─────┬──────┘  │     │
│   │          └─────────────────────┴───────────────────┴───────────────┘         │     │
│   │                                    │                                          │     │
│   │                    ┌───────────────┴────────────────────┐                    │     │
│   │                    │ Sensitive Data Protection           │                    │     │
│   │                    │ (200+ info type detectors)          │                    │     │
│   │                    └───────────────┬────────────────────┘                    │     │
│   │                                    │                                          │     │
│   │                          ┌─────────┴─────────┐                               │     │
│   │                          │  BLOCK or PASS    │                               │     │
│   │                          └─────────┬─────────┘                               │     │
│   └────────────────────────────────────┼──────────────────────────────────────────┘     │
│                                        │                                                 │
│                                        ▼                                                 │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │              LLM CACHE (Memorystore Redis + Vertex AI Context Caching)         │     │
│   │                                                                               │     │
│   │   ┌──────────────────┐       ┌──────────────────────────────────────────┐    │     │
│   │   │  Incoming Query  │──────▶│           CACHE LOOKUP                   │    │     │
│   │   └──────────────────┘       │                                          │    │     │
│   │                              │  Strategy A: Exact Match (Redis hash)     │    │     │
│   │                              │  Strategy B: Semantic Similarity          │    │     │
│   │                              │    (Gemini Embedding cosine ≥ 0.95)       │    │     │
│   │                              │  Strategy C: Context Caching             │    │     │
│   │                              │    (GenAiCacheService for prefixes)       │    │     │
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
│   │              CUSTOM LLM ROUTER (Cloud Run Service)                             │     │
│   │              [Complexity-based routing — custom implementation]                │     │
│   │                                                                               │     │
│   │   ┌──────────────────────────────────────────────────────────────────────┐   │     │
│   │   │  Analyzes query complexity → Routes to optimal Gemini tier           │   │     │
│   │   │  Uses countTokens API for cost estimation                            │   │     │
│   │   │  Context Caching for repeated prefixes                               │   │     │
│   │   └──────────────────────────────────────────────────────────────────────┘   │     │
│   │                                                                               │     │
│   │              ┌─────────────────────┼─────────────────────┐                   │     │
│   │              ▼                     ▼                     ▼                   │     │
│   │      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐             │     │
│   │      │Gemini 3.5   │      │ Gemini 2.5  │      │ Gemini 3.1  │             │     │
│   │      │  Flash      │      │  Flash      │      │  Pro        │             │     │
│   │      │  Fast/Cheap │      │  Balanced   │      │  Capable    │             │     │
│   │      └─────────────┘      └─────────────┘      └─────────────┘             │     │
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
│   │  VERTEX AI RAG       │  │  GRAPHRAG            │  │   CONTEXT MEMORY         │     │
│   │  ENGINE              │  │  (Neo4j Aura on GCP  │  │   (Firestore + Agent     │     │
│   │                      │  │   + Custom Pipeline) │  │    Platform Memory Bank) │     │
│   │  Data Sources:       │  │                      │  │                          │     │
│   │  • Cloud Storage     │  │  Graph Store:        │  │  Query                   │     │
│   │  • Google Drive      │  │  • Neo4j Aura       │  │    │                     │     │
│   │  • Web URLs          │  │    (Marketplace)     │  │    ▼                     │     │
│   │  • Slack, Jira       │  │                      │  │  Semantic Similarity     │     │
│   │                      │  │  Features:           │  │  Search (Firestore       │     │
│   │  Vector Stores:      │  │  • Entity extraction │  │   vector engine)         │     │
│   │  • Vertex AI Vector  │  │    (Gemini-driven)   │  │    │                     │     │
│   │    Search            │  │  • Graph traversal   │  │    ▼                     │     │
│   │  • AlloyDB AI        │  │  • Community         │  │  Firestore / AlloyDB     │     │
│   │  • Firestore         │  │    detection         │  │  (node/edge persistence) │     │
│   │  • BigQuery          │  │  • Multi-hop         │  │    │                     │     │
│   │  • Memorystore       │  │    reasoning         │  │    ▼                     │     │
│   │  • Pinecone/Weaviate │  │                      │  │  Ranked Memories         │     │
│   │                      │  │  Query:              │  │  (thoughts, beliefs,     │     │
│   │  Retrieval:          │  │  • Entity matching   │  │   preferences, goals)    │     │
│   │  • Semantic search   │  │  • Subgraph extract  │  │                          │     │
│   │  • Hybrid retrieval  │  │  • Community context │  │  Async Extraction:       │     │
│   │  • Cross-corpus      │  │                      │  │  • Eventarc trigger      │     │
│   │  • Citations         │  │                      │  │  • Cloud Functions +     │     │
│   │                      │  │                      │  │    Gemini                 │     │
│   └──────────┬───────────┘  └──────────┬───────────┘  └────────────┬─────────────┘     │
│              │                          │                            │                   │
│              └──────────────────────────┼────────────────────────────┘                   │
│                                         │                                                │
│                                         ▼                                                │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                  CONTEXT MANAGER (Cloud Run Service)                            │     │
│   │                                                                               │     │
│   │  ┌─────────────────┐  ┌───────────────────┐  ┌────────────────────────────┐  │     │
│   │  │  Token Counter  │  │ Trimming Strategy │  │  Summarization via        │  │     │
│   │  │  (countTokens   │  │ (FIFO / Sliding   │  │  Vertex AI                │  │     │
│   │  │   API / Gen AI  │  │  Window / Priority)│  │  (Gemini 3.5 Flash —     │  │     │
│   │  │   SDK)          │  │                    │  │   fast, cheap summary)    │  │     │
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
│   │                     VERTEX AI / GEMINI API                                     │     │
│   │                                                                               │     │
│   │         ┌───────────────────────────────────────────────────────┐            │     │
│   │         │              MODEL INVOCATION                          │            │     │
│   │         │                                                       │            │     │
│   │         │  • Google Gen AI SDK (unified client)                 │            │     │
│   │         │  • Streaming via generateContentStream               │            │     │
│   │         │  • Built-in retry with exponential backoff            │            │     │
│   │         │  • Safety settings applied inline                     │            │     │
│   │         │  • Context Caching for repeated prefixes              │            │     │
│   │         │                                                       │            │     │
│   │         │  Models Available (Google):                            │            │     │
│   │         │  • Gemini 3.1 Pro / 2.5 Pro (most capable)           │            │     │
│   │         │  • Gemini 3 Flash / 2.5 Flash (balanced)             │            │     │
│   │         │  • Gemini 3.5 Flash (fastest/cheapest)               │            │     │
│   │         │  • Gemini Embedding 2 (vector representations)       │            │     │
│   │         │                                                       │            │     │
│   │         │  Partner Models (Model Garden):                        │            │     │
│   │         │  • Claude Opus 4.7 / Fable 5 (Anthropic)             │            │     │
│   │         │  • Llama 4 (Meta), Mistral, DeepSeek, Qwen           │            │     │
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
│   │                    MODEL ARMOR (Output)                                        │     │
│   │                 [Same guardrails applied to model responses]                   │     │
│   │                                                                               │     │
│   │   ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│   │   │ PII Redactor │  │  Responsible AI   │  │  Content    │  │  Malicious │  │     │
│   │   │ (Sensitive   │  │  Filters          │  │  Safety     │  │  URL       │  │     │
│   │   │  Data        │  │  (Toxicity)       │  │  (Harmful   │  │  Detection │  │     │
│   │   │  Protection) │  │                   │  │   Content)  │  │            │  │     │
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
│   │              AUTO-CORRECTION PIPELINE (Cloud Workflows)                        │     │
│   │                                                                               │     │
│   │   ┌──────────────────┐       ┌────────────────────────────────────────┐      │     │
│   │   │  Actual Response │──────▶│           COMPARATOR                   │      │     │
│   │   └──────────────────┘       │           (Cloud Function)             │      │     │
│   │                              │                                        │      │     │
│   │                              │  Strategy A: Gemini Embeddings         │      │     │
│   │                              │    (Embedding 2 — cosine sim)          │      │     │
│   │   ┌──────────────────┐       │  Strategy B: Gemini LLM-as-Judge      │      │     │
│   │   │  Ideal Response  │──────▶│    (Gemini Pro evaluation)            │      │     │
│   │   │  (Cloud Storage  │       │                                        │      │     │
│   │   │   ground truth)  │       │  Output: Confidence Score (0.0 - 1.0)  │      │     │
│   │   └──────────────────┘       └───────────────────┬────────────────────┘      │     │
│   │                                                   │                           │     │
│   │                                ┌──────────────────┴──────────────────┐        │     │
│   │                                │                                     │        │     │
│   │                          score >= 0.8                          score < 0.8    │     │
│   │                                │                                     │        │     │
│   │                                ▼                                     ▼        │     │
│   │                          ┌──────────┐                  ┌─────────────────┐   │     │
│   │                          │   PASS   │                  │ Capture to GCS  │   │     │
│   │                          └──────────┘                  │ for fine-tuning │   │     │
│   │                                                        │ (Vertex AI)     │   │     │
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
│   │    OBSERVABILITY       │ │     EVALUATIONS        │ │     EXPLAINABILITY         │  │
│   │                        │ │                        │ │                            │  │
│   │  Cloud Monitoring:     │ │  Gen AI Evaluation     │ │  Grounding with Citations: │  │
│   │  • Custom Metrics      │ │  Service:              │ │  • Source attribution      │  │
│   │    (TTFT, latency,     │ │  • EvalTask SDK        │ │  • RAG Engine citations    │  │
│   │     token usage)       │ │  • Pointwise metrics   │ │  • Vertex AI Search        │  │
│   │  • Dashboards          │ │    (coherence,         │ │    provenance              │  │
│   │  • Alert Policies      │ │     relevance,         │ │                            │  │
│   │                        │ │     fluency)           │ │  Custom Metrics:           │  │
│   │  Cloud Trace:          │ │  • Pairwise metrics    │ │  • Faithfulness scoring    │  │
│   │  • OpenTelemetry       │ │    (AutoSxS)           │ │  • Groundedness checks     │  │
│   │  • Distributed tracing │ │  • Safety metrics      │ │  • Reasoning quality       │  │
│   │  • SQL-based querying  │ │    (CSAM, toxicity)    │ │                            │  │
│   │                        │ │  • Custom metrics      │ │  Cloud Logging:            │  │
│   │  Cloud Logging:        │ │    (rubric-based)      │ │  • Full request/response   │  │
│   │  • Structured JSON     │ │                        │ │    capture                 │  │
│   │  • Log Analytics       │ │  Partner Evaluations:  │ │  • Prompt/response audit   │  │
│   │  • BigQuery export     │ │  • Claude, Llama as    │ │    via BigQuery            │  │
│   │  • Log-based metrics   │ │    judge models        │ │                            │  │
│   │                        │ │                        │ │                            │  │
│   │  Agent Engine:         │ │                        │ │                            │  │
│   │  • Session traces      │ │                        │ │                            │  │
│   │  • Tool invocations    │ │                        │ │                            │  │
│   │  • Agent-to-agent      │ │                        │ │                            │  │
│   │    communication logs  │ │                        │ │                            │  │
│   └────────────────────────┘ └────────────┬───────────┘ └────────────────────────────┘  │
│                                           │                                              │
│                                           ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                  LLM BATCHING (Vertex AI Batch Prediction)                        │   │
│   │                                                                                  │   │
│   │   ┌────────────────────────────────────────────────────────────────────────┐    │   │
│   │   │                      BATCH JOB MANAGER                                 │    │   │
│   │   │                                                                        │    │   │
│   │   │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │    │   │
│   │   │  │  GCS Input JSONL │  │  Batch Prediction│  │  GCS Output JSONL   │  │    │   │
│   │   │  │  (collect eval   │  │  Job (Vertex AI  │  │  (results mapped    │  │    │   │
│   │   │  │   queries)       │  │   BatchPredict-  │  │   back to queries)  │  │    │   │
│   │   │  │                  │  │   ionJob)        │  │                     │  │    │   │
│   │   │  └────────┬─────────┘  └────────┬─────────┘  └──────────┬──────────┘  │    │   │
│   │   │           └─────────────────────┼────────────────────────┘             │    │   │
│   │   │                                 │                                      │    │   │
│   │   │              Up to 50% cost savings vs real-time inference             │    │   │
│   │   └────────────────────────────────────────────────────────────────────────┘    │   │
│   │                                                                                  │   │
│   │   Use Cases:                                                                     │   │
│   │   • Batch evaluation scoring (submit 100s of eval queries at once)              │   │
│   │   • Offline quality assessments (nightly eval runs via Cloud Scheduler)          │   │
│   │   • Fine-tuning data validation (bulk comparison against ground truth)          │   │
│   │   • A/B test evaluation (AutoSxS pairwise comparison at scale)                  │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Phases (GCP Implementation)

### Phase 1: Input Processing

#### Model Armor (Input)

[Model Armor](https://cloud.google.com/model-armor/docs/overview) provides runtime security for GenAI applications as a standalone API:

| Safeguard Type | Function | Configuration |
|---------------|----------|---------------|
| Prompt Injection Detection | Detects jailbreak and injection attempts | Confidence levels: High, Medium+, Low+ |
| Responsible AI Filters | Blocks hate, harassment, sexual content, dangerous content | Adjustable thresholds per category |
| PII Detection & Redaction | Detects/masks sensitive information in prompts | Integration with Sensitive Data Protection |
| Malicious URL Detection | Identifies harmful URLs in prompts/responses | Automatic blocking |
| Custom Security Rules | Organization-wide floor settings | Admin-configurable templates |

**Modes:** "Inspect only" (log violations) or "Inspect and block" (actively prevent)

**Key capability:** Model Armor operates independently of model invocation — it can run as a standalone preprocessing step. Integrates with Agent Platform, ADK, Apigee, and GKE.

```python
from google.cloud import modelarmor_v1

client = modelarmor_v1.ModelArmorClient()

request = modelarmor_v1.SanitizeUserPromptRequest(
    name=f"projects/{PROJECT}/locations/{LOCATION}/templates/{TEMPLATE}",
    user_prompt_data=modelarmor_v1.UserPromptData(
        text=user_input
    )
)

response = client.sanitize_user_prompt(request=request)

if response.sanitization_result.filter_match_state == modelarmor_v1.FilterMatchState.MATCH_FOUND:
    return {"blocked": True, "reason": response.sanitization_result.filter_results}
```

#### Sensitive Data Protection (PII Layer)

[Sensitive Data Protection](https://cloud.google.com/sensitive-data-protection) (formerly Cloud DLP) provides deep PII detection:

- 200+ built-in info type detectors (SSN, credit cards, credentials, medical records)
- Custom info types with regex and dictionary
- De-identification techniques: redaction, masking, tokenization, format-preserving encryption
- Integration: scan prompts before sending to models, mask sensitive data in outputs

```python
from google.cloud import dlp_v2

dlp_client = dlp_v2.DlpServiceClient()

response = dlp_client.inspect_content(
    request={
        "parent": f"projects/{PROJECT}/locations/global",
        "inspect_config": {
            "info_types": [
                {"name": "PHONE_NUMBER"},
                {"name": "EMAIL_ADDRESS"},
                {"name": "US_SOCIAL_SECURITY_NUMBER"}
            ],
            "min_likelihood": dlp_v2.Likelihood.LIKELY
        },
        "item": {"value": user_input}
    }
)

if response.result.findings:
    return {"blocked": True, "pii_detected": [f.info_type.name for f in response.result.findings]}
```

#### LLM Cache (Memorystore Redis + Context Caching)

Two caching layers reduce cost and latency:

**Layer 1: Response Cache (Memorystore for Redis)**
- Exact match: Hash-based key lookup for identical queries (sub-millisecond)
- Semantic match: Embedding similarity via Gemini Embedding 2 (cosine threshold ≥ 0.95)
- TTL management: Configurable expiry per query type
- On **CACHE HIT**: returns cached response immediately, bypassing the full pipeline
- On **CACHE MISS**: continues to Router; after LLM response, writes to cache

**Layer 2: Context Prefix Cache (Vertex AI Context Caching / GenAiCacheService)**
- Caches repeated system prompts and context prefixes across requests
- Up to 75% cost reduction on cached tokens
- Supports time-based and usage-based TTL
- Works with Gemini models natively

```python
import redis
import hashlib
import json

cache = redis.Redis(host="memorystore-ip", port=6379)

def check_cache(query: str) -> dict | None:
    cache_key = hashlib.sha256(query.encode()).hexdigest()
    cached = cache.get(f"llm:exact:{cache_key}")
    if cached:
        return json.loads(cached)
    return None

def write_cache(query: str, response: dict, ttl: int = 3600):
    cache_key = hashlib.sha256(query.encode()).hexdigest()
    cache.setex(f"llm:exact:{cache_key}", ttl, json.dumps(response))
```

#### Custom LLM Router (Cloud Run)

GCP does not provide a native intelligent routing service. The router is implemented as a custom Cloud Run service:

- Analyzes query complexity using heuristic signals (length, vocabulary, domain keywords, question type)
- Uses `countTokens` API for cost estimation before routing
- Leverages Context Caching (`GenAiCacheService`) for repeated context prefixes
- Routes across Gemini model tiers based on complexity score

```python
from google import genai

client = genai.Client(vertexai=True, project=PROJECT, location=LOCATION)

token_count = client.models.count_tokens(
    model="gemini-2.5-flash",
    contents=user_query
)

complexity_score = analyze_complexity(user_query)

if complexity_score <= 0.3:
    model = "gemini-3.5-flash"
elif complexity_score <= 0.7:
    model = "gemini-2.5-flash"
else:
    model = "gemini-3.1-pro"
```

---

### Phase 2: Context Assembly

#### Vertex AI RAG Engine

[Vertex AI RAG Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-overview) provides a managed RAG data framework:

**Data Sources:**
- Cloud Storage (PDF, CSV, TXT, DOCX, HTML)
- Google Drive
- Web URLs (crawled)
- Slack, Jira connectors

**Vector Stores (choose one):**
- Vertex AI Vector Search (managed ANN, low-latency at scale)
- AlloyDB AI (PostgreSQL + pgvector, IVFFlat & HNSW indexes)
- Firestore (serverless, auto-scaling vector search)
- BigQuery (analytics + vector in one place)
- Memorystore for Redis (ultra-low latency)
- Pinecone, Weaviate (third-party)

**Retrieval Pipeline:**
1. Automatic document chunking (configurable strategies)
2. Embedding generation (Gemini Embedding 2 or custom)
3. Semantic search with configurable top-K
4. Cross-corpus retrieval (public preview)
5. Citations with source attribution
6. Metadata filtering

```python
from vertexai import rag

corpus = rag.get_corpus(name=f"projects/{PROJECT}/locations/{LOCATION}/ragCorpora/{CORPUS_ID}")

response = rag.retrieval_query(
    rag_resources=[rag.RagResource(rag_corpus=corpus.name)],
    text="What is our refund policy?",
    similarity_top_k=10,
    vector_distance_threshold=0.7
)

for context in response.contexts.contexts:
    print(f"Source: {context.source_uri}, Score: {context.distance}")
    print(f"Text: {context.text}")
```

#### Vertex AI Search (Low-Code RAG Alternative)

[Vertex AI Search](https://cloud.google.com/enterprise-search) provides an end-to-end search engine with minimal setup:

- AI-powered grounding with Gemini
- Advanced filtering and custom embeddings
- Blended data queries across structured and unstructured sources
- Built-in ranking and relevance tuning

**Key distinction:**
- **Vertex AI Search** = Out-of-box, low-code RAG (minimal setup)
- **Vertex AI RAG Engine** = Customizable RAG framework (pluggable components)

#### GraphRAG (Neo4j Aura + Custom Pipeline)

GCP does not provide a native GraphRAG service. The recommended approach uses Neo4j on Google Cloud Marketplace:

**Graph Store:** Neo4j Aura (managed graph database on GCP)

**Architecture:**
```
Documents (Cloud Storage) → Gemini (Entity Extraction)
                                    │
                                    ▼
                          Neo4j Aura (Graph Database)
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Entity Match    Graph Traversal  Community Context
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
                          Enriched Context for LLM
```

**Neo4j on GCP Features:**
- AuraDB: Fully managed graph database
- Virtual Graph: Knowledge graphs on existing data without migration
- Graph Analytics: Community detection, centrality, path-finding algorithms
- Integration with Vertex AI RAG Engine

**Alternative:** Use LightRAG (open-source) deployed on Cloud Run with AlloyDB as the storage backend.

#### Context Memory (Firestore + Agent Platform Memory Bank)

| Layer | GCP Service | Purpose |
|-------|-------------|---------|
| Node/Edge Persistence | Firestore | Memory nodes, edges, metadata with TTL |
| Vector Search | Firestore (vector engine) or AlloyDB AI | Semantic similarity via ANN |
| Embedding Generation | Vertex AI (Gemini Embedding 2) | Convert memory content to vectors |
| Async Extraction | Eventarc + Cloud Functions | Fire-and-forget memory extraction |
| Session State | Agent Platform Memory Bank | Managed long-term context for agents |

```python
from google.cloud import firestore

db = firestore.Client()

collection = db.collection("memories")
query = collection.where("type", "in", ["thought", "belief", "preference"])

results = query.find_nearest(
    vector_field="embedding",
    query_vector=query_embedding,
    distance_measure=firestore.DistanceMeasure.COSINE,
    limit=10
)

for doc in results:
    memory = doc.to_dict()
    print(f"Type: {memory['type']}, Content: {memory['content']}")
```

#### Context Manager (Cloud Run)

Token counting and context assembly runs as a Cloud Run service:

- **Token counting:** `countTokens` API on Vertex AI Publisher Models or Google Gen AI SDK
- **Trimming strategies:** FIFO, sliding window, priority-based (implemented in application code)
- **Summarization:** Calls Vertex AI (Gemini 3.5 Flash — fastest, cheapest) to condense older turns
- **Context Caching:** `GenAiCacheService` for caching repeated context prefixes (reduces cost)
- **Output:** Token-budgeted context window ready for the final LLM call

---

### Phase 3: LLM Execution

#### Vertex AI / Gemini API

The [Google Gen AI SDK](https://cloud.google.com/vertex-ai/generative-ai/docs/sdks) provides a unified interface across all Gemini models:

```python
from google import genai

client = genai.Client(vertexai=True, project=PROJECT, location=LOCATION)

response = client.models.generate_content_stream(
    model="gemini-3.1-pro",
    contents=[
        genai.types.Content(role="user", parts=[
            genai.types.Part(text=assembled_user_message)
        ])
    ],
    config=genai.types.GenerateContentConfig(
        system_instruction=system_prompt_with_memories,
        max_output_tokens=4096,
        temperature=0.7,
        safety_settings=[
            genai.types.SafetySetting(
                category="HARM_CATEGORY_DANGEROUS_CONTENT",
                threshold="BLOCK_MEDIUM_AND_ABOVE"
            )
        ]
    )
)

for chunk in response:
    yield chunk.text
```

**Key features:**
- Unified SDK across all Gemini models + partner models (Model Garden)
- Native streaming via `generate_content_stream`
- Inline safety settings (configurable per category)
- Context Caching for repeated prefixes (reduces latency and cost)
- Tool use / function calling support
- Grounding with Google Search or custom data sources
- Code execution in sandboxed environment

---

### Phase 4: Output Processing

#### Model Armor (Output)

The same Model Armor configuration applied during input processing also validates model responses:

- **PII Redaction:** Sensitive Data Protection masks any PII that leaked into output
- **Content Safety:** Responsible AI Filters catch harmful content in responses
- **Malicious URLs:** Detects and blocks harmful URLs generated in output
- **Custom Policies:** Organization-wide floor settings enforced on all responses

```python
response = client.sanitize_model_response(
    name=f"projects/{PROJECT}/locations/{LOCATION}/templates/{TEMPLATE}",
    model_response_data=modelarmor_v1.ModelResponseData(
        text=model_output
    )
)

if response.sanitization_result.filter_match_state == modelarmor_v1.FilterMatchState.MATCH_FOUND:
    return regenerate_or_error(response)
```

#### Auto-Correction Pipeline (Cloud Workflows)

```
┌─────────────────────────────────────────────────────────────────┐
│                  CLOUD WORKFLOWS ORCHESTRATION                    │
│                                                                  │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────────┐    │
│  │ Start   │───▶│ Cloud Func:  │───▶│ Cloud Func:         │    │
│  │         │    │ Fetch Ground │    │ Compare (Cosine     │    │
│  │         │    │ Truth (GCS)  │    │ Sim / LLM-Judge)    │    │
│  └─────────┘    └──────────────┘    └──────────┬──────────┘    │
│                                                 │               │
│                              ┌──────────────────┴────────┐      │
│                              │                           │      │
│                        score >= 0.8               score < 0.8   │
│                              │                           │      │
│                              ▼                           ▼      │
│                      ┌────────────┐         ┌────────────────┐  │
│                      │ Pass       │         │ Cloud Func:    │  │
│                      │ (no-op)    │         │ Write to GCS   │  │
│                      └────────────┘         │ (correction    │  │
│                                             │  record)       │  │
│                                             └───────┬────────┘  │
│                                                     │           │
│                                                     ▼           │
│                                             ┌────────────────┐  │
│                                             │ Vertex AI:     │  │
│                                             │ Fine-tuning    │  │
│                                             │ pipeline       │  │
│                                             └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Components:**
- **Ground truth store:** Cloud Storage (structured JSONL files)
- **Comparison strategies:**
  - Cosine similarity via Vertex AI Gemini Embedding 2
  - LLM-as-Judge via Vertex AI (Gemini Pro evaluates response quality)
- **Correction capture:** Cloud Storage + Firestore (query + actual + ideal + metadata)
- **Fine-tuning pipeline:** Vertex AI supervised tuning with accumulated corrections

---

### Phase 5: Monitoring & Continuous Improvement

#### Observability

| Metric | GCP Service | Implementation |
|--------|------------|----------------|
| Time to First Token (TTFT) | Cloud Monitoring (custom metrics) | Measure streaming start in Cloud Run |
| Total Latency | Cloud Trace | End-to-end distributed trace via OpenTelemetry |
| Inter-Token Latency | Cloud Monitoring (custom metrics) | Calculated from stream events |
| Token Usage | Cloud Logging (structured JSON) | Log token counts per request |
| Cost per Request | Cloud Logging + BigQuery | Calculate from token usage × pricing |
| Error Rates | Cloud Monitoring Alert Policies | Threshold-based alerting |
| Distributed Tracing | Cloud Trace (OpenTelemetry) | Service dependency map + latency |

**OpenTelemetry Integration:**
```python
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

tracer_provider = TracerProvider()
tracer_provider.add_span_processor(
    BatchSpanProcessor(CloudTraceSpanExporter(project_id=PROJECT))
)
trace.set_tracer_provider(tracer_provider)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("llm_pipeline") as span:
    span.set_attribute("gen_ai.request.model", "gemini-3.1-pro")
    span.set_attribute("gen_ai.usage.prompt_tokens", prompt_tokens)
    span.set_attribute("gen_ai.usage.completion_tokens", completion_tokens)
```

**Cloud Logging for LLM Observability:**
```python
import google.cloud.logging
import json

logging_client = google.cloud.logging.Client()
logger = logging_client.logger("genai-pipeline")

logger.log_struct({
    "request_id": request_id,
    "model": "gemini-3.1-pro",
    "prompt_tokens": prompt_tokens,
    "completion_tokens": completion_tokens,
    "latency_ms": latency_ms,
    "cost_usd": cost,
    "ttft_ms": ttft
})
```

#### Evaluations

**Vertex AI Gen AI Evaluation Service** (managed):

| Metric Type | Metrics Available |
|------------|-------------------|
| Pointwise Quality | Coherence, Relevance, Fluency, Groundedness, Safety |
| Pairwise Comparison | AutoSxS (side-by-side model comparison) |
| Safety | CSAM detection, Toxicity, Dangerous content |
| Custom Metrics | Rubric-based scoring with custom criteria |
| RAG-Specific | Faithfulness, Context Relevance, Answer Relevance |
| Text Similarity | ROUGE, BLEU, METEOR |

```python
from vertexai.evaluation import EvalTask, PointwiseMetric

eval_task = EvalTask(
    dataset="gs://bucket/eval-dataset.jsonl",
    metrics=[
        "coherence",
        "relevance",
        "groundedness",
        "fluency",
        PointwiseMetric(
            metric="custom_faithfulness",
            metric_prompt_template="Rate faithfulness 1-5: {response} given {context}"
        )
    ],
    experiment="pipeline-eval-v1"
)

results = eval_task.evaluate(model=model)
print(f"Coherence: {results.summary_metrics['coherence/mean']}")
```

**AutoSxS Pipeline:** Automatic pairwise side-by-side evaluation for comparing model outputs systematically (e.g., comparing Gemini Flash vs Pro on your specific use case).

#### LLM Batching (Vertex AI Batch Prediction)

[Vertex AI Batch Prediction](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-gemini) enables submitting large volumes of evaluation queries at reduced cost:

- Submit JSONL files to Cloud Storage with hundreds/thousands of inference requests
- Vertex AI processes asynchronously with up to 50% cost savings vs real-time
- Results written to GCS output bucket, mapped back to original requests
- Supports all Gemini models and partner models in Model Garden

```python
from google.cloud import aiplatform

aiplatform.init(project=PROJECT, location=LOCATION)

batch_job = aiplatform.BatchPredictionJob.create(
    job_display_name="nightly-eval-run",
    model_name="publishers/google/models/gemini-2.5-flash",
    gcs_source="gs://eval-bucket/input/eval-queries.jsonl",
    gcs_destination_prefix="gs://eval-bucket/output/",
    instances_format="jsonl",
    predictions_format="jsonl"
)

batch_job.wait()
print(f"Output: {batch_job.output_info.gcs_output_directory}")
```

**Scheduling:** Use Cloud Scheduler + Cloud Workflows to trigger nightly/weekly batch evaluation runs.

#### Explainability

| Approach | GCP Service | Capability |
|----------|-------------|-----------|
| Source Attribution | Vertex AI RAG Engine / Search | Citations with source URIs |
| Grounding Verification | Gen AI Evaluation (custom metrics) | Faithfulness and groundedness scoring |
| Reasoning Quality | Gen AI Evaluation (custom metrics) | Chain-of-thought quality assessment |
| Full Audit Trail | Cloud Logging + BigQuery | Query logged prompts/responses with SQL |
| Agent Reasoning | Agent Engine Observability | Session traces, tool invocations, decisions |

**Note:** Vertex Explainable AI (feature-based explanations with SHAP/Integrated Gradients) is deprecated and ending March 2027. It was designed for traditional ML, not generative AI. Use grounding with citations and evaluation metrics instead.

---

## Orchestration Services

### Gemini Enterprise Agent Platform

[Agent Platform](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-builder/overview) provides managed agent deployment:

**Components:**
| Component | Purpose |
|-----------|---------|
| Agent Runtime | Serverless agent execution |
| Sessions | Multi-turn state management |
| Memory Bank | Long-term agent context storage |
| Code Execution | Sandboxed code generation and execution |
| Function Calling | External API integration |
| A2A Communication | Agent-to-agent protocol support |

**Deployment Options:**
- Agent Runtime (serverless, fully managed)
- Cloud Run (custom container)
- GKE (Kubernetes, maximum control)

### Agent Development Kit (ADK)

[ADK](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-development-kit) is Google's open-source framework for building agents:

```python
from google.adk import Agent, Tool
from google import genai

@Tool
def search_documents(query: str) -> str:
    """Search the knowledge base for relevant documents."""
    return rag_engine.query(query)

@Tool
def check_memory(query: str) -> str:
    """Retrieve relevant memories for context."""
    return memory_service.search(query)

agent = Agent(
    model="gemini-3.1-pro",
    name="pipeline-orchestrator",
    instructions="Orchestrate the GenAI pipeline...",
    tools=[search_documents, check_memory]
)

response = await agent.run("Process this user query")
```

**ADK CLI (`agents-cli`):**
- Scaffold new agent projects
- Local testing and debugging
- Remote deployment to Agent Engine
- Evaluation and CI/CD setup
- RAG provisioning

### Cloud Workflows (Orchestration)

[Cloud Workflows](https://cloud.google.com/workflows) for deterministic multi-step pipelines:

- Serverless orchestration (scales to zero, pay-per-step)
- Supports: Cloud Run, Cloud Functions, external HTTP APIs
- Advanced patterns: parallel iteration, error handling, retry logic
- Human-in-the-loop approval steps
- Long-running workflow support

---

## Infrastructure Architecture

### Recommended: Cloud Run-First with Serverless Events

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLOUD RUN ARCHITECTURE                                │
│                                                                              │
│   ┌──────────────┐    ┌──────────────────────────────────────────────┐      │
│   │ Cloud        │───▶│ Cloud Run Services                            │      │
│   │ Endpoints    │    │                                              │      │
│   │ + Cloud      │    │  ┌────────────┐  ┌────────────┐  ┌────────┐ │      │
│   │  Armor       │    │  │ FastAPI    │  │ Context    │  │ Memory │ │      │
│   └──────────────┘    │  │ Service    │  │ Manager    │  │ Service│ │      │
│                       │  │ (main)     │  │ (service)  │  │(service)│ │      │
│                       │  └────────────┘  └────────────┘  └────────┘ │      │
│                       └──────────────────────────────────────────────┘      │
│                                                                              │
│   ┌──────────────┐   ┌──────────────┐   ┌────────────────────────────┐     │
│   │ Firestore    │   │ Vertex AI    │   │ Cloud Workflows            │     │
│   │ (Memory +    │   │ Vector Search│   │ (Auto-Correction +        │     │
│   │  Sessions +  │   │ / AlloyDB AI │   │  Async Pipelines)         │     │
│   │  Vectors)    │   │              │   │                           │     │
│   └──────────────┘   └──────────────┘   └────────────────────────────┘     │
│                                                                              │
│   ┌──────────────┐   ┌──────────────┐   ┌────────────────────────────┐     │
│   │ Eventarc     │   │ Pub/Sub      │   │ Cloud Storage              │     │
│   │ (Event       │   │ (Extraction  │   │ (Documents + Logs +        │     │
│   │  routing)    │   │  queue)      │   │  Corrections)             │     │
│   └──────────────┘   └──────────────┘   └────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why Cloud Run:**
- Auto-scaling (including to zero) with request-based billing
- GPU support for local embedding/inference
- WebSocket streaming for real-time responses
- Up to 60-minute request timeouts
- Traffic splitting for canary deployments
- Continuous deployment from Git

**When to use Cloud Functions (2nd Gen):**
- Event-driven triggers (document uploads, Firestore changes via Eventarc)
- Short-lived compute (< 60 minutes)
- Up to 16 GiB RAM, 4 vCPU, GPU configuration
- Concurrency: up to 1,000 concurrent requests per instance

### Deployment Decision Matrix

| Component | Deployment | Rationale |
|-----------|-----------|-----------|
| API Layer | Cloud Endpoints + Cloud Armor | DDoS protection, rate limiting, auth |
| Core Pipeline | Cloud Run | Auto-scaling, GPU support, WebSocket streaming |
| Async Processing | Cloud Functions + Eventarc | Event-driven, cost-efficient |
| Memory Extraction | Cloud Functions + Pub/Sub | Bursty, latency-tolerant |
| Batch Evaluations | Cloud Workflows + Functions | Scheduled, parallelizable |
| Orchestration | Agent Platform (Agent Runtime) | Managed agent infrastructure |
| Heavy ML Workloads | GKE with GPU node pools | Maximum control, tensor parallelism |

---

## Data Flow Summary (GCP)

```
User Query
    │
    ├──► [Cloud Endpoints: Cloud Armor + Rate Limiting + IAM Auth]
    │         │
    │         ▼
    ├──► [Model Armor: Sanitize User Prompt]  ──── BLOCK ──► Error Response
    │         │
    │         PASS
    │         │
    ├──► [Memorystore Redis: Cache Lookup]  ──── HIT ──► Return Cached Response ──► User
    │         │
    │         MISS
    │         │
    ├──► [Cloud Run: Custom Router + countTokens] ──► Model Tier Decision
    │         │
    │    ┌────┴────────────────────────────────────┐
    │    │         PARALLEL RETRIEVAL               │
    │    │                                          │
    │    │  [RAG Engine] ──► Document Chunks        │
    │    │  [Neo4j + Gemini] ──► Entities           │
    │    │  [Firestore + Embeddings] ──► Memories   │
    │    │                                          │
    │    └────┬────────────────────────────────────┘
    │         │
    │         ▼
    │    [Cloud Run: Context Manager — Assemble + Trim + Summarize]
    │         │
    │         ▼
    │    [Vertex AI Gemini API: Routed Model + Context + Safety Settings]
    │         │
    │         ├──► [Memorystore Redis: Write response + metadata + TTL]
    │         │
    │         ▼
    │    [Model Armor: Sanitize Model Response]  ──── BLOCK ──► Regenerate / Error
    │         │
    │         PASS
    │         │
    │    [Cloud Workflows: Auto-Correction Pipeline]
    │         │         │
    │         │    score < threshold ──► Cloud Storage (fine-tuning dataset)
    │         │
    │         ▼
    │    Final Response ──► User
    │
    └──► [Cloud Trace + Monitoring: Metrics + Traces across all steps]
    └──► [Gen AI Evaluation: Quality scoring (batch + scheduled)]
    │         └──► [Vertex AI Batch Prediction: Submit bulk eval queries at 50% cost]
    └──► [Cloud Logging: Full I/O to BigQuery for analysis]
```

---

## GCP Architecture Principles Applied

| Principle | Pipeline Implementation |
|-----------|----------------------|
| Defense in depth | Model Armor + Sensitive Data Protection + Safety Settings (layered) |
| Least privilege | Service accounts with minimal IAM roles, Workload Identity Federation |
| Serverless-first | Cloud Run scales to zero, Cloud Functions for events, Workflows for orchestration |
| Cost optimization | Custom router for model tier selection, Context Caching, Flash for internal tasks |
| Observable by default | OpenTelemetry → Cloud Trace, structured logging → BigQuery, custom metrics |
| Data sovereignty | Regional deployments, VPC Service Controls for data residency |

---

## Technology Stack (GCP)

| Layer | GCP Service |
|-------|------------|
| API Gateway | Cloud Endpoints + Cloud Armor |
| Compute | Cloud Run + Cloud Functions (2nd Gen) |
| LLM Provider | Vertex AI / Gemini API (Gen AI SDK) |
| Guardrails | Model Armor + Sensitive Data Protection |
| Routing | Custom Cloud Run service (no native router) |
| RAG | Vertex AI RAG Engine + Vertex AI Search |
| Vector Store | Firestore (vector) / AlloyDB AI / Vertex AI Vector Search |
| Graph Store | Neo4j Aura (GCP Marketplace) |
| Caching | Memorystore for Redis + Vertex AI Context Caching |
| Batch Processing | Vertex AI Batch Prediction + Cloud Workflows |
| Document Store | Firestore |
| Object Storage | Cloud Storage |
| Orchestration | Agent Platform + Cloud Workflows |
| Event Bus | Eventarc |
| Queue | Pub/Sub |
| Observability | Cloud Monitoring + Cloud Trace (OpenTelemetry) |
| Logging | Cloud Logging + BigQuery (analysis) |
| Evaluation | Vertex AI Gen AI Evaluation Service |
| Fine-Tuning | Vertex AI Supervised Tuning |
| Explainability | Grounding with Citations + Custom Eval Metrics |
| IaC | Terraform (google + google-beta providers) |
| CI/CD | Cloud Build + Cloud Deploy |

---

## Cost Optimization Strategies

| Strategy | Mechanism | Estimated Savings |
|----------|-----------|-------------------|
| Custom Model Router | Route simple queries to Gemini Flash | Up to 50% (Flash is 10-20x cheaper than Pro) |
| Context Caching | Cache repeated context prefixes via GenAiCacheService | Up to 75% on cached tokens |
| Provisioned Throughput | Reserved capacity for predictable workloads | Negotiated pricing |
| Batch Prediction | Non-real-time workloads (evaluations, corrections) | Up to 50% |
| Context Trimming | Reduce input tokens via summarization with Flash | Variable (fewer input tokens) |
| Cloud Run Scale-to-Zero | No compute cost when idle | Pay only for requests |
| Firestore Serverless | No idle cost for memory storage | Pay only for reads/writes |
| Gemini 3.5 Flash for Internal | Use cheapest model for summarization/extraction | 20-50x cheaper than Pro |

---

## Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Edge: Cloud Armor + Cloud CDN                            │    │
│  │  • DDoS protection (adaptive, ML-based)                  │    │
│  │  • Rate limiting (per-client, per-region)                │    │
│  │  • WAF rules (OWASP top 10, custom)                     │    │
│  │  • Bot management                                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Identity: IAM + Workload Identity Federation             │    │
│  │  • Service accounts with minimal roles                   │    │
│  │  • Workload Identity (no service account keys)           │    │
│  │  • IAM Conditions for fine-grained access                │    │
│  │  • Organization policies for guardrails                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Network: VPC Service Controls + Private Google Access    │    │
│  │  • VPC-SC perimeters around AI services                  │    │
│  │  • Private Google Access (no public internet)            │    │
│  │  • Cloud NAT for outbound (controlled egress)            │    │
│  │  • Private Service Connect for Vertex AI                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Data: Encryption + Access Control                        │    │
│  │  • Customer-managed encryption keys (CMEK via Cloud KMS) │    │
│  │  • TLS 1.3 in transit (enforced)                         │    │
│  │  • Secret Manager for API keys + automatic rotation      │    │
│  │  • Sensitive Data Protection for PII scanning            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Audit: Logging + Compliance                              │    │
│  │  • Cloud Audit Logs (Admin + Data Access)                │    │
│  │  • Access Transparency (Google staff access visibility)  │    │
│  │  • Security Command Center (threat detection)            │    │
│  │  • Organization Policy constraints                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Getting Started

### Prerequisites
- GCP project with Vertex AI API enabled
- gcloud CLI installed
- Python 3.11+

### Quick Start with Terraform

```bash
# Authenticate
gcloud auth application-default login

# Initialize Terraform
terraform init

# Deploy the pipeline stack
terraform apply -var="project_id=YOUR_PROJECT" -var="region=us-central1"
```

### Key Terraform Resources

```hcl
resource "google_vertex_ai_index" "vector_index" {
  display_name = "genai-pipeline-vectors"
  region       = var.region

  metadata {
    contents_delta_uri = "gs://${google_storage_bucket.vectors.name}/index"
    config {
      dimensions                  = 768
      approximate_neighbors_count = 150
      distance_measure_type       = "COSINE_DISTANCE"
      algorithm_config {
        tree_ah_config {
          leaf_node_embedding_count    = 500
          leaf_nodes_to_search_percent = 7
        }
      }
    }
  }
}

resource "google_cloud_run_v2_service" "pipeline" {
  name     = "genai-pipeline"
  location = var.region

  template {
    containers {
      image = "gcr.io/${var.project_id}/genai-pipeline:latest"
      resources {
        limits = { cpu = "4", memory = "8Gi" }
      }
    }
    scaling {
      min_instance_count = 0
      max_instance_count = 100
    }
  }
}

resource "google_firestore_database" "memory" {
  project     = var.project_id
  name        = "genai-memory"
  location_id = var.region
  type        = "FIRESTORE_NATIVE"
}
```

---

## References

- [Vertex AI Documentation](https://cloud.google.com/vertex-ai/docs)
- [Gemini API Documentation](https://ai.google.dev/gemini-api/docs)
- [Model Armor Overview](https://cloud.google.com/model-armor/docs/overview)
- [Sensitive Data Protection](https://cloud.google.com/sensitive-data-protection/docs)
- [Vertex AI RAG Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-overview)
- [Vertex AI Search](https://cloud.google.com/enterprise-search)
- [Agent Development Kit (ADK)](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-development-kit)
- [Vertex AI Gen AI Evaluation](https://cloud.google.com/vertex-ai/generative-ai/docs/models/evaluation-overview)
- [Cloud Workflows](https://cloud.google.com/workflows/docs)
- [Cloud Run](https://cloud.google.com/run/docs)
- [Cloud Trace + OpenTelemetry](https://cloud.google.com/trace/docs)
- [Neo4j on Google Cloud](https://neo4j.com/cloud/google-cloud/)

---

## License

MIT
