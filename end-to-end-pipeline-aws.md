# AI GenAI End-to-End Pipeline — AWS Implementation

A production-grade implementation of the GenAI LLM application pipeline using AWS managed services. This document maps each pipeline component to its AWS equivalent, following the [AWS Well-Architected Generative AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lens.html) (published November 2025) which defines six architectural pillars: operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability.

---

## AWS Service Mapping

| # | Pipeline Component | AWS Service(s) |
|---|-------------------|----------------|
| 1 | Input/Output Guardrails | Amazon Bedrock Guardrails + ApplyGuardrail API |
| 2 | LLM Router | Amazon Bedrock Intelligent Prompt Routing |
| 3 | Context Manager | AWS Lambda + Amazon Bedrock (summarization) |
| 4 | Context Memory | Amazon DynamoDB + Amazon OpenSearch Serverless (vector) |
| 5 | RAG Pipeline | Amazon Bedrock Knowledge Bases + Amazon OpenSearch Serverless |
| 6 | GraphRAG | Amazon Bedrock Knowledge Bases (Graph) + Amazon Neptune |
| 7 | Auto-Correction | AWS Step Functions + Amazon Bedrock + Amazon S3 |
| 8 | Observability | Amazon CloudWatch + AWS X-Ray + Bedrock Model Invocation Logging |
| 9 | Evaluations | Amazon Bedrock Evaluations + FMEval |
| 10 | Explainability | Amazon SageMaker Clarify + Custom Lambda |
| 11 | LLM Caching | Amazon ElastiCache (Redis OSS) + Amazon Bedrock Prompt Caching |
| 12 | LLM Batching | Amazon Bedrock Batch Inference + AWS Step Functions |
| 13 | Orchestration | Amazon Bedrock Flows + Amazon Bedrock Agents |
| 14 | Infrastructure | API Gateway + Lambda/ECS Fargate + Step Functions + EventBridge + SQS |

---

