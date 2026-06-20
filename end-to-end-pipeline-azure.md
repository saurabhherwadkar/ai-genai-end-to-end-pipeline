# AI GenAI End-to-End Pipeline — Azure Implementation

A production-grade implementation of the GenAI LLM application pipeline using Azure managed services. This document maps each pipeline component to its Azure equivalent, following the [Azure Well-Architected Framework for AI Workloads](https://learn.microsoft.com/en-us/azure/well-architected/ai/get-started) which defines five architectural pillars: reliability, security, cost optimization, operational excellence, and performance efficiency.

---

## Azure Service Mapping

| # | Pipeline Component | Azure Service(s) |
|---|-------------------|----------------|
| 1 | Input/Output Guardrails | Azure AI Content Safety + Azure OpenAI Content Filtering + Prompt Shields |
| 2 | LLM Router | Microsoft Foundry Model Router (Responses API) |
| 3 | Context Manager | Azure Container Apps / Azure Functions |
| 4 | Context Memory | Azure Cosmos DB (NoSQL + DiskANN vectors) + Azure AI Search |
| 5 | RAG Pipeline | Azure AI Search + Microsoft Foundry IQ + Agentic Retrieval |
| 6 | GraphRAG | Microsoft GraphRAG (open-source) + Azure Cosmos DB Gremlin API |
| 7 | Auto-Correction | Azure Durable Functions + Azure OpenAI + Azure Blob Storage |
| 8 | Observability | Azure Monitor + Application Insights + Foundry Agent Monitoring |
| 9 | Evaluations | Microsoft Foundry Evaluations |
| 10 | Explainability | Azure ML Responsible AI + Foundry Observability Tracing |
| 11 | LLM Caching | Azure API Management Semantic Cache + Azure Cache for Redis |
| 12 | LLM Batching | Azure OpenAI Batch API + Azure Durable Functions |
| 13 | Orchestration | Foundry Agent Service + Microsoft Agent Framework |
| 14 | Infrastructure | API Management + Container Apps + Functions + Event Grid + Service Bus |

---

## End-to-End Architecture Diagram (Azure)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    USER / CLIENT APP                                     │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           AZURE API MANAGEMENT (GenAI Gateway)                            │
│                    ┌──────────────────────────────────────────────────┐                   │
│                    │  WAF + Token Rate Limiting + Semantic Cache +    │                   │
│                    │  Load Balancing + Content Safety Policy          │                   │
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
│   │                    AZURE AI CONTENT SAFETY (Input)                             │     │
│   │                    [Standalone API — no model invocation needed]               │     │
│   │                                                                               │     │
│   │   ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│   │   │ Sensitive     │  │ Content Filters   │  │   Denied    │  │  Prompt    │  │     │
│   │   │ Information   │  │ (Hate, Sexual,    │  │   Topics    │  │  Shields   │  │     │
│   │   │ (PII)        │  │  Violence, Self-  │  │  (Custom)   │  │ (Injection │  │     │
│   │   │              │  │  Harm, Prompt     │  │             │  │  Detection)│  │     │
│   │   │              │  │  Attack)          │  │             │  │            │  │     │
│   │   └──────┬───────┘  └─────────┬─────────┘  └──────┬──────┘  └─────┬──────┘  │     │
│   │          └─────────────────────┴───────────────────┴───────────────┘         │     │
│   │                                    │                                          │     │
│   │                    ┌───────────────┴────────────────────┐                    │     │
│   │                    │ Groundedness Detection              │                    │     │
│   │                    │ (hallucination + relevance)         │                    │     │
│   │                    └───────────────┬────────────────────┘                    │     │
│   │                                    │                                          │     │
│   │                          ┌─────────┴─────────┐                               │     │
│   │                          │  BLOCK or PASS    │                               │     │
│   │                          └─────────┬─────────┘                               │     │
│   └────────────────────────────────────┼──────────────────────────────────────────┘     │
│                                        │                                                 │
│                                        ▼                                                 │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │              LLM CACHE (APIM Semantic Cache + Azure Cache for Redis)           │     │
│   │                                                                               │     │
│   │   ┌──────────────────┐       ┌──────────────────────────────────────────┐    │     │
│   │   │  Incoming Query  │──────▶│           CACHE LOOKUP                   │    │     │
│   │   └──────────────────┘       │                                          │    │     │
│   │                              │  Strategy A: Exact Match (Redis hash)     │    │     │
│   │                              │  Strategy B: APIM Semantic Cache          │    │     │
│   │                              │    (embedding similarity ≥ 0.95)          │    │     │
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
│   │              MICROSOFT FOUNDRY MODEL ROUTER                                    │     │
│   │              [Automatic model selection via Responses API]                     │     │
│   │                                                                               │     │
│   │   ┌──────────────────────────────────────────────────────────────────────┐   │     │
│   │   │  Analyzes prompt complexity → Routes to optimal model tier           │   │     │
│   │   │  model="model-router" in Responses API                               │   │     │
│   │   └──────────────────────────────────────────────────────────────────────┘   │     │
│   │                                                                               │     │
│   │              ┌─────────────────────┼─────────────────────┐                   │     │
│   │              ▼                     ▼                     ▼                   │     │
│   │      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐             │     │
│   │      │ GPT-4.1-mini│      │  GPT-4.1    │      │  GPT-5.5    │             │     │
│   │      │ / GPT-5.4-  │      │ / GPT-5.4   │      │ / o3-pro    │             │     │
│   │      │  mini       │      │             │      │             │             │     │
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
│   │  AZURE AI SEARCH     │  │  GRAPHRAG            │  │   CONTEXT MEMORY         │     │
│   │  (RAG Pipeline)      │  │  (Cosmos DB Gremlin  │  │   (Cosmos DB NoSQL +     │     │
│   │                      │  │   + MS GraphRAG)     │  │    AI Search Vectors)    │     │
│   │  Data Sources:       │  │                      │  │                          │     │
│   │  • Azure Blob Storage│  │  Graph Store:        │  │  Query                   │     │
│   │  • SharePoint Online │  │  • Cosmos DB Gremlin │  │    │                     │     │
│   │  • SQL Database      │  │  • Apache TinkerPop  │  │    ▼                     │     │
│   │  • OneLake           │  │                      │  │  Semantic Similarity     │     │
│   │                      │  │  Features:           │  │  Search (AI Search       │     │
│   │  Vector Stores:      │  │  • LLM entity        │  │   vector engine)         │     │
│   │  • AI Search         │  │    extraction        │  │    │                     │     │
│   │    (built-in)        │  │  • Community          │  │    ▼                     │     │
│   │  • Cosmos DB NoSQL   │  │    detection          │  │  Cosmos DB (node/edge   │     │
│   │    (DiskANN)         │  │  • Global/local       │  │   persistence, TTL)     │     │
│   │  • Cosmos DB         │  │    summarization      │  │    │                     │     │
│   │    PostgreSQL        │  │  • Multi-hop          │  │    ▼                     │     │
│   │                      │  │    reasoning          │  │  Ranked Memories         │     │
│   │  Retrieval:          │  │                      │  │  (thoughts, beliefs,     │     │
│   │  • Semantic search   │  │  Query:              │  │   preferences, goals)    │     │
│   │  • Hybrid (vector    │  │  • Entity matching   │  │                          │     │
│   │    + keyword)        │  │  • Subgraph extract  │  │  Async Extraction:       │     │
│   │  • Semantic ranking  │  │  • Community context │  │  • Event Grid trigger    │     │
│   │  • Agentic retrieval │  │                      │  │  • Functions + OpenAI    │     │
│   │  • Metadata filters  │  │                      │  │  • Cosmos DB Change Feed │     │
│   └──────────┬───────────┘  └──────────┬───────────┘  └────────────┬─────────────┘     │
│              │                          │                            │                   │
│              └──────────────────────────┼────────────────────────────┘                   │
│                                         │                                                │
│                                         ▼                                                │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │             CONTEXT MANAGER (Azure Container Apps / Functions)                  │     │
│   │                                                                               │     │
│   │  ┌─────────────────┐  ┌───────────────────┐  ┌────────────────────────────┐  │     │
│   │  │  Token Counter  │  │ Trimming Strategy │  │  Summarization via        │  │     │
│   │  │  (tiktoken /    │  │ (FIFO / Sliding   │  │  Azure OpenAI             │  │     │
│   │  │   OpenAI SDK)   │  │  Window / Priority)│  │  (GPT-4.1-mini — fast,   │  │     │
│   │  │                 │  │                    │  │   cheap summarization)    │  │     │
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
│   │                     AZURE OPENAI RESPONSES API                                 │     │
│   │                                                                               │     │
│   │         ┌───────────────────────────────────────────────────────┐            │     │
│   │         │              MODEL INVOCATION                          │            │     │
│   │         │                                                       │            │     │
│   │         │  • Unified API across all Azure OpenAI models         │            │     │
│   │         │  • Streaming via Server-Sent Events                   │            │     │
│   │         │  • Built-in retry with exponential backoff            │            │     │
│   │         │  • Content filters applied inline                     │            │     │
│   │         │  • Managed Identity authentication (keyless)          │            │     │
│   │         │                                                       │            │     │
│   │         │  Models Available:                                     │            │     │
│   │         │  • GPT-5.5 / GPT-5.4 / GPT-5.4-mini (1M+ context)   │            │     │
│   │         │  • GPT-4.1 / GPT-4.1-mini (1M context)              │            │     │
│   │         │  • o3-pro / o4-mini / o3 (reasoning)                  │            │     │
│   │         │  • text-embedding-3-large (3072 dims)                 │            │     │
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
│   │                 AZURE AI CONTENT SAFETY (Output)                               │     │
│   │                 [Same content filter applied to model responses]               │     │
│   │                                                                               │     │
│   │   ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│   │   │ PII Redactor │  │  Groundedness     │  │  Content    │  │  Protected │  │     │
│   │   │ (Sensitive   │  │  Detection        │  │  Filter     │  │  Material  │  │     │
│   │   │  Info Filter)│  │  (Hallucination)  │  │  (Toxicity) │  │  Detection │  │     │
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
│   │              AUTO-CORRECTION PIPELINE (Azure Durable Functions)                │     │
│   │                                                                               │     │
│   │   ┌──────────────────┐       ┌────────────────────────────────────────┐      │     │
│   │   │  Actual Response │──────▶│           COMPARATOR                   │      │     │
│   │   └──────────────────┘       │           (Activity Function)          │      │     │
│   │                              │                                        │      │     │
│   │                              │  Strategy A: Azure OpenAI Embeddings   │      │     │
│   │                              │    (text-embedding-3 — cosine sim)     │      │     │
│   │   ┌──────────────────┐       │  Strategy B: Azure OpenAI LLM-Judge   │      │     │
│   │   │  Ideal Response  │──────▶│    (GPT-4.1 evaluation)               │      │     │
│   │   │  (Blob Storage   │       │                                        │      │     │
│   │   │   ground truth)  │       │  Output: Confidence Score (0.0 - 1.0)  │      │     │
│   │   └──────────────────┘       └───────────────────┬────────────────────┘      │     │
│   │                                                   │                           │     │
│   │                                ┌──────────────────┴──────────────────┐        │     │
│   │                                │                                     │        │     │
│   │                          score >= 0.8                          score < 0.8    │     │
│   │                                │                                     │        │     │
│   │                                ▼                                     ▼        │     │
│   │                          ┌──────────┐                  ┌─────────────────┐   │     │
│   │                          │   PASS   │                  │ Capture to Blob │   │     │
│   │                          └──────────┘                  │ for fine-tuning │   │     │
│   │                                                        │ (Azure ML)      │   │     │
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
│   │  Azure Monitor:        │ │  Foundry Evaluations:  │ │  Responsible AI Dashboard: │  │
│   │  • Custom Metrics      │ │  • Agent Evaluators    │ │  • Fairness assessment     │  │
│   │    (TTFT, latency,     │ │    (task adherence,    │ │  • Feature attribution     │  │
│   │     token usage)       │ │     tool correctness)  │ │  • Error analysis          │  │
│   │  • Log Analytics       │ │  • Quality Evaluators  │ │                            │  │
│   │  • Dashboards          │ │    (coherence,         │ │  Foundry Tracing:          │  │
│   │  • Alerts + Action     │ │     relevance,         │ │  • Full reasoning chain    │  │
│   │    Groups              │ │     fluency)           │ │  • Tool invocation logs    │  │
│   │                        │ │  • Safety Evaluators   │ │  • Token attribution       │  │
│   │  Application Insights: │ │    (violence, hate,    │ │    via prompts             │  │
│   │  • Distributed tracing │ │     sexual, self-harm) │ │                            │  │
│   │  • OpenTelemetry       │ │  • Text Similarity     │ │  Invocation Logging:       │  │
│   │  • Live Metrics        │ │    (BLEU, ROUGE,       │ │  • Full request/response   │  │
│   │                        │ │     METEOR)            │ │    capture to Blob         │  │
│   │  Foundry Monitoring:   │ │                        │ │  • Prompt analysis via     │  │
│   │  • Token consumption   │ │  Red Team Scans:       │ │    Log Analytics           │  │
│   │  • Per-agent metrics   │ │  • Adversarial testing │ │                            │  │
│   │  • TTFB / TBT latency │ │  • Jailbreak probes    │ │                            │  │
│   │  • Success rates       │ │                        │ │                            │  │
│   │                        │ │                        │ │                            │  │
│   │  Cost Management:      │ │                        │ │                            │  │
│   │  • Per-deployment      │ │                        │ │                            │  │
│   │    token tracking      │ │                        │ │                            │  │
│   │  • Budget alerts       │ │                        │ │                            │  │
│   └────────────────────────┘ └────────────┬───────────┘ └────────────────────────────┘  │
│                                           │                                              │
│                                           ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                  LLM BATCHING (Azure OpenAI Batch API)                            │   │
│   │                                                                                  │   │
│   │   ┌────────────────────────────────────────────────────────────────────────┐    │   │
│   │   │                      BATCH JOB MANAGER                                 │    │   │
│   │   │                                                                        │    │   │
│   │   │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │    │   │
│   │   │  │  Upload JSONL    │  │  Batch Job       │  │  Download Results   │  │    │   │
│   │   │  │  (collect eval   │  │  (/batches API   │  │  (map responses     │  │    │   │
│   │   │  │   queries)       │  │   with 24h       │  │   back to queries)  │  │    │   │
│   │   │  │                  │  │   completion)    │  │                     │  │    │   │
│   │   │  └────────┬─────────┘  └────────┬─────────┘  └──────────┬──────────┘  │    │   │
│   │   │           └─────────────────────┼────────────────────────┘             │    │   │
│   │   │                                 │                                      │    │   │
│   │   │              Up to 50% cost savings vs real-time inference             │    │   │
│   │   └────────────────────────────────────────────────────────────────────────┘    │   │
│   │                                                                                  │   │
│   │   Use Cases:                                                                     │   │
│   │   • Batch evaluation scoring (submit 100s of eval queries at once)              │   │
│   │   • Offline quality assessments (nightly eval runs via Durable Functions)        │   │
│   │   • Fine-tuning data validation (bulk comparison against ground truth)          │   │
│   │   • A/B test evaluation (compare model versions across test suites)             │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Phases (Azure Implementation)

### Phase 1: Input Processing

#### Azure AI Content Safety (Input)

[Azure AI Content Safety](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/) provides configurable safeguards that can be applied to both input prompts and model responses:

| Safeguard Type | Function | Configuration |
|---------------|----------|---------------|
| Content Filters | Blocks hate, sexual content, violence, self-harm, misconduct | Severity 0-7, configurable threshold (LOW/MEDIUM/HIGH) |
| Prompt Shields | Detects user prompt attacks + indirect/document injection | Confidence levels: High, Medium+, Low+ |
| Denied Topics | Rejects queries on restricted subjects | Custom topic definitions with examples |
| Word Filters | Blocks specific words/phrases | Custom word lists + managed profanity list |
| Sensitive Information Filters | Detects/redacts PII (SSN, email, phone, names) | Probabilistic detection + custom regex |
| Groundedness Detection | Verifies factual grounding against provided context | Threshold-based scoring, supports reasoning mode |
| Protected Material Detection | Detects copyrighted text and GitHub code | Always-on for protected content |
| Task Adherence | Detects agent misalignment with instructions | Preview — monitors agent behavior drift |

**Key capability:** Content Safety APIs run independently of model invocation — guardrails can be applied as a standalone preprocessing step before any LLM call.

```python
from azure.ai.contentsafety import ContentSafetyClient
from azure.identity import DefaultAzureCredential

client = ContentSafetyClient(
    endpoint="https://YOUR-RESOURCE.cognitiveservices.azure.com",
    credential=DefaultAzureCredential()
)

response = client.analyze_text(
    text=user_input,
    categories=["Hate", "Sexual", "Violence", "SelfHarm"],
    output_type="EightSeverityLevels"
)

if any(cat.severity >= 4 for cat in response.categories_analysis):
    return {"blocked": True, "reason": "Content policy violation"}
```

#### Microsoft Foundry Model Router

The [Foundry Model Router](https://learn.microsoft.com/en-us/azure/ai-services/openai/) dynamically routes each request to the model predicted most likely to give the desired response at the lowest cost:

- Analyzes prompt complexity automatically (no custom signals needed)
- Routes across the configured model pool (GPT-4.1-mini through GPT-5.5)
- Includes automatic failover, prompt caching, and rate limiting
- No manual threshold tuning — Foundry manages the routing logic

```python
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default"
)

client = AzureOpenAI(
    azure_endpoint="https://YOUR-RESOURCE.openai.azure.com",
    azure_ad_token_provider=token_provider,
    api_version="2025-12-01-preview"
)

response = client.responses.create(
    model="model-router",
    input="What is the capital of France?"
)
```

#### LLM Cache (APIM Semantic Cache + Azure Cache for Redis)

Two caching layers reduce cost and latency:

**Layer 1: API Management Semantic Cache (built-in)**
- Semantic similarity matching on incoming requests (embedding-based)
- Configurable similarity threshold (recommended ≥ 0.95)
- TTL-based expiry per cache entry
- No custom infrastructure — configured as an APIM policy
- On **CACHE HIT**: returns cached response at the gateway level (no backend call)

**Layer 2: Application-Level Cache (Azure Cache for Redis)**
- Exact match: Hash-based key lookup for identical queries (sub-millisecond)
- Semantic match: Embedding similarity via Azure OpenAI text-embedding-3
- TTL management per query type
- On **CACHE MISS**: continues to Router; after LLM response, writes to cache

```python
from azure.identity import DefaultAzureCredential
import redis
import hashlib
import json

cache = redis.Redis(
    host="your-cache.redis.cache.windows.net",
    port=6380,
    ssl=True,
    password=get_redis_key()
)

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

---

### Phase 2: Context Assembly

#### Azure AI Search (RAG)

[Azure AI Search](https://learn.microsoft.com/en-us/azure/search/) provides fully managed retrieval with vector, keyword, and hybrid search:

**Data Sources:**
- Azure Blob Storage (PDF, DOCX, TXT, HTML, MD, CSV)
- SharePoint Online
- Azure SQL Database / Cosmos DB
- OneLake (Microsoft Fabric)
- Custom connectors via push API

**Vector Capabilities:**
- Built-in vectorizers (Azure OpenAI, custom model endpoints)
- Hybrid retrieval (vector + BM25 keyword search)
- Semantic ranking (neural reranker, L2 model)
- Agentic retrieval (MCP-based query planning and decomposition)
- Metadata filtering for precision
- Source attribution in responses

**Agentic Retrieval (Preview):**
- Exposed via Model Context Protocol (MCP) endpoints
- Automatic query decomposition for complex questions
- Multi-index federation
- Integrated with Foundry Agent Service

```python
from azure.search.documents import SearchClient
from azure.identity import DefaultAzureCredential

search_client = SearchClient(
    endpoint="https://YOUR-SERVICE.search.windows.net",
    index_name="documents",
    credential=DefaultAzureCredential()
)

results = search_client.search(
    search_text="refund policy",
    vector_queries=[{
        "kind": "text",
        "text": "What is our refund policy?",
        "fields": "content_vector",
        "k_nearest_neighbors": 10
    }],
    query_type="semantic",
    semantic_configuration_name="default",
    top=5
)
```

#### GraphRAG (Microsoft GraphRAG + Cosmos DB Gremlin)

[Microsoft GraphRAG](https://github.com/microsoft/graphrag) is an open-source toolkit for knowledge-graph-enhanced retrieval, deployable on Azure:

**Graph Store:** Azure Cosmos DB Gremlin API (Apache TinkerPop)

**Capabilities:**
- LLM-driven entity and relationship extraction from documents
- Community detection via Leiden algorithm
- Global summarization (theme-level answers across entire corpus)
- Local search (entity-centric, multi-hop traversal)
- Drift search (combining local entity context with community insights)

**Architecture:**
```
Documents (Blob Storage) → Azure OpenAI (Entity Extraction)
                                    │
                                    ▼
                          Cosmos DB Gremlin (Graph)
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Entity Match    Graph Traversal  Community Context
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
                          Enriched Context for LLM
```

**Cosmos DB Gremlin Features:**
- Fully managed, globally distributed graph database
- Elastic scalability (billions of vertices/edges)
- Automatic indexing on all properties
- Five tunable consistency levels
- 99.999% SLA availability

#### Context Memory (Cosmos DB + AI Search)

Azure provides a hybrid approach using multiple services:

| Layer | Azure Service | Purpose |
|-------|-------------|---------|
| Node/Edge Persistence | Azure Cosmos DB (NoSQL API) | Memory nodes, edges, metadata with TTL |
| Vector Search | Azure Cosmos DB (DiskANN) or Azure AI Search | Semantic similarity via k-NN / ANN |
| Embedding Generation | Azure OpenAI (text-embedding-3-large) | Convert memory content to vectors |
| Async Extraction | Azure Event Grid + Functions | Fire-and-forget memory extraction |
| Graph Queries | Cosmos DB Change Feed + Functions | BFS traversal via partition key design |

**Cosmos DB Vector Index Types:**
- **Flat:** Exact brute-force, max 505 dimensions
- **QuantizedFlat:** DiskANN quantization, max 4096 dimensions (datasets <= 50K vectors)
- **DiskANN:** Best for large datasets (>50K vectors), lowest latency at scale

```python
from azure.cosmos import CosmosClient
from azure.identity import DefaultAzureCredential

client = CosmosClient(
    url="https://YOUR-ACCOUNT.documents.azure.com",
    credential=DefaultAzureCredential()
)

container = client.get_database_client("memory").get_container_client("nodes")

results = container.query_items(
    query="""
        SELECT TOP 10 c.id, c.content, c.type,
               VectorDistance(c.embedding, @query_vector) AS score
        FROM c
        WHERE c.type IN ('thought', 'belief', 'preference')
        ORDER BY VectorDistance(c.embedding, @query_vector)
    """,
    parameters=[{"name": "@query_vector", "value": query_embedding}],
    enable_cross_partition_query=True
)
```

#### Context Manager (Container Apps / Functions)

Token counting and context assembly runs as an Azure Container App or Function:

- **Token counting:** tiktoken library or Azure OpenAI SDK (returns `prompt_tokens` and `completion_tokens`)
- **Trimming strategies:** FIFO, sliding window, priority-based (implemented in application code)
- **Summarization:** Calls Azure OpenAI (GPT-4.1-mini — fast, cheap) to condense older conversation turns
- **Output:** Token-budgeted context window ready for the final LLM call

---

### Phase 3: LLM Execution

#### Azure OpenAI Responses API

The [Responses API](https://learn.microsoft.com/en-us/azure/ai-services/openai/) provides a unified interface across all Azure OpenAI models:

```python
response = client.responses.create(
    model="gpt-5.4",
    input=[
        {"role": "system", "content": system_prompt_with_memories},
        {"role": "user", "content": assembled_user_message}
    ],
    max_output_tokens=4096,
    temperature=0.7,
    stream=True
)

for event in response:
    if event.type == "response.output_text.delta":
        yield event.delta
```

**Key features:**
- Unified API across all GPT and reasoning models
- Native streaming via Server-Sent Events
- Inline content filtering (no separate call needed)
- Managed Identity authentication (keyless, no API keys)
- Built-in retry logic with exponential backoff (Azure SDK)
- Tool use / function calling support
- Structured output with JSON schema enforcement

---

### Phase 4: Output Processing

#### Azure AI Content Safety (Output)

The same content filters applied during input processing also validate model responses:

- **PII Redaction:** Sensitive Information Filters mask any PII that leaked into output
- **Hallucination Detection:** Groundedness Detection verifies output against provided context
- **Content Safety:** Content Filters catch toxic/harmful content in responses
- **Protected Material:** Detects copyrighted text (song lyrics, news articles) and code

When configured as an output filter on the Azure OpenAI deployment, these run automatically without a separate API call.

#### Auto-Correction Pipeline (Azure Durable Functions)

```
┌─────────────────────────────────────────────────────────────────┐
│                  AZURE DURABLE FUNCTIONS WORKFLOW                 │
│                                                                  │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────────┐    │
│  │ Start   │───▶│ Activity:    │───▶│ Activity:           │    │
│  │ (Orch.) │    │ Fetch Ground │    │ Compare (Cosine     │    │
│  │         │    │ Truth (Blob) │    │ Sim / LLM-Judge)    │    │
│  └─────────┘    └──────────────┘    └──────────┬──────────┘    │
│                                                 │               │
│                              ┌──────────────────┴────────┐      │
│                              │                           │      │
│                        score >= 0.8               score < 0.8   │
│                              │                           │      │
│                              ▼                           ▼      │
│                      ┌────────────┐         ┌────────────────┐  │
│                      │ Pass       │         │ Activity:      │  │
│                      │ (no-op)    │         │ Write to Blob  │  │
│                      └────────────┘         │ (correction    │  │
│                                             │  record)       │  │
│                                             └───────┬────────┘  │
│                                                     │           │
│                                                     ▼           │
│                                             ┌────────────────┐  │
│                                             │ Azure ML:      │  │
│                                             │ Fine-tuning    │  │
│                                             │ dataset (batch)│  │
│                                             └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Components:**
- **Ground truth store:** Azure Blob Storage (structured JSONL files)
- **Comparison strategies:**
  - Cosine similarity via Azure OpenAI text-embedding-3-large
  - LLM-as-Judge via Azure OpenAI (GPT-4.1 evaluates response quality)
- **Correction capture:** Blob Storage + Cosmos DB (query + actual + ideal + metadata)
- **Fine-tuning pipeline:** Azure Machine Learning for model customization

---

### Phase 5: Monitoring & Continuous Improvement

#### Observability

| Metric | Azure Service | Implementation |
|--------|------------|----------------|
| Time to First Token (TTFT) | Application Insights | Custom metric from streaming start |
| Total Latency | Application Insights | End-to-end distributed trace |
| Inter-Token Latency (TBT) | Foundry Agent Monitoring | Calculated from stream events |
| Token Usage | Foundry Monitoring Dashboard | Automatic per-deployment tracking |
| Cost per Request | Azure Cost Management | Per-deployment token attribution |
| Error Rates | Azure Monitor Alerts | Threshold-based alerting via Action Groups |
| Distributed Tracing | Application Insights (OpenTelemetry) | Service map + latency analysis |

**OpenTelemetry Integration:**
```python
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace

configure_azure_monitor(
    connection_string="InstrumentationKey=YOUR-KEY"
)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("llm_pipeline") as span:
    span.set_attribute("gen_ai.request.model", "gpt-5.4")
    span.set_attribute("gen_ai.usage.prompt_tokens", prompt_tokens)
    span.set_attribute("gen_ai.usage.completion_tokens", completion_tokens)
```

**Foundry Agent Monitoring** provides pre-built dashboards tracking:
- Token consumption (prompt + completion) per agent
- Latency analysis (Time to Response, Time to Last Byte, Time Between Tokens)
- Success/failure rates per agent step
- Tool invocation frequency and duration

#### Evaluations

**Microsoft Foundry Evaluations** (managed service):

| Evaluator Category | Metrics |
|-------------------|---------|
| Agent Evaluators | Task Adherence, Tool Use Correctness, Search Quality |
| Quality Evaluators | Coherence, Relevance, Fluency, Groundedness |
| Safety Evaluators | Violence, Hate/Fairness, Sexual, Self-Harm |
| Text Similarity | BLEU, ROUGE, METEOR, Semantic Similarity |

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

client = AIProjectClient(
    credential=DefaultAzureCredential(),
    endpoint="https://YOUR-PROJECT.ai.azure.com"
)

eval_run = client.evals.runs.create(
    eval_id="quality-eval",
    dataset_id="test-dataset-v1",
    evaluators=["coherence", "relevance", "groundedness", "fluency"],
    model="gpt-4.1"
)
```

**Continuous evaluation:** Foundry supports scheduled evaluations + continuous evaluation rules on sampled production responses + alert configuration for quality degradation.

#### LLM Batching (Azure OpenAI Batch API)

The [Azure OpenAI Batch API](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/batch) enables submitting large volumes of evaluation queries at reduced cost:

- Upload JSONL files containing hundreds/thousands of requests
- Batch jobs complete within 24 hours at up to 50% cost savings
- Results downloadable as JSONL, mapped back to original custom_ids
- Supports all Azure OpenAI models (GPT-4.1, GPT-5.4, etc.)

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://YOUR-RESOURCE.openai.azure.com",
    azure_ad_token_provider=token_provider,
    api_version="2025-12-01-preview"
)

# Upload batch input file
batch_file = client.files.create(
    file=open("eval-queries.jsonl", "rb"),
    purpose="batch"
)

# Create batch job
batch_job = client.batches.create(
    input_file_id=batch_file.id,
    endpoint="/chat/completions",
    completion_window="24h"
)

# Poll for completion
status = client.batches.retrieve(batch_job.id)
if status.status == "completed":
    results = client.files.content(status.output_file_id)
```

**Scheduling:** Use Azure Durable Functions with timer triggers for nightly/weekly batch evaluation runs.

#### Explainability

| Approach | Azure Service | Capability |
|----------|-------------|-----------|
| Feature Attribution | Azure ML Responsible AI | SHAP-based feature importance |
| Fairness Assessment | Azure ML Responsible AI | Group fairness across sensitive attributes |
| Error Analysis | Azure ML Responsible AI | Failure cohort identification |
| Chain-of-Thought Analysis | Foundry Tracing | Parse and score reasoning steps |
| Prompt Analysis | Log Analytics (KQL) | Query logged prompts/responses |
| Reasoning Chain | Foundry Agent Monitoring | Full tool/reasoning audit trail |

---

## Orchestration Services

### Microsoft Foundry Agent Service

[Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-services/) provides managed agent deployment:

**Agent Types:**
| Type | Purpose |
|------|---------|
| Prompt Agents | No-code, defined via portal/SDK — production agents without custom logic |
| Hosted Agents | Custom business logic — deploy your code with managed infrastructure |

**Hosted Agent Frameworks:**
- Microsoft Agent Framework (recommended)
- LangGraph (LangChain)
- OpenAI Agents SDK
- Anthropic Agent SDK
- GitHub Copilot SDK
- Custom code

**Protocol Support:** MCP (Model Context Protocol), A2A (Agent-to-Agent) endpoints

**Features:**
- Multi-agent collaboration (supervisor + specialized sub-agents)
- Built-in session and long-term memory
- Code interpreter (sandboxed execution)
- Function calling with automatic retry
- Content safety at every agent step

### Microsoft Agent Framework

[Microsoft Agent Framework](https://github.com/microsoft/agent-framework) (successor to Semantic Kernel) for custom orchestration:

```python
from agent_framework import Agent, Kernel
from agent_framework.connectors import AzureOpenAIChatCompletion

kernel = Kernel()
kernel.add_service(AzureOpenAIChatCompletion(
    deployment_name="gpt-5.4",
    endpoint="https://YOUR-RESOURCE.openai.azure.com",
    credential=DefaultAzureCredential()
))

agent = Agent(
    kernel=kernel,
    name="pipeline-orchestrator",
    instructions="Orchestrate the GenAI pipeline...",
    plugins=[guardrails_plugin, rag_plugin, memory_plugin]
)

response = await agent.invoke("Process this user query")
```

- Python 1.43+, .NET 10.0+, Java JDK 17+
- Supports: Azure OpenAI, OpenAI, Hugging Face, NVIDIA, Ollama, ONNX
- Multi-agent orchestration patterns: sequential, concurrent, group chat, handoff

### Azure API Management (GenAI Gateway)

[API Management](https://learn.microsoft.com/en-us/azure/api-management/) provides GenAI-specific policies:

| Policy | Purpose |
|--------|---------|
| Token Rate Limiting | TPM/RPM control per deployment/consumer |
| Token Metrics Emission | Track consumption for cost attribution |
| Content Safety Enforcement | Apply guardrails at gateway level |
| Semantic Caching | Cache similar requests to reduce cost |
| Load Balancing | Distribute across multiple model deployments |
| Circuit Breaker | Automatic failover on deployment errors |

---

## Infrastructure Architecture

### Recommended: Container-First with Serverless Events

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HYBRID ARCHITECTURE                                   │
│                                                                              │
│   ┌──────────────┐    ┌──────────────────────────────────────────────┐      │
│   │ API Mgmt     │───▶│ Azure Container Apps                          │      │
│   │ (GenAI       │    │                                              │      │
│   │  Gateway)    │    │  ┌────────────┐  ┌────────────┐  ┌────────┐ │      │
│   └──────────────┘    │  │ FastAPI    │  │ Context    │  │ Memory │ │      │
│                       │  │ Service    │  │ Manager    │  │ Service│ │      │
│                       │  │ (main)     │  │ (sidecar)  │  │(sidecar)│ │      │
│                       │  └────────────┘  └────────────┘  └────────┘ │      │
│                       └──────────────────────────────────────────────┘      │
│                                                                              │
│   ┌──────────────┐   ┌──────────────┐   ┌────────────────────────────┐     │
│   │ Cosmos DB    │   │ AI Search    │   │ Durable Functions          │     │
│   │ (Memory +    │   │ (Vectors +   │   │ (Auto-Correction +        │     │
│   │  Sessions +  │   │  Semantic    │   │  Async Workflows)         │     │
│   │  Vectors)    │   │  Ranking)    │   │                           │     │
│   └──────────────┘   └──────────────┘   └────────────────────────────┘     │
│                                                                              │
│   ┌──────────────┐   ┌──────────────┐   ┌────────────────────────────┐     │
│   │ Event Grid   │   │ Service Bus  │   │ Blob Storage               │     │
│   │ (Async       │   │ (Extraction  │   │ (Documents + Logs +        │     │
│   │  triggers)   │   │  queue)      │   │  Corrections)             │     │
│   └──────────────┘   └──────────────┘   └────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**When to use Container Apps:**
- GPU workloads (embedding models, local inference)
- Sustained throughput with auto-scaling
- WebSocket streaming for real-time responses
- DAPR integration for microservice patterns
- Scales to zero when idle

**When to use Azure Functions:**
- Event-driven triggers (document uploads, Cosmos DB change feed)
- Short-lived compute (< 10 minutes)
- Durable Functions for stateful multi-step workflows
- Cost optimization for sporadic workloads

### Deployment Decision Matrix

| Component | Deployment | Rationale |
|-----------|-----------|-----------|
| API Layer | API Management | GenAI gateway policies, token rate limiting |
| Core Pipeline | Container Apps | Sustained throughput, GPU support, WebSocket |
| Async Processing | Functions + Event Grid | Event-driven, cost-efficient |
| Memory Extraction | Functions + Service Bus | Bursty, latency-tolerant |
| Batch Evaluations | Durable Functions | Scheduled, parallelizable |
| Orchestration | Foundry Agent Service | Managed agent infrastructure |

---

## Data Flow Summary (Azure)

```
User Query
    │
    ├──► [API Management: WAF + Token Limits + Semantic Cache]
    │         │
    │         ▼
    ├──► [Content Safety: Analyze Text API]  ──── BLOCK ──► Error Response
    │         │
    │         PASS
    │         │
    ├──► [APIM Semantic Cache + Redis: Lookup]  ──── HIT ──► Return Cached Response ──► User
    │         │
    │         MISS
    │         │
    ├──► [Foundry Model Router: model="model-router"] ──► Model Tier Decision
    │         │
    │    ┌────┴────────────────────────────────────┐
    │    │         PARALLEL RETRIEVAL               │
    │    │                                          │
    │    │  [AI Search] ──► Document Chunks         │
    │    │  [GraphRAG + Cosmos Gremlin] ──► Entities│
    │    │  [Cosmos DB + AI Search] ──► Memories     │
    │    │                                          │
    │    └────┬────────────────────────────────────┘
    │         │
    │         ▼
    │    [Container App: Context Manager — Assemble + Trim + Summarize]
    │         │
    │         ▼
    │    [Azure OpenAI Responses API: Routed Model + Context + Inline Filters]
    │         │
    │         ├──► [Redis Cache: Write response + metadata + TTL]
    │         │
    │         ▼
    │    [Content Safety: Output Filters]  ──── BLOCK ──► Regenerate / Error
    │         │
    │         PASS
    │         │
    │    [Durable Functions: Auto-Correction Workflow]
    │         │         │
    │         │    score < threshold ──► Blob Storage (fine-tuning dataset)
    │         │
    │         ▼
    │    Final Response ──► User
    │
    └──► [Application Insights + Monitor: Metrics + Traces across all steps]
    └──► [Foundry Evaluations: Quality scoring (scheduled + continuous)]
    │         └──► [Azure OpenAI Batch API: Submit bulk eval queries at 50% cost]
    └──► [Foundry Agent Monitoring: Per-agent observability]
```

---

## Azure Well-Architected Framework — AI Design Principles Applied

| Principle | Pipeline Implementation |
|-----------|----------------------|
| Design for reliability | Multi-region deployments, API Management circuit breaker, Cosmos DB global distribution |
| Implement defense in depth | Content Safety at input + output, API Management content policy, Prompt Shields |
| Secure by default | Managed Identity (keyless), Private Endpoints, Key Vault for secrets |
| Optimize for cost | Model Router for tier selection, semantic caching, Container Apps scale-to-zero |
| Achieve operational excellence | Foundry Agent Monitoring, Application Insights tracing, continuous evaluations |
| Design for performance | AI Search semantic ranking, Cosmos DB DiskANN vectors, streaming responses |

---

## Technology Stack (Azure)

| Layer | Azure Service |
|-------|------------|
| API Gateway | Azure API Management (GenAI policies) |
| Compute | Azure Container Apps + Azure Functions |
| LLM Provider | Azure OpenAI Service (Responses API) |
| Guardrails | Azure AI Content Safety + Prompt Shields |
| Routing | Microsoft Foundry Model Router |
| RAG | Azure AI Search (hybrid + semantic ranking) |
| Vector Store | Azure Cosmos DB (DiskANN) + Azure AI Search |
| Graph Store | Azure Cosmos DB Gremlin API |
| Caching | Azure API Management Semantic Cache + Azure Cache for Redis |
| Batch Processing | Azure OpenAI Batch API + Durable Functions |
| Document Store | Azure Cosmos DB (NoSQL API) |
| Object Storage | Azure Blob Storage |
| Orchestration | Foundry Agent Service + Durable Functions |
| Event Bus | Azure Event Grid |
| Queue | Azure Service Bus |
| Observability | Azure Monitor + Application Insights (OpenTelemetry) |
| Monitoring | Foundry Agent Monitoring Dashboard |
| Evaluation | Microsoft Foundry Evaluations |
| Fine-Tuning | Azure Machine Learning |
| Explainability | Azure ML Responsible AI Dashboard |
| IaC | Bicep + Terraform (AzureRM provider) |
| CI/CD | Azure DevOps Pipelines / GitHub Actions |

---

## Cost Optimization Strategies

| Strategy | Mechanism | Estimated Savings |
|----------|-----------|-------------------|
| Model Router | Route simple queries to cheaper models automatically | Up to 30% |
| Semantic Caching (APIM) | Cache semantically similar requests | Variable (30-70% for repetitive workloads) |
| Provisioned Throughput | Reserved PTUs for predictable workloads | Up to 50% vs pay-per-token |
| Batch API | Non-real-time workloads (evaluations, corrections) | Up to 50% |
| Context Trimming | Reduce input tokens via summarization | Variable (fewer input tokens) |
| Container Apps Scale-to-Zero | No compute cost when idle | Pay only for usage |
| Cosmos DB Serverless | Low-throughput memory storage | No idle cost |
| GPT-4.1-mini for Summarization | Use cheapest model for internal tasks | 10-20x cheaper than GPT-5.5 |

---

## Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Edge: API Management + WAF + Azure Front Door            │    │
│  │  • DDoS protection                                       │    │
│  │  • Token rate limiting (TPM/RPM)                         │    │
│  │  • Subscription keys + OAuth2                            │    │
│  │  • Content Safety policy enforcement                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Identity: Microsoft Entra ID + Managed Identity          │    │
│  │  • Keyless authentication (DefaultAzureCredential)       │    │
│  │  • RBAC per resource (fine-grained)                      │    │
│  │  • Service-to-service via Managed Identity               │    │
│  │  • No API keys in code or config                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Network: Private Endpoints + VNet Integration            │    │
│  │  • Private Endpoints for OpenAI, AI Search, Cosmos DB    │    │
│  │  • VNet-integrated Container Apps                        │    │
│  │  • NSG rules (least privilege)                           │    │
│  │  • No public internet access to AI services              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Data: Encryption + Access Control                        │    │
│  │  • Customer-managed keys via Key Vault (CMK)             │    │
│  │  • TLS 1.3 in transit                                    │    │
│  │  • Azure Key Vault for secrets + automatic rotation      │    │
│  │  • Content Safety PII filtering at gateway               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Audit: Logging + Compliance                              │    │
│  │  • Azure Activity Log (control plane audit)              │    │
│  │  • Diagnostic Settings (full I/O capture)                │    │
│  │  • Microsoft Defender for Cloud (threat detection)       │    │
│  │  • Azure Policy (compliance enforcement)                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Getting Started

### Prerequisites
- Azure subscription with Azure OpenAI access enabled
- Azure CLI installed
- Python 3.12+

### Quick Start with Bicep

```bash
# Login to Azure
az login

# Create resource group
az group create --name genai-pipeline-rg --location eastus2

# Deploy the pipeline stack
az deployment group create \
    --resource-group genai-pipeline-rg \
    --template-file main.bicep \
    --parameters projectName=genai-pipeline
```

### Key Bicep Resources

```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2024-10-01' = {
  name: 'genai-openai'
  location: location
  kind: 'OpenAI'
  sku: { name: 'S0' }
  properties: {
    customSubDomainName: 'genai-openai'
    publicNetworkAccess: 'Disabled'
  }
}

resource search 'Microsoft.Search/searchServices@2024-06-01-preview' = {
  name: 'genai-search'
  location: location
  sku: { name: 'standard' }
  properties: {
    semanticSearch: 'standard'
    publicNetworkAccess: 'disabled'
  }
}

resource cosmosdb 'Microsoft.DocumentDB/databaseAccounts@2024-05-15' = {
  name: 'genai-cosmos'
  location: location
  kind: 'GlobalDocumentDB'
  properties: {
    capabilities: [{ name: 'EnableServerless' }]
    databaseAccountOfferType: 'Standard'
  }
}
```

---

## References

- [Azure Well-Architected Framework — AI Workloads](https://learn.microsoft.com/en-us/azure/well-architected/ai/get-started)
- [Azure OpenAI Service Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [Azure AI Content Safety](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/)
- [Azure AI Search](https://learn.microsoft.com/en-us/azure/search/)
- [Azure Cosmos DB Vector Search](https://learn.microsoft.com/en-us/azure/cosmos-db/vector-database)
- [Microsoft Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-services/)
- [Microsoft Agent Framework](https://github.com/microsoft/agent-framework)
- [Microsoft GraphRAG](https://github.com/microsoft/graphrag)
- [Azure API Management GenAI Gateway](https://learn.microsoft.com/en-us/azure/api-management/)
- [Azure Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/)
- [Microsoft Foundry Evaluations](https://learn.microsoft.com/en-us/azure/ai-studio/concepts/evaluation-approach-gen-ai)
- [Azure ML Responsible AI](https://learn.microsoft.com/en-us/azure/machine-learning/concept-responsible-ai)

---

## License

MIT
