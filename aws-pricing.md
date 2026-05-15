# AWS Pricing Reference — Document AI Platform

**Region anchor:** `us-east-1` (N. Virginia)
**Scope:** Managed-path services only. Hybrid/EKS/GPU pricing intentionally deferred.
**Last verified:** May 2026 against the cited official AWS pages.
**Purpose:** A single reference to ground every cost claim in the architecture artifact. Every dollar figure in the artifact must trace back to a line in this file.

---

## How to use this document

Every section below documents:

- **Unit price** in `us-east-1`.
- **Billing dimension** (the thing AWS actually counts — often subtler than it looks).
- **Gotchas** — pricing edges that have bitten people before.
- **Source URL** — the authoritative AWS page or model card.

When the architecture cites a cost, it should cite the specific line in this file and re-derive the arithmetic. The previous artifact's failure was that its cost claims were not traceable to verified unit prices, so we treat this file as the single source of truth.

---

## 1. AI tier — the dominant cost lever

These three services drive >70% of variable cost in the managed path. They get the most detailed treatment.

### 1.1 Amazon Bedrock — Meta Llama 3.1 Instruct

This is the selected text LLM family for Stage 4 extraction. We use Bedrock-managed Llama 3.1 in v1 so the pipeline can later move to self-hosted Llama on EKS/vLLM/TGI with less model-family drift. We explicitly avoid `meta.llama3-2-11b-instruct-v1:0` for text extraction because it is a vision/multimodal model.

**Model identity**

| Attribute | Llama 3.1 8B Instruct | Llama 3.1 70B Instruct |
|---|---|---|
| Model ID | `meta.llama3-1-8b-instruct-v1:0` | `meta.llama3-1-70b-instruct-v1:0` |
| Geo inference ID | `us.meta.llama3-1-8b-instruct-v1:0` | `us.meta.llama3-1-70b-instruct-v1:0` |
| Launch date | 2024-07-23 | 2024-07-23 |
| EOL date | 2026-07-23 | 2026-07-23 |
| Lifecycle | Legacy | Legacy |
| Context window | 128K tokens | 128K tokens |
| Max output | 2K tokens | 2K tokens |
| Bedrock APIs | `Invoke`, `Converse` | `Invoke`, `Converse` |
| Endpoint support in us-east-1 | Geo US | Geo US |
| Stage 4 role | Default extraction, pending eval | Stronger recovery path |

**Pricing**

The 8B rate below was provided from the us-east-1 Bedrock pricing page/account view and is now the Stage 4 default extraction rate. Keep the 70B recovery rate as a planning estimate until verified in the same account/region.

| Model | Price per 1M input tokens | Price per 1M output tokens | Notes |
|---|---:|---:|---|
| Llama 3.1 8B Instruct | `$0.22` | `$0.22` | us-east-1 default extraction rate |
| Llama 3.1 8B Instruct batch | `$0.11` | `$0.11` | Batch mode; use only when latency permits |
| Llama 3.1 70B Instruct | `$2.00` | `$2.00` | Planning rate for stronger recovery; verify before final artifact |

> ⚠ **Final-pricing requirement.** Before publishing the final artifact, verify the exact Bedrock Llama 3.1 70B token rates for `us.meta.llama3-1-70b-instruct-v1:0` directly on the AWS Bedrock pricing page or account billing console.

**Endpoint and compliance note**

The selected Llama 3.1 models support US geo inference IDs. This is good for the compliance story because requests stay within the US geography, but it means capacity and quota planning must use the geo inference profile.

**Sources**
- Llama 3.1 8B model card: https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-meta-llama-3-1-8b-instruct.html
- Llama 3.1 70B model card: https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-meta-llama-3-1-70b-instruct.html
- Bedrock pricing page: https://aws.amazon.com/bedrock/pricing/

---

### 1.1A Amazon Bedrock — Claude Haiku 4.5 (superseded reference)

This was the earlier pricing baseline for extraction. It is kept here only as a comparison point because earlier cost math used it. Stage 4 now uses the Llama 3.1 Instruct family instead.

**Model identity**

| Attribute | Value |
|---|---|
| Model ID | `anthropic.claude-haiku-4-5-20251001-v1:0` |
| Launch date | 2025-10-16 |
| Lifecycle | Active (EOL no sooner than 2026-10-01) |
| Context window | 200K tokens |
| Max output | 64K tokens |
| Bedrock APIs | `Invoke`, `Converse` |
| Features supported | Tool use, structured outputs, prompt caching, guardrails, prompt management, agents, flows |
| Features **NOT** supported | Batch inference, intelligent prompt routing |

**Pricing (standard, us-east-1)**

| Dimension | Price per 1M tokens |
|---|---:|
| Input tokens | `$0.80` |
| Output tokens | `$4.00` |

> ⚠ **Critical correction vs. feedback-artifact-1.md.** The feedback cited Anthropic platform pricing of `$1.00 / $5.00` per 1M tokens. The **Bedrock model card** for us-east-1 lists `$0.80 / $4.00`. Bedrock is roughly 20% cheaper than Anthropic platform direct for Haiku 4.5 in this region. The architecture will cite the Bedrock figures, because we're consuming via Bedrock.

**Prompt caching (Bedrock supports — pricing details below)**

Bedrock supports Anthropic-style prompt caching for Haiku 4.5. The economic shape mirrors Anthropic platform:

| Cache operation | Approx. price per 1M tokens (Bedrock, us-east-1) |
|---|---:|
| 5-minute cache write | `~$1.00` (1.25× input price) |
| 1-hour cache write | `~$1.60` (2× input price) |
| Cache read | `~$0.08` (0.1× input price) |

These are the standard Anthropic prompt-caching multipliers applied to Bedrock's per-token rate. Verify against the live console before publishing the artifact, as Bedrock cache pricing is documented on the main Bedrock pricing page rather than the model card.

**Batch inference: NOT supported on Bedrock for Haiku 4.5.** The 50% batch discount available on the Anthropic platform is **unavailable** here. This is a real constraint — we cannot rely on batch pricing for cost optimization.