## End-to-End Architecture Diagram (AWS)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    USER / CLIENT APP                                     │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              AMAZON API GATEWAY (REST/WebSocket)                          │
│                         ┌──────────────────────────────────────┐                         │
│                         │  WAF + Throttling + API Keys + CORS  │                         │
│                         └──────────────────────────────────────┘                         │
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
│   │                    AMAZON BEDROCK GUARDRAILS (Input)                           │     │
│   │                    [ApplyGuardrail API — standalone, no model needed]          │     │
│   │                                                                               │     │
│   │   ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│   │   │ Sensitive     │  │ Content Filters   │  │   Denied    │  │  Word      │  │     │
│   │   │ Information   │  │ (Hate, Violence,  │  │   Topics    │  │  Filters   │  │     │
│   │   │ Filters (PII) │  │  Prompt Attack)   │  │             │  │            │  │     │
│   │   └──────┬───────┘  └─────────┬─────────┘  └──────┬──────┘  └─────┬──────┘  │     │
│   │          └─────────────────────┴───────────────────┴───────────────┘         │     │
│   │                                    │                                          │     │
│   │                    ┌───────────────┴────────────────────┐                    │     │
│   │                    │ Contextual Grounding Check         │                    │     │
│   │                    │ (hallucination + relevance)         │                    │     │
│   │                    └───────────────┬────────────────────┘                    │     │
│   │                                    │                                          │     │
│   │                          ┌─────────┴─────────┐                               │     │
│   │                          │ INTERVENE or PASS  │                               │     │
│   │                          └─────────┬─────────┘                               │     │
│   └────────────────────────────────────┼──────────────────────────────────────────┘     │
│                                        │                                                 │
│                                        ▼                                                 │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                    LLM CACHE (ElastiCache Redis + Bedrock Prompt Caching)      │     │
│   │                                                                               │     │
│   │   ┌──────────────────┐       ┌──────────────────────────────────────────┐    │     │
│   │   │  Incoming Query  │──────▶│           CACHE LOOKUP                   │    │     │
│   │   └──────────────────┘       │                                          │    │     │
│   │                              │  Strategy A: Exact Match (Redis hash)     │    │     │
│   │                              │  Strategy B: Semantic Similarity          │    │     │
│   │                              │    (Titan Embeddings cosine ≥ 0.95)       │    │     │
│   │                              │  Strategy C: Bedrock Prompt Caching       │    │     │
│   │                              │    (reuse common context prefixes)        │    │     │
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
│   │              AMAZON BEDROCK INTELLIGENT PROMPT ROUTING                         │     │
│   │              [Up to 30% cost reduction while maintaining quality]              │     │
│   │                                                                               │     │
│   │   ┌──────────────────────────────────────────────────────────────────────┐   │     │
│   │   │  Analyzes query complexity → Routes to optimal model tier            │   │     │
│   │   │  Supports: Anthropic (Haiku/Sonnet/Opus), Meta Llama, Amazon Nova    │   │     │
│   │   └──────────────────────────────────────────────────────────────────────┘   │     │
│   │                                                                               │     │
│   │              ┌─────────────────────┼─────────────────────┐                   │     │
│   │              ▼                     ▼                     ▼                   │     │
│   │      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐             │     │
│   │      │ Claude Haiku │      │Claude Sonnet│      │ Claude Opus │             │     │
│   │      │ / Nova Micro │      │ / Nova Lite │      │ / Nova Pro  │             │     │
│   │      │  Fast/Cheap  │      │  Balanced   │      │  Capable    │             │     │
│   │      └─────────────┘      └─────────────┘      └─────────────┘             │     │
│   └───────────────────────────────────────────────────────────────────────────────┘     │
│                                                                                         │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │              AMAZON BEDROCK CROSS-REGION INFERENCE                             │     │
│   │              [No additional routing cost — pricing based on source Region]     │     │
│   │                                                                               │     │
│   │      ┌─────────────────┐              ┌──────────────────────┐               │     │
│   │      │  Geographic     │              │  Global              │               │     │
│   │      │  (US/EU/APAC)   │              │  (All commercial     │               │     │
│   │      │                 │              │   Regions, ~10%      │               │     │
│   │      │                 │              │   savings)           │               │     │
│   │      └─────────────────┘              └──────────────────────┘               │     │
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
│   │  BEDROCK KNOWLEDGE   │  │  BEDROCK KNOWLEDGE   │  │   CONTEXT MEMORY         │     │
│   │  BASES (RAG)         │  │  BASES (GraphRAG)    │  │   (DynamoDB + OpenSearch │     │
│   │                      │  │                      │  │    Serverless)           │     │
│   │  Data Sources:       │  │  Graph Store:        │  │                          │     │
│   │  • Amazon S3         │  │  • Amazon Neptune    │  │  Query                   │     │
│   │  • Web Crawler       │  │  • Neptune Analytics │  │    │                     │     │
│   │  • Confluence        │  │                      │  │    ▼                     │     │
│   │  • SharePoint        │  │  Features:           │  │  Semantic Similarity     │     │
│   │  • Salesforce        │  │  • Entity extraction │  │  Search (OpenSearch      │     │
│   │                      │  │  • Relationship      │  │   k-NN vector engine)    │     │
│   │  Vector Stores:      │  │    mapping           │  │    │                     │     │
│   │  • OpenSearch        │  │  • Graph traversal   │  │    ▼                     │     │
│   │    Serverless        │  │  • Community         │  │  DynamoDB (node/edge     │     │
│   │  • Aurora pgvector   │  │    detection         │  │   persistence, TTL)      │     │
│   │  • Amazon Neptune    │  │  • Multi-hop         │  │    │                     │     │
│   │  • Pinecone          │  │    reasoning         │  │    ▼                     │     │
│   │  • Redis Enterprise  │  │                      │  │  Ranked Memories         │     │
│   │                      │  │  Query:              │  │  (thoughts, beliefs,     │     │
│   │  Retrieval:          │  │  • Entity matching   │  │   preferences, goals)    │     │
│   │  • Semantic search   │  │  • Subgraph extract  │  │                          │     │
│   │  • Hybrid (semantic  │  │  • Context assembly  │  │  Async Extraction:       │     │
│   │    + keyword)        │  │                      │  │  • EventBridge trigger   │     │
│   │  • Reranking         │  │                      │  │  • Lambda + Bedrock      │     │
│   │  • Metadata filters  │  │                      │  │  • DynamoDB Streams      │     │
│   └──────────┬───────────┘  └──────────┬───────────┘  └────────────┬─────────────┘     │
│              │                          │                            │                   │
│              └──────────────────────────┼────────────────────────────┘                   │
│                                         │                                                │
│                                         ▼                                                │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │                  CONTEXT MANAGER (AWS Lambda / ECS Task)                       │     │
│   │                                                                               │     │
│   │  ┌─────────────────┐  ┌───────────────────┐  ┌────────────────────────────┐  │     │
│   │  │  Token Counter  │  │ Trimming Strategy │  │  Summarization via        │  │     │
│   │  │  (tiktoken /    │  │ (FIFO / Sliding   │  │  Amazon Bedrock           │  │     │
│   │  │   Bedrock API)  │  │  Window / Priority)│  │  (Claude Haiku — fast,   │  │     │
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
│   │                     AMAZON BEDROCK CONVERSE API                                │     │
│   │                                                                               │     │
│   │         ┌───────────────────────────────────────────────────────┐            │     │
│   │         │              MODEL INVOCATION                          │            │     │
│   │         │                                                       │            │     │
│   │         │  • Unified API across all Bedrock models              │            │     │
│   │         │  • Streaming via ConverseStream                       │            │     │
│   │         │  • Built-in retry with exponential backoff            │            │     │
│   │         │  • Guardrails applied inline (guardrailConfig)        │            │     │
│   │         │  • Cross-region inference for burst capacity          │            │     │
│   │         │                                                       │            │     │
│   │         │  Models Available:                                     │            │     │
│   │         │  • Anthropic: Claude Opus, Sonnet, Haiku              │            │     │
│   │         │  • Amazon: Nova Pro, Nova Lite, Nova Micro            │            │     │
│   │         │  • Meta: Llama 3.x / 4.x                             │            │     │
│   │         │  • Mistral, Cohere, AI21, Stability                   │            │     │
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
│   │                 AMAZON BEDROCK GUARDRAILS (Output)                             │     │
│   │                 [Same guardrail applied to model responses]                    │     │
│   │                                                                               │     │
│   │   ┌──────────────┐  ┌───────────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│   │   │ PII Redactor │  │  Contextual       │  │  Content    │  │  Automated │  │     │
│   │   │ (Sensitive   │  │  Grounding Check  │  │  Filter     │  │  Reasoning │  │     │
│   │   │  Info Filter)│  │  (Hallucination)  │  │  (Toxicity) │  │  Checks    │  │     │
│   │   └──────┬───────┘  └─────────┬─────────┘  └──────┬──────┘  └─────┬──────┘  │     │
│   │          └─────────────────────┴───────────────────┴───────────────┘         │     │
│   │                                    │                                          │     │
│   │                          ┌─────────┴─────────┐                               │     │
│   │                          │ INTERVENE or PASS  │                               │     │
│   │                          └─────────┬─────────┘                               │     │
│   └────────────────────────────────────┼──────────────────────────────────────────┘     │
│                                        │                                                 │
│                                        ▼                                                 │
│   ┌───────────────────────────────────────────────────────────────────────────────┐     │
│   │              AUTO-CORRECTION PIPELINE (AWS Step Functions)                     │     │
│   │                                                                               │     │
│   │   ┌──────────────────┐       ┌────────────────────────────────────────┐      │     │
│   │   │  Actual Response │──────▶│           COMPARATOR                   │      │     │
│   │   └──────────────────┘       │           (Lambda Function)            │      │     │
│   │                              │                                        │      │     │
│   │                              │  Strategy A: Bedrock Embeddings        │      │     │
│   │                              │    (Titan Embeddings — cosine sim)     │      │     │
│   │   ┌──────────────────┐       │  Strategy B: Bedrock LLM-as-Judge     │      │     │
│   │   │  Ideal Response  │──────▶│    (Claude evaluation)                │      │     │
│   │   │  (S3 ground      │       │                                        │      │     │
│   │   │   truth store)   │       │  Output: Confidence Score (0.0 - 1.0)  │      │     │
│   │   └──────────────────┘       └───────────────────┬────────────────────┘      │     │
│   │                                                   │                           │     │
│   │                                ┌──────────────────┴──────────────────┐        │     │
│   │                                │                                     │        │     │
│   │                          score >= 0.8                          score < 0.8    │     │
│   │                                │                                     │        │     │
│   │                                ▼                                     ▼        │     │
│   │                          ┌──────────┐                  ┌─────────────────┐   │     │
│   │                          │   PASS   │                  │ Capture to S3   │   │     │
│   │                          └──────────┘                  │ for fine-tuning │   │     │
│   │                                                        │ (SageMaker)     │   │     │
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
│   │  Amazon CloudWatch:    │ │  Bedrock Evaluations:  │ │  SageMaker Clarify:        │  │
│   │  • Custom Metrics      │ │  • Automatic (built-in │ │  • Feature attribution     │  │
│   │    (TTFT, latency,     │ │    metrics: accuracy,  │ │  • SHAP values             │  │
│   │     token usage)       │ │    robustness,         │ │  • Bias detection          │  │
│   │  • CloudWatch Logs     │ │    toxicity)           │ │                            │  │
│   │  • Dashboards          │ │  • Human-based (custom │ │  Custom Lambda:            │  │
│   │  • Alarms + SNS       │ │    workforce review)   │ │  • Token attribution       │  │
│   │                        │ │  • LLM-as-judge        │ │  • Chain-of-thought        │  │
│   │  AWS X-Ray:            │ │                        │ │    quality scoring         │  │
│   │  • Distributed tracing │ │  FMEval (open-source): │ │  • Attention analysis      │  │
│   │  • Service map         │ │  • Factual knowledge   │ │                            │  │
│   │  • Latency analysis    │ │  • Summarization       │ │  Bedrock Invocation Logs:  │  │
│   │                        │ │  • QA accuracy         │ │  • Full request/response   │  │
│   │  Bedrock Invocation    │ │  • Toxicity            │ │    capture to S3           │  │
│   │  Logging:              │ │  • Stereotyping        │ │  • Prompt analysis         │  │
│   │  • S3 / CloudWatch     │ │  • Prompt stereotyping │ │                            │  │
│   │  • Full I/O capture    │ │                        │ │                            │  │
│   │  • Cost metadata       │ │                        │ │                            │  │
│   │                        │ │                        │ │                            │  │
│   │  Cost Tracking:        │ │                        │ │                            │  │
│   │  • Request metadata    │ │                        │ │                            │  │
│   │    tags (cost center,  │ │                        │ │                            │  │
│   │    team, project)      │ │                        │ │                            │  │
│   │  • AWS Cost Explorer   │ │                        │ │                            │  │
│   │  • Budget Alerts       │ │                        │ │                            │  │
│   └────────────────────────┘ └────────────┬───────────┘ └────────────────────────────┘  │
│                                           │                                              │
│                                           ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                  LLM BATCHING (Bedrock Batch Inference)                           │   │
│   │                                                                                  │   │
│   │   ┌────────────────────────────────────────────────────────────────────────┐    │   │
│   │   │                      BATCH JOB MANAGER                                 │    │   │
│   │   │                                                                        │    │   │
│   │   │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │    │   │
│   │   │  │  S3 Input JSONL  │  │  Batch Inference │  │  S3 Output JSONL    │  │    │   │
│   │   │  │  (collect eval   │  │  Job (Bedrock    │  │  (results mapped    │  │    │   │
│   │   │  │   queries)       │  │   CreateModel-   │  │   back to queries)  │  │    │   │
│   │   │  │                  │  │   InvocationJob) │  │                     │  │    │   │
│   │   │  └────────┬─────────┘  └────────┬─────────┘  └──────────┬──────────┘  │    │   │
│   │   │           └─────────────────────┼────────────────────────┘             │    │   │
│   │   │                                 │                                      │    │   │
│   │   │              Up to 50% cost savings vs real-time inference             │    │   │
│   │   └────────────────────────────────────────────────────────────────────────┘    │   │
│   │                                                                                  │   │
│   │   Use Cases:                                                                     │   │
│   │   • Batch evaluation scoring (submit 100s of eval queries at once)              │   │
│   │   • Offline quality assessments (nightly eval runs via EventBridge)              │   │
│   │   • Fine-tuning data validation (bulk comparison against ground truth)          │   │
│   │   • A/B test evaluation (compare model versions across test suites)             │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Phases (AWS Implementation)

### Phase 1: Input Processing

#### Amazon Bedrock Guardrails (Input)

[Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) provides six configurable safeguard types that can be applied to both input prompts and model responses:

| Safeguard Type | Function | Configuration |
|---------------|----------|---------------|
| Content Filters | Blocks hate, insults, sexual content, violence, misconduct, prompt attacks | Configurable strength (NONE/LOW/MEDIUM/HIGH) per category |
| Denied Topics | Rejects queries on restricted subjects | Custom topic definitions with example phrases |
| Word Filters | Blocks specific words/phrases | Custom word lists + managed profanity list |
| Sensitive Information Filters | Detects/redacts PII (SSN, email, phone, address) | Probabilistic detection + custom regex patterns |
| Contextual Grounding Checks | Verifies factual grounding and relevance in RAG responses | Threshold-based scoring |
| Automated Reasoning Checks | Validates logical consistency | Rule-based verification |

**Key capability:** The `ApplyGuardrail` API enables content evaluation independently of model invocation — guardrails can run as a standalone preprocessing step before any LLM call.

```python
import boto3

bedrock_runtime = boto3.client("bedrock-runtime")

response = bedrock_runtime.apply_guardrail(
    guardrailIdentifier="your-guardrail-id",
    guardrailVersion="DRAFT",
    source="INPUT",
    content=[{"text": {"text": user_input}}]
)

if response["action"] == "GUARDRAIL_INTERVENED":
    return {"blocked": True, "message": response["outputs"][0]["text"]}
```

#### Amazon Bedrock Intelligent Prompt Routing

[Intelligent Prompt Routing](https://aws.amazon.com/bedrock/) dynamically routes each request to the model predicted most likely to give the desired response at the lowest cost:

- Analyzes query complexity automatically (no custom signals needed)
- Routes across model pairs: Anthropic (Haiku/Sonnet/Opus), Meta Llama, Amazon Nova
- Reduces costs by up to 30% while maintaining response quality
- No manual threshold tuning required — AWS manages the routing logic

```python
response = bedrock_runtime.converse(
    modelId="us.anthropic.prompt-router-v1:0",
    messages=[{"role": "user", "content": [{"text": query}]}]
)
```

#### Amazon Bedrock Cross-Region Inference

[Cross-region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html) manages traffic bursts and throughput optimization:

| Profile Type | Scope | Cost |
|-------------|-------|------|
| Geographic | Routes within US, EU, or APAC boundaries | Source Region pricing (no extra cost) |
| Global | Routes to any supported AWS commercial Region | ~10% savings vs Geographic |

- Handles unplanned traffic bursts by automatically routing to Regions with available capacity
- No additional routing cost — pricing is based on source Region
- Limitation: does not support Provisioned Throughput

#### LLM Cache (ElastiCache + Bedrock Prompt Caching)

Two caching layers reduce cost and latency:

**Layer 1: Response Cache (Amazon ElastiCache for Redis OSS)**
- Exact match: Hash-based key lookup for identical queries (sub-millisecond)
- Semantic match: Embedding similarity via Titan Embeddings (cosine threshold ≥ 0.95)
- TTL management: Configurable expiry per query type
- On **CACHE HIT**: returns cached response immediately, bypassing the full pipeline
- On **CACHE MISS**: continues to Router; after LLM response, writes to cache

**Layer 2: Context Prefix Cache (Amazon Bedrock Prompt Caching)**
- Caches common system prompts and context prefixes across requests
- Reduces input token costs for repeated context patterns
- Managed by Bedrock — no infrastructure to configure

```python
import redis
import hashlib
import json

cache = redis.Redis(host="elasticache-endpoint", port=6379, ssl=True)

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

#### Amazon Bedrock Knowledge Bases (RAG)

[Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) provides fully managed RAG:

**Data Sources:**
- Amazon S3 (documents: PDF, TXT, MD, HTML, DOC, CSV)
- Web Crawler (automated URL ingestion)
- Confluence, SharePoint, Salesforce connectors

**Vector Stores (choose one):**
- Amazon OpenSearch Serverless (recommended for scale — auto-scales, no cluster management)
- Amazon Aurora PostgreSQL (pgvector extension)
- Amazon Neptune (graph + vector)
- Pinecone, Redis Enterprise, MongoDB Atlas (third-party)

**Retrieval Pipeline:**
1. Automatic document chunking (fixed-size, semantic, or hierarchical)
2. Embedding generation (Amazon Titan Embeddings or Cohere)
3. Hybrid retrieval (semantic + keyword search)
4. Built-in reranking (Cohere Rerank or Amazon Rerank model)
5. Metadata filtering for precision
6. Source attribution in responses

```python
bedrock_agent_runtime = boto3.client("bedrock-agent-runtime")

response = bedrock_agent_runtime.retrieve_and_generate(
    input={"text": "What is our refund policy?"},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": "KB_ID",
            "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0",
            "retrievalConfiguration": {
                "vectorSearchConfiguration": {
                    "numberOfResults": 10,
                    "overrideSearchType": "HYBRID"
                }
            }
        }
    }
)
```

#### Amazon Bedrock Knowledge Bases (GraphRAG)

[Bedrock Knowledge Bases with Graph](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs.html) enables knowledge-graph-enhanced retrieval:

**Graph Store:** Amazon Neptune / Neptune Analytics

**Capabilities:**
- Automatic entity and relationship extraction from documents
- Graph construction and community detection
- Multi-hop reasoning via graph traversal
- Combines vector similarity with graph neighborhood context
- Subgraph extraction for contextually rich answers

**Architecture:**
```
Documents (S3) → Bedrock KB Ingestion → Entity Extraction (LLM)
                                              │
                                              ▼
                                    Amazon Neptune (Graph)
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                        Entity Match    Graph Traversal  Community Context
                              │               │               │
                              └───────────────┼───────────────┘
                                              ▼
                                    Enriched Context for LLM
```

#### Context Memory (DynamoDB + OpenSearch Serverless)

AWS does not provide a single managed "memory graph" service, so this component uses a combination:

| Layer | AWS Service | Purpose |
|-------|-------------|---------|
| Node/Edge Persistence | Amazon DynamoDB | Memory nodes, edges, metadata with TTL |
| Vector Search | Amazon OpenSearch Serverless (vector engine) | Semantic similarity search via k-NN |
| Embedding Generation | Amazon Bedrock (Titan Embeddings) | Convert memory content to vectors |
| Async Extraction | Amazon EventBridge + Lambda | Fire-and-forget memory extraction |
| Graph Queries | AWS AppSync + DynamoDB | BFS traversal via GSI adjacency lists |

**Architecture:**
```
Request → Lambda (Context Builder)
              │
              ├── OpenSearch Serverless: k-NN search for relevant memories
              ├── DynamoDB: Fetch full node data + traverse edges (GSI)
              ├── Bedrock (Titan Embeddings): Query embedding
              └── Token Budget: Greedy select within limit
              │
              ▼
         Formatted memory context → injected into system prompt

Post-Response (async):
         EventBridge → Lambda (Extraction) → Bedrock (structured output)
              │
              ├── DynamoDB: Persist new memory nodes/edges
              └── OpenSearch: Index new embeddings