**Service tiers available**

| Standard | Priority | Flex | Reserved |
|---|---|---|---|
| Yes | No | No | Yes |

**Endpoint pricing — global vs. regional / geo**

Haiku 4.5 in us-east-1 only supports cross-region routing — **no in-region** endpoint exists for us-east-1:

| Endpoint type | Available in us-east-1 | Routing | Pricing impact |
|---|---|---|---|
| In-Region | ❌ No | Stays in one region | n/a in us-east-1 |
| Geo (e.g., `us.anthropic.*`) | ✅ Yes | Routes within US geography | 10% premium over global (per Anthropic pricing page) |
| Global | ✅ Yes | Routes worldwide | Base price ($0.80 / $4.00) |

> ⚠ **Compliance implication.** If we use the global endpoint, the request may be served from any US region OR any region worldwide. For data-residency-sensitive tenants, we must use the geo endpoint (US geography), which carries the 10% premium. For the artifact's cost claims, **we assume the geo (US) endpoint at +10%** by default — it's the only honest default given the case study's compliance mandates.

**Effective pricing assumed in the architecture** (geo endpoint, US geography):

| Dimension | Price per 1M tokens |
|---|---:|
| Input (geo) | `$0.88` |
| Output (geo) | `$4.40` |

**Quotas (us-east-1, default — adjustable)**

| Quota | Default | Adjustable |
|---|---:|---|
| Cross-region requests/minute | 10,000 | Yes |
| Cross-region tokens/minute | 5,000,000 | Yes |
| Max tokens/day | 3,600,000,000 | No |

**Sources**
- Bedrock model card: https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-4-5.html
- Anthropic platform pricing (cache, batch, regional premium structure): https://platform.claude.com/docs/en/about-claude/pricing

---

### 1.2 Amazon Bedrock — Titan Text Embeddings V2

Used for few-shot example retrieval (OpenSearch vector index of canonical extractions).

| Attribute | Value |
|---|---|
| Model ID | `amazon.titan-embed-text-v2:0` |
| Context window | 8K tokens (max 50K chars input) |
| Output | 1024-dim vector (configurable down to 256 or 512) |
| Lifecycle | Active |
| Endpoints | In-Region only (no Geo, no Global) |
| Batch | Not supported |

**Pricing (us-east-1, on-demand)**

| Dimension | Price |
|---|---:|
| Input tokens | `$0.02 per 1M tokens` (= `$0.00002 / 1K tokens`) |

Output (the embedding vector itself) is not billed separately — it's bundled into the input charge.

**Sources**
- Titan Text Embeddings V2 model card: https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-amazon-titan-text-embeddings-v2.html
- Bedrock pricing page: https://aws.amazon.com/bedrock/pricing/

---

### 1.3 Amazon Textract

OCR + optional structured extraction. **The single highest cost-cliff in the platform.** Choosing the wrong API tier can blow the per-100-page budget by an order of magnitude before any LLM token cost.

**API options & pricing (us-east-1, first 1M pages/month)**

| API | Price per 1,000 pages | Price per page | Volume tier (after 1M pages/month) |
|---|---:|---:|---|
| `DetectDocumentText` (raw OCR) | `$1.50` | `$0.0015` | `$0.60 / 1K pages` (= `$0.0006/page`) |
| `AnalyzeDocument` — **Forms** only | `$50.00` | `$0.050` | `$40.00 / 1K pages` |
| `AnalyzeDocument` — **Tables** only | `$15.00` | `$0.015` | `$10.00 / 1K pages` |
| `AnalyzeDocument` — **Queries** only | `$15.00` | `$0.015` | `$15.00 / 1K pages` |
| `AnalyzeDocument` — **Signatures** only | `$3.50` | `$0.0035` | `$1.40 / 1K pages` |
| `AnalyzeDocument` — **Layout** only | free **when combined with Tables**; otherwise priced as Tables | — | — |
| `AnalyzeDocument` — **Forms + Tables** | `$65.00` | `$0.065` | `$50.00 / 1K pages` |
| `AnalyzeDocument` — **Tables + Queries** | `$20.00` | `$0.020` | `$15.00 / 1K pages` |
| `AnalyzeDocument` — **Forms + Queries** | `$55.00` | `$0.055` | — |
| `AnalyzeDocument` — **Forms + Tables + Queries** | `$70.00` | `$0.070` | `$55.00 / 1K pages` |
| `AnalyzeDocument` — **Custom Queries** | `$25.00` | `$0.025` | `$15.00 / 1K pages` |
| `AnalyzeDocument` — **Pretrained Forms + Custom Queries** | `$65.00` | `$0.065` | `$50.00 / 1K pages` |
| `AnalyzeExpense` | `$10.00` | `$0.010` | `$8.00 / 1K pages` |
| `AnalyzeID` | `$25.00` per 1K (first 100K) | `$0.025` | `$10.00 / 1K pages` after 100K |
| `AnalyzeLending` | `$70.00` | `$0.070` | `$55.00 / 1K pages` after 1M |

**Synchronous vs. asynchronous APIs** — priced identically. The architecture uses **async only** for operational reasons (Step Functions `.waitForTaskToken` integration), not cost reasons.

**Per-page cost shape for a 100-page document**

| Pipeline | Per-100-page Textract cost | Notes |
|---|---:|---|
| Raw OCR only (`DetectDocumentText`) | `$0.15` | Fits the $0.10 budget only if LLM is near-free |
| OCR + Tables | `$1.50` | Already **15× over** the per-doc budget |
| OCR + Forms | `$5.00` | Out of the question for the cost-controlled path |
| OCR + Forms + Tables + Queries | `$7.00` | Reserve for premium tier only |

> ⚠ **Architectural implication.** The cost-controlled default path uses **`DetectDocumentText` (raw OCR) only**. Structured Textract features (Forms/Tables/Queries) are a **per-tenant opt-in** for premium tiers because they consume the entire $0.10/100-page budget on Textract alone. The LLM does the structured extraction in the default path.