```

#### Context Manager (Lambda Function)

Token counting and context assembly runs as an AWS Lambda function:

- **Token counting:** tiktoken library (packaged as Lambda Layer) or Bedrock API counting
- **Trimming strategies:** FIFO, sliding window, priority-based (implemented in Lambda code)
- **Summarization:** Calls Bedrock (Claude Haiku — fast, cheap) to condense older conversation turns
- **Output:** Token-budgeted context window ready for the final LLM call

---

### Phase 3: LLM Execution

#### Amazon Bedrock Converse API

The [Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-call.html) provides a unified interface across all Bedrock models:

```python
response = bedrock_runtime.converse_stream(
    modelId="anthropic.claude-sonnet-4-20250514-v1:0",
    messages=assembled_messages,
    system=[{"text": system_prompt_with_memories}],
    inferenceConfig={"maxTokens": 4096, "temperature": 0.7},
    guardrailConfig={
        "guardrailIdentifier": "guardrail-id",
        "guardrailVersion": "1",
        "streamProcessingMode": "async"
    }
)

for event in response["stream"]:
    if "contentBlockDelta" in event:
        yield event["contentBlockDelta"]["delta"]["text"]
```

**Key features:**
- Unified API across Anthropic, Amazon, Meta, Mistral, Cohere models
- Native streaming via `ConverseStream`
- Inline guardrail application (no separate call needed)
- Built-in retry logic with exponential backoff (AWS SDK)
- Tool use / function calling support
- Cross-region inference for burst capacity

---

### Phase 4: Output Processing

#### Amazon Bedrock Guardrails (Output)

The same guardrail applied during input processing also validates model responses:

- **PII Redaction:** Sensitive Information Filters mask any PII that leaked into output
- **Hallucination Detection:** Contextual Grounding Check verifies output against provided context
- **Content Safety:** Content Filters catch toxic/harmful content in responses
- **Automated Reasoning:** Logical consistency validation

When applied inline via `guardrailConfig` in the Converse API, output guardrails run automatically without a separate API call.

#### Auto-Correction Pipeline (AWS Step Functions)

```
┌─────────────────────────────────────────────────────────────────┐
│                  AWS STEP FUNCTIONS WORKFLOW                      │
│                                                                  │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────────┐    │
│  │ Start   │───▶│ Lambda:      │───▶│ Lambda:             │    │
│  │         │    │ Fetch Ground │    │ Compare (Cosine     │    │
│  │         │    │ Truth (S3)   │    │ Sim / LLM-Judge)    │    │
│  └─────────┘    └──────────────┘    └──────────┬──────────┘    │
│                                                 │               │
│                              ┌──────────────────┴────────┐      │
│                              │                           │      │
│                        score >= 0.8               score < 0.8   │
│                              │                           │      │
│                              ▼                           ▼      │
│                      ┌────────────┐         ┌────────────────┐  │
│                      │ Pass       │         │ Lambda:        │  │
│                      │ (no-op)    │         │ Write to S3    │  │
│                      └────────────┘         │ (correction    │  │
│                                             │  record)       │  │
│                                             └───────┬────────┘  │
│                                                     │           │
│                                                     ▼           │
│                                             ┌────────────────┐  │
│                                             │ SageMaker:     │  │
│                                             │ Fine-tuning    │  │
│                                             │ dataset (batch)│  │
│                                             └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Components:**
- **Ground truth store:** Amazon S3 (structured JSONL files)
- **Comparison strategies:**
  - Cosine similarity via Amazon Bedrock Titan Embeddings
  - LLM-as-Judge via Bedrock (Claude evaluates response quality)