**Free tier**

| API | Free pages/month (first 3 months) |
|---|---:|
| `DetectDocumentText` | 1,000 |
| `AnalyzeDocument` Forms/Tables/Layout | 100 |
| `AnalyzeDocument` Queries combos | 100 |
| `AnalyzeDocument` Signatures | 1,000 |
| `AnalyzeExpense` | 100 |
| `AnalyzeID` | 100 |
| `AnalyzeLending` | 2,000 |

**Source**
- https://aws.amazon.com/textract/pricing/

---

### 1.4 Amazon Comprehend — PII Detection (`DetectPiiEntities`)

Used for PII detection/redaction in audit/log paths and (optionally) for output redaction.

**Pricing (us-east-1)**

| Volume tier | Price per unit (= 100 characters) |
|---|---:|
| 0 – 10M units | `$0.0001` |
| 10M – 50M units | `$0.00005` |
| 50M – 100M units | `$0.000025` |
| Over 100M units | volume-discounted further |

**Billing dimensions**

- 1 unit = 100 characters.
- **3-unit (300-character) minimum per request.**
- Tiering is by `Inference Units` (IU) consumed within the month.

**`ContainsPii` vs. `DetectPiiEntities`** — `ContainsPii` (screening only) is orders of magnitude cheaper per call. Use `ContainsPii` as a router; only escalate to `DetectPiiEntities` for full location/redaction when PII is detected.

**Cost shape for the architecture**

For a 100-page document averaging 2,000 chars/page = 200,000 chars = 2,000 units → cost = `$0.20`. This is **2× the entire $0.10 per-100-page budget** if applied naïvely to every output. So PII detection runs only on **redaction-mode outputs** (an explicit tenant flag) and on **audit-bound log payloads after structured filtering**, not on every chunk's raw OCR.

**Source**
- https://aws.amazon.com/comprehend/pricing/

---

## 2. Compute & orchestration

### 2.1 AWS Lambda

The default compute for validation, chunking, merge, webhook delivery, SFN integration, and short-lived steps.

**Pricing (us-east-1, on-demand)**

| Architecture | Per request | Per GB-second (first 6B GB-s/month) | Per GB-second (next 9B) | Per GB-second (over 15B) |
|---|---:|---:|---:|---:|
| x86 | `$0.20 / 1M requests` | `$0.0000166667` | `$0.000015` | `$0.000013334` |
| arm64 (Graviton) | `$0.20 / 1M requests` | `$0.0000133334` (~20% cheaper than x86) | tiered similarly | tiered similarly |

**Provisioned Concurrency**: extra fee for kept-warm capacity — `$0.0000041667/GB-second` plus the per-request fee. Use **selectively** on the submission Lambda and webhook delivery Lambda where cold-start latency matters; do not apply broadly.

**Free tier (permanent)**: 1M requests/month + 400,000 GB-seconds/month.

**Architectural choice**: default to **arm64 / Graviton** for everything that isn't blocked by a native dep. ~20% cost reduction and slightly better cold-start.

**Source**
- https://aws.amazon.com/lambda/pricing/

---

### 2.2 AWS Step Functions

Parent orchestration uses **Standard workflows** (long-running, audit-rich, durable). Distributed Map child iterations use **Express workflows** (high-volume, short-duration).

**Standard workflows (us-east-1)**

| Dimension | Price |
|---|---:|
| Per state transition | `$0.000025` (= `$25 / 1M transitions`) |

Free tier: 4,000 state transitions/month.

**Express workflows (us-east-1)**

| Dimension | Price |
|---|---:|
| Per request (execution started) | `$1.00 / 1M` |
| Per GB-second duration | `$0.00001667` (rounded up to nearest 100ms, billed in 64-MB chunks) |

**Distributed Map** — billing depends on the child workflow type (Standard vs Express). For per-chunk fan-out at 10K docs/day × ~5 chunks/doc avg = 50K child executions/day:

- If children are Express (recommended): cheap. 50K × $1/1M = `$0.05/day` for requests + duration.
- If children are Standard: each transition is $25/1M. A typical child workflow has 5–10 transitions → 250K–500K transitions/day → `$6–$12.50/day`. Acceptable but materially higher.

**Architectural choice**: parent = Standard (durability + audit), Distributed Map children = Express (volume).

**Source**
- https://aws.amazon.com/step-functions/pricing/

---

### 2.3 Amazon API Gateway

Tenant-facing API.

**REST API (us-east-1)**

| Tier | Price per million requests |
|---|---:|
| First 333M / month | `$3.50` |
| Next 667M | `$2.80` |
| Next 19B | `$2.38` |
| Over 20B | `$1.51` |
| Highest tier (low-published) | down to `~$0.90` |

**HTTP API (us-east-1)**

| Tier | Price per million requests |
|---|---:|
| First 300M | `$1.00` |
| Over 300M | `$0.90` |

HTTP API is ~71% cheaper than REST.

**WebSocket** (not in scope for v1).

**Architectural choice**: HTTP API for tenant-facing endpoints unless we need REST-only features (request validation by JSON Schema, full IAM/Cognito custom authorizers with multiple methods, API keys with usage plans). At our volume — 10K submissions/day = ~300K/month — the absolute cost difference is small; the choice should be on features. **Tentative: HTTP API** with a Lambda authorizer for tenant auth, falling back to REST only if a specific feature requires it.

**Plus**: data transfer out at standard egress (see §6.2).

**Source**
- https://aws.amazon.com/api-gateway/pricing/

---

### 2.4 Amazon SQS

Used for chunk queues, webhook queues, and DLQs.

**Pricing (us-east-1)**

| Queue type | Per million requests |
|---|---:|
| Standard | `$0.40` |
| FIFO | `$0.50` (25% premium) |

**Critical billing dimension**: a "request" includes every send, receive, delete, and `ChangeMessageVisibility`. With long polling, batch operations of up to 10 messages each count as 1 receive request. **Use batched operations religiously.**