- **Correction capture:** S3 + DynamoDB (query + actual + ideal + metadata)
- **Fine-tuning pipeline:** Amazon SageMaker for model customization with accumulated corrections

---

### Phase 5: Monitoring & Continuous Improvement

#### Observability

| Metric | AWS Service | Implementation |
|--------|------------|----------------|
| Time to First Token (TTFT) | CloudWatch Custom Metrics | Lambda measures Bedrock streaming start |
| Total Latency | AWS X-Ray | End-to-end trace across all services |
| Inter-Token Latency | CloudWatch Custom Metrics | Calculated from stream events |
| Token Usage | Bedrock Model Invocation Logging | Automatic capture to S3/CloudWatch |
| Cost per Request | Bedrock Request Metadata Tags | Tag by team/project/cost-center |
| Error Rates | CloudWatch Alarms + SNS | Threshold-based alerting |
| Distributed Tracing | AWS X-Ray | Service map + latency analysis |

**Bedrock Model Invocation Logging:**
```python
bedrock = boto3.client("bedrock")

bedrock.put_model_invocation_logging_configuration(
    loggingConfig={
        "cloudWatchConfig": {
            "logGroupName": "/aws/bedrock/model-invocations",
            "roleArn": "arn:aws:iam::ACCOUNT:role/BedrockLoggingRole",
            "largeDataDelivery": {
                "s3Config": {
                    "bucketName": "bedrock-logs-bucket",
                    "keyPrefix": "invocations/"
                }
            }
        },
        "s3Config": {
            "bucketName": "bedrock-logs-bucket",
            "keyPrefix": "full-logs/"
        },
        "textDataDeliveryEnabled": True,
        "imageDataDeliveryEnabled": True
    }
)
```

**Cost Tracking with Request Metadata:**
```python
response = bedrock_runtime.converse(
    modelId="anthropic.claude-sonnet-4-20250514-v1:0",
    messages=messages,
    requestMetadata={
        "team": "search-platform",
        "project": "customer-support-bot",
        "costCenter": "CC-1234"
    }
)
```

#### Evaluations

**Amazon Bedrock Evaluations** (managed service):
- Automatic evaluation: built-in metrics for accuracy, robustness, toxicity
- Human evaluation: custom workforce review via Amazon SageMaker Ground Truth
- LLM-as-judge: model-based evaluation with customizable criteria

**FMEval** (open-source library by AWS):
| Metric | What It Measures |
|--------|-----------------|
| Factual Knowledge | Does the model know correct facts? |
| QA Accuracy | Are answers correct for given questions? |
| Summarization | Quality of text summarization |
| Toxicity | Harmful content generation rate |
| Stereotyping | Bias in model outputs |
| Prompt Stereotyping | Whether prompts elicit biased responses |

```python
from fmeval.eval_algorithms.qa_accuracy import QAAccuracy
from fmeval.model_runners.bedrock_model_runner import BedrockModelRunner

model_runner = BedrockModelRunner(
    model_id="anthropic.claude-sonnet-4-20250514-v1:0",
    content_template='{"messages": [{"role": "user", "content": "$prompt"}]}'
)

eval_algo = QAAccuracy()
results = eval_algo.evaluate(model=model_runner, dataset=eval_dataset)
```

#### LLM Batching (Bedrock Batch Inference)

[Amazon Bedrock Batch Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html) enables submitting large volumes of evaluation queries at reduced cost:

- Submit JSONL files to S3 with hundreds/thousands of inference requests
- Bedrock processes asynchronously with up to 50% cost savings vs real-time
- Results written to S3 output bucket, mapped back to original queries
- Supports all Bedrock models (Claude, Nova, Llama, Mistral)

```python
bedrock = boto3.client("bedrock")

response = bedrock.create_model_invocation_job(
    jobName="nightly-eval-run",
    modelId="anthropic.claude-sonnet-4-20250514-v1:0",
    roleArn="arn:aws:iam::ACCOUNT:role/BedrockBatchRole",
    inputDataConfig={
        "s3InputDataConfig": {
            "s3Uri": "s3://eval-bucket/input/eval-queries.jsonl",
            "s3InputFormat": "JSONL"
        }
    },
    outputDataConfig={
        "s3OutputDataConfig": {
            "s3Uri": "s3://eval-bucket/output/"
        }
    }
)
```

**Scheduling:** Use Amazon EventBridge to trigger nightly/weekly batch evaluation runs via Step Functions orchestration.

#### Explainability

| Approach | AWS Service | Capability |
|----------|-------------|-----------|
| Feature Attribution | SageMaker Clarify | SHAP-based token importance |
| Bias Detection | SageMaker Clarify | Pre/post-training bias metrics |
| Chain-of-Thought Analysis | Custom Lambda + Bedrock | Parse and score reasoning steps |
| Prompt Analysis | Bedrock Invocation Logs + Athena | Query logged prompts/responses |
| Attention Visualization | Custom (SageMaker Notebook) | Model-specific attention weights |

---

## Orchestration Services

### Amazon Bedrock Flows