**Payload-size pricing**: messages over 64 KB are billed in 64-KB chunks. A 256-KB message = 4 billable requests. **Architectural implication**: do not put chunk OCR results in SQS payloads — put S3 pointers in the message body.

**Free tier**: 1M requests/month, permanent.

**Architectural choice (vs. artifact-1's FIFO-everywhere)**: per feedback, use **Standard SQS** for high-throughput chunk dispatch and rely on token-bucket admission control + Distributed Map's native ordering for fairness. Reserve FIFO for the webhook queue where per-tenant ordering matters.

**Source**
- https://aws.amazon.com/sqs/pricing/

---

### 2.5 Amazon EventBridge

Custom event bus for audit fan-out and DDB-stream → SQS plumbing via Pipes.

**Pricing (us-east-1)**

| Component | Price |
|---|---:|
| Custom event bus (events published) | `$1.00 / 1M events` |
| EventBridge Pipes (events processed) | `$0.40 / 1M events` |
| EventBridge Scheduler | covered separately if used |

**Critical billing dimension**: **EventBridge bills in 64-KB chunks of payload.** A 1 MB event = 16 billable events. **Architectural implication**: keep audit events small (key fields only, no document text) and put any large payload reference as an S3 pointer.

**Source**
- https://aws.amazon.com/eventbridge/pricing/

---

## 3. Storage & data services

### 3.1 Amazon S3

Buckets: `input` (uploads), `intermediate` (chunked artifacts, ephemeral), `output` (results), `audit` (immutable).

**Storage classes (us-east-1, per GB-month)**

| Class | Price/GB-month | Min storage duration | Min object size billed |
|---|---:|---:|---:|
| Standard | `$0.023` | None | Actual |
| Standard-IA | `$0.0125` | 30d | 128 KB |
| Glacier Instant Retrieval | `$0.004` | 90d | 128 KB |
| Glacier Flexible Retrieval | `$0.0036` | 90d | 40 KB |
| Glacier Deep Archive | `$0.00099` | 180d | 40 KB |

**Request pricing (us-east-1)**

| Class | PUT/COPY/POST/LIST (per 1K) | GET/SELECT (per 1K) |
|---|---:|---:|
| Standard | `$0.005` | `$0.0004` |
| Standard-IA | `$0.01` | `$0.001` |
| Glacier Instant | `$0.02` | `$0.01` |
| Glacier Flexible (request + retrieval) | `$0.03` (and retrieval fees) | varies |
| Glacier Deep Archive | `$0.05` | `$0.10` per 1K + retrieval |

**Transfer Acceleration**

- Additional `$0.04 / GB` on top of normal data-transfer pricing for accelerated uploads via CloudFront's edge.
- Required by the case study for the 300-MB upload SLA.

**Object Lock**

- **No additional charge** for enabling Object Lock. You pay for the storage of versions (Versioning must be enabled for Object Lock). For audit bucket with compliance-mode 7y retention, expect to pay for the full 7 years of accumulated audit volume.

**S3 Inventory, S3 Storage Lens, S3 Replication**: priced separately; not in v1 scope.

**Architectural lifecycle (drives storage cost)**

| Bucket | Class | Lifecycle |
|---|---|---|
| `input` | Standard → IA at 30d → Glacier Flexible at 90d → delete at retention | 365d default |
| `intermediate` | Standard | delete at 7d |
| `output` | Standard → IA at 30d → delete at retention | 365d default |
| `audit` | Standard-IA from day 1 (rarely read, but cheaper at-rest) → Glacier IR at 90d → Deep Archive at 1y | 7y compliance-locked |

**Source**
- https://aws.amazon.com/s3/pricing/

---

### 3.2 Amazon DynamoDB

`JobStateTable`, `IdempotencyTable`, `TenantConfigTable`, `TenantQuotaTable`, `ChunkStateTable`, `ModelRegistryTable`.

**On-demand pricing (Standard table class, us-east-1)** — reflects the Nov 2024 50% reduction:

| Dimension | Price |
|---|---:|
| Write request unit (WRU, 1 KB write) | `$0.625 / 1M` |
| Read request unit (RRU, strongly-consistent ≤4 KB read) | `$0.125 / 1M` |
| Eventually-consistent read (½ RRU) | effectively `$0.0625 / 1M` |
| Transactional read | `$0.25 / 1M` (2× RRU) |
| Transactional write | `$1.25 / 1M` (2× WRU) |
| Storage (Standard) | `$0.25 / GB-month` |

**Standard-IA table class** (use for `JobStateTable` if storage > 50% of cost):

| Dimension | Price |
|---|---:|
| WRU | `$0.78 / 1M` |
| RRU | `$0.155 / 1M` |
| Storage | `$0.10 / GB-month` |

**DynamoDB Streams**

| Dimension | Price |
|---|---:|
| Streams read request | `$0.02 / 100K requests` (= `$0.20 / 1M`) |

**Important free**: GetRecords invoked by **Lambda triggers on DDB Streams is NOT billed** (the most common pattern). EventBridge Pipes consuming from a DDB Stream **is** billed as Streams reads at the rate above.

**Continuous backup (PITR)**: `$0.20 / GB-month`.
**On-demand backup**: `$0.10 / GB-month`.
**Restore**: `$0.15 / GB`.

**Free tier (permanent)**:
- 25 GB storage (Standard table class)
- 2.5M stream read requests/month

**Architectural implication**: read-heavy workflows (per-chunk state checks) should use eventually-consistent reads where safe — halves the cost.

**Source**
- https://aws.amazon.com/dynamodb/pricing/on-demand/

---

### 3.3 Amazon Data Firehose (Kinesis Firehose)

Audit pipeline: EventBridge → Firehose → S3 (Parquet).

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| Ingestion (first 500 TB / month) | `~$0.029 / GB` |
| Format conversion (JSON → Parquet/ORC) | `$0.018 / GB` ingested |
| VPC delivery (if destination is in VPC) | per-AZ-hour + per-GB |

**Critical billing dimension**: Firehose bills in **5-KB increments**. A 3-KB record is billed as 5 KB; a 12-KB record is billed as 15 KB. **For audit events <5 KB each, every event is billed as 5 KB.** At 10K docs × ~10 audit events each × 5 KB = 500 MB/day; over a month ~15 GB → $0.44/month for ingestion + ~$0.27/month for Parquet conversion. Trivial.

**Source**
- https://aws.amazon.com/firehose/pricing/

---

### 3.4 Amazon OpenSearch Serverless

Vector index for few-shot example retrieval.

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| OCU (OpenSearch Compute Unit) — indexing + search | `$0.24 per OCU-hour` |
| Managed storage | `$0.024 / GB-month` |

**Minimum capacity**: 2 OCUs total (1 indexing + 1 search) at half-OCU resolution. Workloads with low query rates can run on 0.5 OCU each = 1 OCU baseline.

**Minimum monthly cost**: 2 OCU × 24 × 30 × $0.24 = `~$346/month` floor, even with zero queries. For a single-collection vector workload at our scale, expect `~$350–$700/month` floor before storage.

**Vector search collections** require **dedicated OCUs** — they cannot share OCUs with non-vector collections. This is a hard floor.

> ⚠ **Architectural implication.** OpenSearch Serverless has a steep fixed floor relative to the rest of the stack. For low-tenant-count v1, consider **OpenSearch managed (single-AZ, smallest t3.small.search instance ~$25/month)** for the vector workload, or **pgvector on Aurora Serverless v2** for the few-shot store. Document this as a v1 cost optimization to evaluate; for now, model the OSS floor at `~$700/month` to be conservative.

**Source**
- https://aws.amazon.com/opensearch-service/pricing/

---

### 3.5 Amazon ElastiCache Serverless (Valkey)

Per-tenant cluster-wide concurrency counters (used in hybrid; in managed v1, could replace DDB-based per-second rate limit counters if RCU cost exceeds the cache cost).

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| Storage (Valkey) | `$0.084 / GB-hour` (= `~$60.50 / GB-month`) |
| ECPU (compute) | `$0.0023 / 1M ECPUs` |

**ECPU consumption**: 1 ECPU per KB transferred for a simple `GET`/`SET`. A 100-byte INCR ≈ 1 ECPU. At 100 req/s sustained ≈ 260M ECPU/month ≈ `~$0.60/month` of ECPU.

**Valkey vs. Redis OSS minimum storage**:
- Valkey: 100 MB/cache minimum → ~`$6/month` floor.
- Redis OSS: 1 GB/cache minimum → ~`$60/month` floor.

**Architectural implication**: if used, **Valkey** for the 10× lower floor.

> ⚠ **Note on pricing precision.** ElastiCache Serverless storage pricing has been published as both `$0.084 GB-hour` and `$0.125 GB-hour` across sources. Treat as `$0.084 GB-hour` for the architecture and verify against the live console before any commitment.

**Source**
- https://aws.amazon.com/elasticache/pricing/

---

## 4. Security & identity

### 4.1 AWS KMS

Per-tenant Customer Managed Keys (CMKs) for input/intermediate/output buckets; a separate platform-managed CMK for the audit bucket.

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| Per CMK | `$1.00 / month` (prorated hourly) |
| API requests | `$0.03 / 10K requests` (= `$3 / 1M`) |
| Multi-Region keys | `$1.00 per primary + $1.00 per replica` |

**Free tier**: 20,000 requests/month, symmetric operations only.

**Encryption context**: free to use, and the architecture **must** use it — for tenant isolation, include `tenant_id` in the encryption context on every encrypt/decrypt, and enforce it in IAM key policies.

**Architectural cost shape**

At 500 tenants × $1/CMK + 1 platform audit CMK = **`~$501/month` baseline** just for keys.
At 200K KMS calls/day (≈ chunked OCR/extraction × encrypt + decrypt cycles) × 30d = 6M requests = `~$18/month`.

**Source**
- https://aws.amazon.com/kms/pricing/

---

### 4.2 AWS Secrets Manager

API HMAC keys for tenant webhooks; Bedrock model API tokens; any RDS credentials if Aurora ends up in the picture.

**Pricing (all regions including us-east-1)**

| Dimension | Price |
|---|---:|
| Per secret | `$0.40 / month` |
| API calls | `$0.05 / 10K calls` |

**Architectural cost shape**

Per tenant ~3 secrets (webhook HMAC, API key set, integration token) → 500 tenants × 3 × $0.40 = **`~$600/month`**.

**Mitigation**: use **SSM Parameter Store SecureString** for non-rotating items — `$0.05 / 10K API calls`, free per-parameter — and reserve Secrets Manager for items that need automatic rotation. Cuts the per-tenant secrets footprint substantially.

**Source**
- https://aws.amazon.com/secrets-manager/pricing/

---

### 4.3 AWS WAF

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| Per Web ACL | `$5.00 / month` (prorated hourly) |
| Per rule | `$1.00 / month` (prorated hourly) |
| Requests evaluated | `$0.60 / 1M requests` |
| Additional WCU (per 500 over 1500 default) | `+$0.20 / 1M requests` |
| Body inspection > 16 KB | `+$0.30 / 1M requests` for each extra 16 KB |

**Architectural cost shape**

1 Web ACL + ~10 managed rule groups + ~5 custom rules: `$5 + $15 = $20/month` fixed, plus `~$0.60 × 0.3M = $0.18/month` variable at our submission volume. Trivial.

**Source**
- https://aws.amazon.com/waf/pricing/

---

## 5. Observability

### 5.1 Amazon CloudWatch — Logs

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| Standard log ingestion | `$0.50 / GB` |
| Infrequent Access (IA) log ingestion | `$0.25 / GB` |
| Lambda logs (tiered) | starts `$0.50 / GB`, tiers down to `$0.05 / GB` at very high volume |
| Storage (compressed) | `$0.03 / GB-month` |
| Logs Insights query | `$0.005 / GB scanned` |
| Live Tail | `$0.01 / minute / session` |

**Architectural implication**: log lines have a cost. The default application log payload **must not include OCR'd text or extracted PII**. Structured logging with explicit field allowlists is a cost and a compliance control simultaneously.

At 200 GB/month ingestion: `$100/month`. Cost grows quickly with debug-mode logging.

**Source**
- https://aws.amazon.com/cloudwatch/pricing/

---

### 5.2 Amazon CloudWatch — Metrics & Alarms

**Custom metrics (tiered, us-east-1)**

| Volume tier | Per metric / month |
|---|---:|
| First 10K | `$0.30` |
| Next 240K | `$0.10` |
| Next 750K | `$0.05` |
| Over 1M | `$0.02` |

**Alarms**

| Type | Per alarm metric / month |
|---|---:|
| Standard resolution (60s) | `$0.10` |
| High resolution (10s) | `$0.30` |
| Composite alarm | `$0.50` |

**`PutMetricData` API calls**: `$0.01 / 1,000 calls` beyond the free tier.

**Architectural implication**: dimensional explosion is the killer. Per-tenant per-stage per-status metrics for 500 tenants × 7 stages × 4 statuses = 14,000 metrics → `~$3,400/month` just for metric storage. **Mitigation**: use **EMF (embedded metric format)** to extract metrics from structured log events; pay logs cost, not metrics cost.

**Source**
- https://aws.amazon.com/cloudwatch/pricing/

---

### 5.3 AWS X-Ray

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| Traces recorded | `$5.00 / 1M traces` |
| Traces retrieved | `$0.50 / 1M traces` |
| Traces scanned (query) | `$0.50 / 1M traces` |

**Free tier**: 100K traces recorded/month + 1M traces retrieved + 1M scanned.

**Sampling**: the architecture must use sampling rules — at 10K docs/day × 7 stages = 70K spans/day = 2.1M spans/month = `$10.50/month`. Without sampling, the per-chunk fanout multiplies this by ~5×.

**Source**
- https://aws.amazon.com/xray/pricing/

---

### 5.4 Amazon Managed Service for Prometheus (AMP)

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| Metric samples ingested (first 2B samples / month) | `$0.90 / 10M samples` |
| Metric samples ingested (over 2B / month) | tiered down to `$0.16 / 10M samples` |
| Query samples processed (QSP) | `$0.10 / 1B query samples` |
| Storage | `$0.03 / GB-month` |

**Architectural cost shape**: at 1K active series × 60s scrape interval × 30d = ~43M samples/month = `~$3.90/month` ingestion. Cheap for an MLOps stack, especially compared to CW custom metrics at our cardinality.

**Architectural implication**: prefer AMP over CloudWatch for high-cardinality MLOps metrics (per-tenant per-job token counts, TTFT histograms, etc.). Use CloudWatch for service-level metrics where AWS already emits them for free.

**Source**
- https://aws.amazon.com/prometheus/pricing/

---

### 5.5 Amazon Managed Grafana (AMG)

**Pricing (us-east-1, all regions same)**

| User license | Per active user / month |
|---|---:|
| Admin / Editor | `$9.00` |
| Viewer | `$5.00` |

**"Active user"**: any user that logged in or made an API call at least once in the billing month. **Inactive users in a workspace are NOT billed.**

**Architectural cost shape**: 5 Admins + 5 Editors + 20 Viewers = `$190/month` at full active. Trivial.

**Source**
- https://aws.amazon.com/grafana/pricing/

---

### 5.6 Amazon Athena

**Pricing (us-east-1)**

| Dimension | Price |
|---|---:|
| Per TB scanned | `$5.00` |
| Minimum per query | 10 MB equivalent |

**Cost-control levers** (compounding):
- **Parquet/ORC + columnar projection** → up to 96% scan reduction.
- **Partitioning** (year/month/day/tenant_id in our audit bucket) → query only relevant partitions.
- **Compression** (Snappy/Zstd on Parquet) → additional 2–4× reduction.

**Architectural implication**: audit pipeline writes Parquet partitioned `yyyy/mm/dd/tenant_id`. A per-tenant audit query for one day's events at our volume scans <100 MB → `~$0.0005/query`. Essentially free.

**Source**
- https://aws.amazon.com/athena/pricing/

---

### 5.7 AWS Lake Formation

**No service-side charge.** Lake Formation governs access to S3/Glue/Athena resources; you pay only the underlying AWS service costs (S3 storage, Athena scans, Glue jobs). Row-level filtering for per-tenant audit access — free in terms of LF charges; the Athena scans are still billed.

**Source**
- https://aws.amazon.com/lake-formation/pricing/

---

## 6. Networking & data transfer

### 6.1 Data Transfer In

**Free** — AWS does not charge for data transferred into AWS services.

### 6.2 Data Transfer Out (Egress)

**Pricing (us-east-1 → Internet)**

| Tier | Price/GB |
|---|---:|
| First 100 GB / month (across all AWS) | Free |
| Next 9.9 TB | `$0.09` |
| Next 40 TB | `$0.085` |
| Next 100 TB | `$0.07` |
| Over 150 TB | `$0.05` |

**Architectural implication**: result payloads (final JSON) typically ≤100 KB → trivial egress cost. The cost lever is **avoid serving full PDFs over webhook** — use presigned URLs (egress is paid when the tenant fetches, but still much cheaper than streaming via Lambda + API Gateway).

### 6.3 NAT Gateway

If used for outbound from private subnets:
- Per NAT GW per hour: `$0.045`
- Per GB processed: `$0.045`

**Architectural choice**: use **VPC Endpoints (Gateway endpoints for S3/DDB are free; Interface endpoints `$0.01/AZ-hour + $0.01/GB`)** rather than NAT. For managed-path v1 with mostly Lambda compute, Lambda runs outside our VPC by default → no NAT needed for AWS API calls.

### 6.4 VPC Interface Endpoints (PrivateLink)

- `$0.01 / AZ / hour` per endpoint (so a 3-AZ deployment = `$0.03/hour` = `~$22/month` per endpoint).
- `$0.01 / GB` data processed.

**Architectural implication**: only stand up Interface endpoints for services we'd otherwise route via NAT/Internet. For the managed path, **S3 Gateway endpoint is free** and is the primary need.

**Source**
- https://aws.amazon.com/ec2/pricing/on-demand/ (data transfer section)
- https://aws.amazon.com/privatelink/pricing/

---

## 7. Cost-defensible per-100-page math (sanity check)

Plugging the unit prices above into a 100-page document on the **default cost-controlled path** (managed Bedrock Llama 3.1 8B, geo endpoint, raw OCR only, few-shot retrieval enabled):

**Assumptions** (these are the per-doc token budget per the feedback's recommendation):

| Knob | Value |
|---|---:|
| Pages | 100 |
| Pages routed to OCR (digital text bypasses) | 40 |
| Textract API | `DetectDocumentText` only |
| Input tokens to LLM | 40K |
| Output tokens from LLM | 4K aggregate across chunks |
| Few-shot embedding input tokens | 1K (one query per doc) |

**Cost breakdown**

| Line | Calculation | Cost |
|---|---|---:|
| Textract OCR | 40 pages × $0.0015 | `$0.060` |
| Bedrock Llama 3.1 8B input | 40K × $0.22/M | `$0.0088` |
| Bedrock Llama 3.1 8B output | 4K × $0.22/M | `$0.00088` |
| Titan embedding | 1K × $0.02/M | `$0.00002` |
| Step Functions Express (1 parent + ~5 chunk children, ~10 transitions equiv) | ~$0.000005 per doc | `$0.00001` |
| Lambda + SQS + DDB + S3 (per-doc overhead) | amortized | `~$0.001` |
| **Total per 100-page doc** | | **`~$0.071`** |

**Per 100 pages**: `~$0.071` using the us-east-1 Llama 3.1 8B rate — **under the $0.10 target, but still dependent on representative mix and routing.** Realistic risks such as scanned-heavy documents, longer prompts, stronger-model recovery, retry storms, or structured Textract calls can push this over. The architecture artifact must:

1. Document the assumptions above explicitly.
2. Treat the budget as **conditional on routing, token discipline, and representative document mix**.
3. Add a stop-loss control (mid-job estimated-cost threshold; abort or downgrade if exceeded).

> ⚠ **Batch mode** for Llama 3.1 8B is half price (`$0.11/$0.11 per 1M tokens`) but should only be used for non-latency-sensitive replays, backfills, or offline evaluation. The interactive pipeline cannot rely on batch mode for the 5-minute P95 SLA.

> ⚠ **If Stage 4 escalates broadly to Llama 3.1 70B**, the model line can increase materially. The 70B path is therefore reserved for critical recovery and is non-billable to tenants in v1.

> ⚠ **If we use Textract `Forms + Tables` on those 40 pages instead of raw OCR**, the Textract line alone becomes `40 × $0.065 = $2.60/100 pages` — `~30× over the budget`. This is the hardest cost cliff in the platform.

---

## 8. Service-by-service summary (one-line reference)

**Validation note:** spot-checked on May 15, 2026. This list is sufficient for the **managed-path data plane** plus core operations, security, and observability. Hybrid EKS/GPU inference remains intentionally out of scope for this managed-path table.

| Service | Headline price | Architectural role | Cost share at 10K docs/day |
|---|---|---|---|
| Bedrock Llama 3.1 8B Instruct | `$0.22 / $0.22 per 1M tokens`; batch `$0.11 / $0.11` | Default schema-constrained extraction | ~10% |
| Bedrock Llama 3.1 70B Instruct | planning `$2.00 / $2.00 per 1M tokens` | Critical-field recovery only | varies |
| Textract `DetectDocumentText` | `$1.50 / 1K pages` | OCR for scanned/hybrid pages | ~25% |
| Titan Embeddings V2 | `$0.02 / 1M tokens` | Few-shot retrieval | <1% |
| Comprehend PII | `ContainsPii: $0.000002 / 100 chars; DetectPii: $0.0001 / 100 chars` | PII routing + redaction (opt-in) | varies |
| Lambda (arm64) | `$0.20 / 1M req + $0.0000133/GB-s` | Compute glue | ~5% |
| Step Functions Standard | `$25 / 1M transitions` | Parent orchestration | ~3% |
| Step Functions Express | `$1 / 1M req + $0.00001667/GB-s` | Distributed Map children | ~2% |
| API Gateway HTTP | `$1 / 1M req` | Tenant-facing API | <1% |
| SQS Standard | `$0.40 / 1M req` | Chunk dispatch | <1% |
| SQS FIFO | `$0.50 / 1M req` | Ordered webhook delivery | <1% |
| EventBridge bus | `$1 / 1M events` | Audit fan-out | <1% |
| EventBridge Pipes | `$0.40 / 1M events` | DDB-Stream → SQS | <1% |
| S3 Standard | `$0.023 / GB-month` | Input/output buckets | ~5% (driven by retention) |
| S3 Transfer Acceleration | `+$0.04 / GB` | 300 MB upload acceleration | varies |
| Route 53 | `$0.50 / hosted zone-month + $0.40 / 1M standard queries` | Public DNS for APIs/custom domains | <1% |
| ACM public certificates | `$0 for non-exportable public certs on integrated AWS services` | TLS certificates for API/custom domains | $0 |
| DynamoDB On-Demand | `$0.625 W / $0.125 R per 1M` | Job state + idempotency | ~5% |
| DynamoDB Streams | `$0.20 / 1M reads` | Change capture (free w/ Lambda trigger) | <1% |
| Firehose | `$0.029 / GB ingested` | Audit pipeline | <1% |
| OpenSearch Serverless | `$0.24 / OCU-hour + $0.02 / GB-month managed storage` | Few-shot vector store | ~10% (fixed floor) |
| ElastiCache Serverless (Valkey) | `$0.084 / GB-hour, $0.0023 / 1M ECPU` | Counters (optional) | <1% |
| KMS | `$1 / CMK / month + $3 / 1M req` | Per-tenant encryption | scales with tenant count |
| Secrets Manager | `$0.40 / secret / month` | Webhook secrets, model keys | scales with tenant count |
| WAF | `$5 ACL + $1/rule + $0.60 / 1M req` | API protection | <1% |
| PrivateLink / VPC endpoints | `$0.01 / endpoint-AZ-hour + $0.01 / GB` | Private AWS service access where required | depends on endpoint count |
| CloudTrail trails | `1 management-event copy free; data events $0.10 / 100K` | AWS API audit trail for SOC 2/GDPR evidence | varies |
| GuardDuty (optional) | `CloudTrail mgmt analysis $4 / 1M events; VPC/DNS logs $1 / GB first 500 GB` | Threat detection/security monitoring | varies |
| GuardDuty Malware Protection for S3 | `$0.09 / GB scanned + $0.215 / 1K objects evaluated` | Malware scan gate for uploaded documents | varies with upload size |
| CloudWatch Logs | `$0.50 / GB ingest + $0.03 / GB-month` | Operational logs | ~5% (volume-driven) |
| CloudWatch Custom Metrics | `$0.30 / metric / month` (then tiered) | Service-level metrics | depends on cardinality |
| CloudWatch Alarms | `$0.10 / alarm` | Alerts | <1% |
| X-Ray | `stored: $5 / 1M traces; scanned/retrieved: $0.50 / 1M traces` | Distributed tracing | <1% (with sampling) |
| Amazon Managed Prometheus | `$0.90 / 10M samples` | MLOps metrics | <1% |
| Amazon Managed Grafana | `$9 / Admin or $5 / Viewer per active month` | Dashboards | <1% |
| Athena | `$5 / TB scanned` | Audit queries | <1% |
| Lake Formation | $0 service charge | Audit row-level filtering | $0 |
| Data Transfer Out (Internet) | `$0.09 / GB` after free 100 GB | Egress | <1% |

---

## 9. Open questions to resolve before pricing the final artifact

1. **Exact Bedrock Llama 3.1 70B price.** The architecture now uses Llama 3.1 8B/70B. The 8B rate has been supplied from the us-east-1 Bedrock pricing view; the 70B recovery rate in Section 1.1 remains a planning estimate and must be confirmed before publishing final recovery-cost claims.
2. **OpenSearch Serverless alternative.** If we don't need the serverless scaling, OpenSearch managed (single-AZ) or pgvector on Aurora Serverless v2 could replace the OSS floor of ~$350+/month. Flag for v1 cost-trim decision.
3. **DynamoDB Standard vs. Standard-IA for JobStateTable.** With 365-day retention, storage may dominate WRU/RRU cost. Run the math against expected average job-row size (~5 KB) at 10K docs/day × 365d = 3.65M rows × 5 KB = 18 GB → still small enough that Standard wins on read pricing. Standard-IA only wins above ~100 GB.
4. **Egress to tenant webhooks.** We're not paying for tenant API ingress, but webhook delivery + result retrieval are paid on our side. Confirm the result-fetch egress is at standard rates (which is true for S3 presigned URLs from our buckets).

---

## 10. Citations

- Bedrock Llama 3.1 8B model card — https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-meta-llama-3-1-8b-instruct.html
- Bedrock Llama 3.1 70B model card — https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-meta-llama-3-1-70b-instruct.html
- Bedrock Claude Haiku 4.5 model card — https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-4-5.html
- Bedrock Titan Text Embeddings V2 model card — https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-amazon-titan-text-embeddings-v2.html
- Bedrock pricing — https://aws.amazon.com/bedrock/pricing/
- Anthropic platform pricing — https://platform.claude.com/docs/en/about-claude/pricing
- Textract pricing — https://aws.amazon.com/textract/pricing/
- Comprehend pricing — https://aws.amazon.com/comprehend/pricing/
- Lambda pricing — https://aws.amazon.com/lambda/pricing/
- Step Functions pricing — https://aws.amazon.com/step-functions/pricing/
- API Gateway pricing — https://aws.amazon.com/api-gateway/pricing/
- SQS pricing — https://aws.amazon.com/sqs/pricing/
- EventBridge pricing — https://aws.amazon.com/eventbridge/pricing/
- S3 pricing — https://aws.amazon.com/s3/pricing/
- DynamoDB on-demand pricing — https://aws.amazon.com/dynamodb/pricing/on-demand/
- Data Firehose pricing — https://aws.amazon.com/firehose/pricing/
- OpenSearch Service pricing — https://aws.amazon.com/opensearch-service/pricing/
- ElastiCache pricing — https://aws.amazon.com/elasticache/pricing/
- KMS pricing — https://aws.amazon.com/kms/pricing/
- Secrets Manager pricing — https://aws.amazon.com/secrets-manager/pricing/
- WAF pricing — https://aws.amazon.com/waf/pricing/
- CloudWatch pricing — https://aws.amazon.com/cloudwatch/pricing/
- X-Ray pricing — https://aws.amazon.com/xray/pricing/
- Amazon Managed Prometheus pricing — https://aws.amazon.com/prometheus/pricing/
- Amazon Managed Grafana pricing — https://aws.amazon.com/grafana/pricing/
- Athena pricing — https://aws.amazon.com/athena/pricing/
- Lake Formation pricing — https://aws.amazon.com/lake-formation/pricing/
- PrivateLink pricing — https://aws.amazon.com/privatelink/pricing/
- CloudTrail pricing — https://aws.amazon.com/cloudtrail/pricing/
- GuardDuty pricing — https://aws.amazon.com/guardduty/pricing/
- Route 53 pricing — https://aws.amazon.com/route53/pricing/
- AWS Certificate Manager pricing — https://aws.amazon.com/certificate-manager/pricing/