[Bedrock Flows](https://docs.aws.amazon.com/bedrock/latest/userguide/flows.html) enables visual, serverless workflow orchestration for the entire pipeline:

**Node Types:**
| Node | Purpose |
|------|---------|
| Prompt | Invoke a model with a managed prompt |
| Knowledge Base | RAG retrieval |
| Agent | Multi-step reasoning |
| Lambda | Custom logic (trimming, scoring, etc.) |
| Guardrails | Safety checks on prompt/KB nodes |
| S3 | Read/write data |
| Amazon Lex | Conversational routing |
| Condition | Branching logic |
| Iterator | Loop over collections |
| Collector | Aggregate results |

**Deployment Model:**
- Working drafts for iterative development and testing
- Immutable version snapshots for production
- Aliases for routing traffic (blue/green, canary)
- API invocation — no infrastructure to manage

```
┌──────────┐    ┌────────────┐    ┌─────────────┐    ┌──────────┐    ┌──────────┐
│  Input   │───▶│ Guardrails │───▶│ Knowledge   │───▶│  Prompt  │───▶│  Output  │
│  Node    │    │  Node      │    │  Base Node  │    │  Node    │    │  Node    │
└──────────┘    └────────────┘    └─────────────┘    └──────────┘    └──────────┘
```

### Amazon Bedrock Agents

[Bedrock Agents](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html) for complex multi-step task orchestration:

- **ReAct reasoning:** Breaks down requests into logical steps
- **Action groups:** Connect to external APIs and services
- **Knowledge Base integration:** Built-in RAG for agent context
- **Multi-agent collaboration:** Supervisor agent coordinates specialized sub-agents
- **Memory retention:** Session and long-term memory for task continuity
- **Guardrails integration:** Safety applied at every agent step
- **Code interpreter:** Execute generated code in a sandbox

### Amazon Bedrock AgentCore

A serverless platform for building agents with any framework:
- Supports LangChain, OpenAI Agents SDK, Claude Agent SDK, Strands SDK
- Pay-per-use (billed per-second of CPU/memory)
- No infrastructure management
- Use when you need custom frameworks beyond Bedrock Agents

### Amazon Bedrock Prompt Management

Centralized prompt versioning and management:
- Store and version prompt templates
- A/B test prompt variations
- Dynamic variable injection
- Integration with Flows and Agents

---

## Infrastructure Architecture

### Recommended: Serverless-First

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SERVERLESS ARCHITECTURE                               │
│                                                                              │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────────┐  │
│   │ API Gateway  │───▶│ Lambda       │───▶│ Amazon Bedrock               │  │
│   │ (REST/WS)    │    │ (Orchestrator)│    │ (Guardrails + Router +      │  │
│   │              │    │              │    │  Converse + KB)              │  │
│   └──────────────┘    └──────┬───────┘    └──────────────────────────────┘  │
│                              │                                               │
│           ┌──────────────────┼──────────────────────────┐                   │
│           ▼                  ▼                          ▼                   │
│   ┌──────────────┐   ┌──────────────┐   ┌────────────────────────────┐     │
│   │ DynamoDB     │   │ OpenSearch   │   │ Step Functions             │     │
│   │ (Memory +    │   │ Serverless   │   │ (Auto-Correction +        │     │
│   │  Sessions)   │   │ (Vectors)    │   │  Async Workflows)         │     │
│   └──────────────┘   └──────────────┘   └────────────────────────────┘     │
│                                                                              │
│   ┌──────────────┐   ┌──────────────┐   ┌────────────────────────────┐     │
│   │ EventBridge  │   │ SQS          │   │ S3                         │     │
│   │ (Async       │   │ (Extraction  │   │ (Documents + Logs +        │     │
│   │  triggers)   │   │  queue)      │   │  Corrections)             │     │
│   └──────────────┘   └──────────────┘   └────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**When to use Serverless (Lambda + API Gateway):**
- Request volumes < 10,000/second
- Latency tolerance > 100ms (Lambda cold start)
- Variable/bursty traffic patterns
- Cost optimization for low-medium traffic

### Alternative: Container-Based (ECS Fargate)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CONTAINER ARCHITECTURE                                 │
│                                                                              │
│   ┌──────────────┐    ┌────────────────────────────────────────────────┐    │
│   │ ALB          │───▶│ ECS Fargate                                    │    │
│   │ (Load        │    │                                                │    │
│   │  Balancer)   │    │  ┌────────────┐  ┌────────────┐  ┌─────────┐ │    │
│   └──────────────┘    │  │ FastAPI    │  │ Context    │  │ Memory  │ │    │
│                       │  │ Service    │  │ Manager    │  │ Service │ │    │
│                       │  │ (main)     │  │ (sidecar)  │  │ (sidecar)│ │    │
│                       │  └────────────┘  └────────────┘  └─────────┘ │    │
│                       └────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**When to use Containers (ECS Fargate):**
- Sustained high-throughput (> 10,000 req/s)
- Latency-sensitive (avoid cold starts)
- Long-running connections (WebSocket streaming)
- Complex dependency graphs (sentence-transformers, large models)
- Team familiarity with container patterns

### Hybrid Approach (Recommended for Production)

| Component | Deployment | Rationale |
|-----------|-----------|-----------|
| API Layer | API Gateway + Lambda | Serverless scaling, WAF integration |
| Core Pipeline | ECS Fargate | Sustained throughput, no cold starts |
| Async Processing | Lambda + EventBridge | Event-driven, cost-efficient |
| Memory Extraction | Lambda + SQS | Bursty, tolerant of latency |
| Batch Evaluations | Step Functions + Lambda | Scheduled, parallelizable |
| ML Models (local) | ECS Fargate (GPU) | Sentence-transformers, reranking |

---

## Data Flow Summary (AWS)

```
User Query
    │
    ├──► [API Gateway: WAF + Throttle + Auth]
    │         │
    │         ▼
    ├──► [Bedrock Guardrails: ApplyGuardrail API]  ──── INTERVENE ──► Error Response
    │         │
    │         PASS
    │         │
    ├──► [ElastiCache: Cache Lookup]  ──── HIT ──► Return Cached Response ──► User
    │         │
    │         MISS
    │         │
    ├──► [Bedrock Intelligent Prompt Routing] ──► Model Tier Decision
    │         │
    │    ┌────┴────────────────────────────────────┐
    │    │         PARALLEL RETRIEVAL               │
    │    │                                          │
    │    │  [Bedrock KB] ──► Document Chunks        │
    │    │  [Bedrock KB Graph + Neptune] ──► Entities│
    │    │  [DynamoDB + OpenSearch] ──► Memories     │
    │    │                                          │
    │    └────┬────────────────────────────────────┘
    │         │
    │         ▼
    │    [Lambda: Context Manager — Assemble + Trim + Summarize]
    │         │
    │         ▼
    │    [Bedrock Converse API: Selected Model + Context + Inline Guardrails]
    │         │
    │         ├──► [ElastiCache: Write response + metadata + TTL]
    │         │
    │         ▼
    │    [Bedrock Guardrails: Output]  ──── INTERVENE ──► Regenerate / Error
    │         │
    │         PASS
    │         │
    │    [Step Functions: Auto-Correction Workflow]
    │         │         │
    │         │    score < threshold ──► S3 (fine-tuning dataset)
    │         │
    │         ▼
    │    Final Response ──► User
    │
    └──► [CloudWatch + X-Ray: Metrics + Traces across all steps]
    └──► [Bedrock Evaluations: Quality scoring (batch)]
    │         └──► [Bedrock Batch Inference: Submit bulk eval queries at 50% cost]
    └──► [Bedrock Invocation Logging: Full I/O to S3]
```

---

## AWS Well-Architected GenAI Lens — Design Principles Applied

| Principle | Pipeline Implementation |
|-----------|----------------------|
| Design for controlled autonomy | Bedrock Guardrails on every input/output; Agents bounded by action groups |
| Implement comprehensive observability | CloudWatch + X-Ray + Invocation Logging + Cost metadata tags |
| Secure interaction boundaries | API Gateway WAF, IAM least-privilege, VPC endpoints for Bedrock |
| Optimize for cost efficiency | Intelligent Prompt Routing, cross-region Global profiles, Haiku for summarization |
| Design for resilience | Cross-region inference for burst capacity, DLQ on SQS, Step Functions retry |
| Minimize environmental impact | Right-size models (Router), serverless scaling to zero, spot instances for batch |

---

## Technology Stack (AWS)

| Layer | AWS Service |
|-------|------------|
| API Gateway | Amazon API Gateway (REST + WebSocket) |
| Compute | AWS Lambda + Amazon ECS Fargate |
| LLM Provider | Amazon Bedrock (Converse API) |
| Guardrails | Amazon Bedrock Guardrails |
| Routing | Amazon Bedrock Intelligent Prompt Routing |
| RAG | Amazon Bedrock Knowledge Bases |
| Vector Store | Amazon OpenSearch Serverless (vector engine) |
| Graph Store | Amazon Neptune / Neptune Analytics |
| Caching | Amazon ElastiCache (Redis OSS) + Bedrock Prompt Caching |
| Batch Processing | Amazon Bedrock Batch Inference + Step Functions |
| Key-Value Store | Amazon DynamoDB |
| Object Storage | Amazon S3 |
| Orchestration | Amazon Bedrock Flows + AWS Step Functions |
| Event Bus | Amazon EventBridge |
| Queue | Amazon SQS |
| Observability | Amazon CloudWatch + AWS X-Ray |
| Logging | Bedrock Model Invocation Logging |
| Evaluation | Amazon Bedrock Evaluations + FMEval |
| Fine-Tuning | Amazon SageMaker |
| Explainability | Amazon SageMaker Clarify |
| IaC | AWS CDK (generative-ai-cdk-constructs) |
| CI/CD | AWS CodePipeline + CodeBuild |

---

## Cost Optimization Strategies

| Strategy | Mechanism | Estimated Savings |
|----------|-----------|-------------------|
| Intelligent Prompt Routing | Route simple queries to cheaper models | Up to 30% |
| Cross-Region Global Profiles | Route to cheapest available Region | ~10% vs Geographic |
| Provisioned Throughput | Reserved capacity for predictable workloads | Up to 50% vs on-demand |
| Batch Inference | Non-real-time workloads (evaluations, corrections) | Up to 50% |
| Context Trimming | Reduce input tokens via summarization | Variable (fewer input tokens) |
| Caching (Prompt Caching) | Reuse common prefixes across requests | Variable |
| Serverless Scaling | Lambda scales to zero when idle | Pay only for usage |

---

## Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Edge: API Gateway + WAF + CloudFront                     │    │
│  │  • DDoS protection (Shield)                              │    │
│  │  • Rate limiting                                         │    │
│  │  • IP allowlisting                                       │    │
│  │  • API key / Cognito auth                                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Network: VPC + PrivateLink                               │    │
│  │  • VPC endpoints for Bedrock (no internet transit)       │    │
│  │  • Private subnets for ECS tasks                         │    │
│  │  • Security groups (least privilege)                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Data: Encryption + Access Control                        │    │
│  │  • KMS encryption at rest (S3, DynamoDB, OpenSearch)     │    │
│  │  • TLS 1.3 in transit                                    │    │
│  │  • IAM roles (least privilege per service)               │    │
│  │  • Secrets Manager for API keys                          │    │
│  │  • Bedrock Guardrails PII filtering                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Audit: Logging + Compliance                              │    │
│  │  • CloudTrail (API audit)                                │    │
│  │  • Bedrock Invocation Logging (full I/O)                 │    │
│  │  • Config Rules (compliance validation)                  │    │
│  │  • Macie (S3 sensitive data discovery)                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Getting Started

### Prerequisites
- AWS Account with Bedrock model access enabled
- AWS CDK v2 installed
- Python 3.12+

### Quick Start with CDK

```bash
# Install generative-ai-cdk-constructs
pip install cdklabs.generative-ai-cdk-constructs

# Bootstrap CDK
cdk bootstrap aws://ACCOUNT_ID/REGION

# Deploy the pipeline stack
cdk deploy GenAIPipelineStack
```

### Key CDK Constructs Available

The [generative-ai-cdk-constructs](https://github.com/awslabs/generative-ai-cdk-constructs) library provides pre-built patterns:
- `bedrock.KnowledgeBase` — RAG with vector store
- `bedrock.Agent` — Multi-step agent
- `bedrock.Guardrail` — Safety configuration
- `bedrock.Flow` — Workflow orchestration
- `opensearchserverless.VectorCollection` — Vector store
- `neptune.GraphDatabase` — Knowledge graph

---

## References

- [AWS Well-Architected Generative AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lens.html)
- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/)
- [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Amazon Bedrock Flows](https://docs.aws.amazon.com/bedrock/latest/userguide/flows.html)
- [Amazon Bedrock Agents](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html)
- [Amazon Bedrock Cross-Region Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [Amazon Bedrock Model Invocation Logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [Amazon Bedrock Evaluations](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html)
- [FMEval (open-source)](https://github.com/aws/fmeval)
- [Generative AI CDK Constructs](https://github.com/awslabs/generative-ai-cdk-constructs)
- [Amazon Bedrock Samples](https://github.com/aws-samples/amazon-bedrock-samples)

---

## License

MIT
