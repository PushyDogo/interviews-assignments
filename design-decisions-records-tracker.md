# Design Decisions Records Tracker

Major architectural and cross-cutting decisions only. Implementation-level choices belong in the stage design docs.

---

## Stage 1 — Ingestion

| # | Decision | Choice | Reason | Alternative Considered |
|---|---|---|---|---|
| D1 | Upload mechanism | Direct S3 presigned POST — Lambda is not in the data path | Avoids 6 MB Lambda payload limit; eliminates unnecessary compute cost and latency on the hot path. | API Gateway + Lambda proxy — ruled out due to payload size limit. |
| D2 | Upload validation trigger | S3 `ObjectCreated` event → Validation Lambda | Simpler customer integration; no extra API call required after upload. | Client-side `POST /jobs/{id}/complete-upload` finalization — deferred to v2. |
| D3 | Malware scan gate | GuardDuty Malware Protection for S3 gates acceptance before quota is consumed | Prevents malicious files from entering the OCR/LLM pipeline; quota is not consumed for rejected files. | ClamAV on Lambda/ECS — viable fallback if GuardDuty is unavailable in the target region. |
| D4 | Customer authentication | API key + signed short-lived JWT (dual factor) | API key identifies the tenant/application; JWT carries authorization claims and scope. Single-factor API key is insufficient for a multi-tenant platform handling sensitive documents. | API key only — insufficient; OAuth client credentials — possible v2 path. |
| D5 | Document encryption | Per-tenant KMS CMK | Gives each tenant cryptographic isolation, not only IAM isolation. Enables GDPR crypto-shredding on erasure. | Single platform CMK — simpler but provides only IAM-level tenant boundary. |
| D6 | Immutable audit storage | EventBridge → Firehose → S3 Object Lock (compliance mode) | Provides tamper-resistant SOC 2 evidence. Audit records are kept PII-free so they survive GDPR erasure intact. | CloudWatch Logs alone — not immutable and not compliant for SOC 2 evidence. |
| D7 | SLA clock definition | 5-minute P95 clock starts at `accepted_at`, after malware scan and validation complete | Upload time is customer/network-dependent and outside platform control. Measuring from `accepted_at` gives a fair and enforceable SLA. | Clock from first API call or S3 object creation — includes time outside platform control. |
| D8 | API exposure | WAF-protected public API in v1; PrivateLink deferred to v2 | Simpler to operate and onboard customers in v1. Enterprise private-network access added as a tier-based feature in v2. | PrivateLink from day one — over-engineered for v1 unless a specific enterprise customer requires it. |
| D9 | DynamoDB key design | Single-table with high-cardinality `JOB#<job_id>` PK; write-sharded tenant and quota indexes | Avoids hot partitions during 50,000-document burst; distributes writes across the partition space. | Tenant as primary partition key — creates hot partitions during bulk tenant uploads. |

## Stage 2 — Pre-processing

| # | Decision | Choice | Reason | Alternative Considered |
|---|---|---|---|---|
| D10 | Compute path for pre-processing | Lambda for ≤250 pages / ≤100 MB; ECS/Fargate for everything heavier | Lambda cannot safely handle 1,000-page / 300 MB documents within timeout, memory, and /tmp limits. Fargate provides a bounded path without the pretence that Lambda covers the full range. | Lambda only — unsafe at the upper bound; ECS/Fargate only — over-provisioned and slower for small documents. |
| D11 | Chunking strategy | Token-aware chunking (25K soft cap, 30K hard cap) in addition to the 20–50 page count range | Page count alone does not bound LLM context window usage. A 50-page legal contract can exceed a downstream token budget. Token estimation must happen before OCR so Stage 3 can process chunks safely. | Page count only — risks LLM context overflow for dense documents. |
| D12 | Page classification approach | Deterministic heuristics first (text density, dictionary-word ratio, image coverage, rotation); no LLM calls during pre-processing | LLM calls in pre-processing would add latency, cost, and a dependency on inference availability before OCR has even run. Deterministic rules are fast, cheap, and explainable. Ambiguous pages default to OCR in v1. | LLM-based classifier — deferred to v2 once labeled production data is available. |
| D13 | Manifest design | Separate page manifest and chunk manifest written as deterministic S3 artifacts | Stage 3 needs exact per-page routing decisions to know which pages need OCR, which can bypass it, and which are unsupported. Aggregate chunk counts are insufficient. Deterministic keys enable idempotent replay. | Single combined manifest — loses per-page routing granularity needed downstream. |
| D14 | Stage 3 fan-out mechanism | Step Functions Distributed Map over the chunk manifest | Provides elastic chunk-level parallelism with built-in concurrency controls per tenant, preserving fairness during bursts without custom queue management. | SQS fan-out — more flexible but requires custom concurrency management and loses the built-in orchestration state. |

## Stage 3 — OCR and Structure Extraction

| # | Decision | Choice | Reason | Alternative Considered |
|---|---|---|---|---|
| D15 | OCR invocation model | Async Textract with SNS → SQS callback (not synchronous) | Long-running OCR must not hold Lambda execution open. Callbacks can be retried, validated, and durably waited on by Step Functions. | Synchronous Textract — only viable for small single-page inputs; unsafe at chunk scale. |
| D16 | Structured OCR activation | Policy-controlled and schema-gated; text OCR is the default; Forms/Tables enabled only per schema | Form extraction costs `$0.0040/page` vs `$0.0010/page` for text/tables. Enabling it by default for all pages would blow the `$0.10/100-page` budget on Textract alone. | Default structured OCR for all pages — makes the cost target impossible to meet. |
| D17 | Textract `OutputConfig` with tenant CMK | Required on every async Textract call | Without `OutputConfig`, Textract writes raw output outside the tenant-scoped S3 prefix and tenant CMK, breaking the per-tenant cryptographic isolation established in Stage 1. | Omit `OutputConfig` — violates the Stage 1 security decision. |
| D18 | OCR idempotency key | Scoped by `tenant_id + job_id + chunk_id + ocr_unit_id + ocr_policy_version + input_artifact_hash` | Prevents duplicate paid Textract starts on Lambda retry or Step Functions replay. Same key with a different artifact hash or policy version is rejected as a conflict. | Job ID alone — insufficient; would allow different inputs to reuse the same OCR slot. |
| D19 | Single-region Textract in v1 | Primary AWS region only; cross-region failover deferred to v2 | Cross-region failover complicates per-tenant CMK isolation, data residency, quota management, callback routing, and audit evidence. Circuit breaker + backpressure handles outages in v1. | Cross-region failover from day one — over-engineered for v1; deferred until data-residency requirements are clear. |
| D20 | Stage 3 latency budget | P50 ≤60s / P95 ≤180s / P99 ≤240s from `OCR_QUEUED` to terminal chunk state | Leaves ~90 seconds for AI extraction, merge, validation, storage, and notification within the 5-minute P95 SLA. Explicit budget enables admission backpressure before SLO misses become widespread. | No explicit budget — makes it impossible to apply backpressure before the 5-minute target is breached. |

## Stage 4 — AI Extraction

| # | Decision | Choice | Reason | Alternative Considered |
|---|---|---|---|---|
| D21 | Default extraction model | Llama 3.1 8B Instruct on Bedrock (`$0.22/1M` tokens) | Lowest managed cost at acceptable quality. Same model family as self-hosted path — migration to EKS vLLM requires no model change. | Claude Haiku / other models — different family complicates the self-hosted migration path. |
| D22 | Recovery model | Llama 3.1 70B Instruct on Bedrock (`$0.72/1M` tokens) for critical-field failures only | 3.3× more expensive than 8B; must be selective. Automatic for critical fields in v1; non-billable to tenants — treated as platform quality recovery. | Routing all extractions to 70B — blows the cost target. |
| D23 | Structured output contract | Bedrock Converse API tool use (`submit_extraction` tool) | Forces schema-valid JSON; free-form assistant text is rejected. Model must call the tool exactly once. If constrained decoding becomes available, it can reinforce — not replace — this contract. | Regex/parse of free-form output — brittle and harder to audit. |
| D24 | Hallucination control | Every non-null extracted value must have a non-empty `source_citations` array resolving to a valid Stage 3 page/block/span | Grounds extraction in OCR evidence. Values without citations are rejected and marked `not_found`. Prevents the model from satisfying required schema fields by invention. | Post-hoc confidence scores only — doesn't prevent fabricated values. |
| D25 | Few-shot retrieval | Enabled in v1; schema-matched, tenant-scoped, capped at 3 examples and 4K tokens | Improves extraction quality on varied schemas. Tenant-scoped to prevent PII leakage across tenants. | Disabled — simpler but lower quality on complex schemas. |
| D26 | Bedrock Guardrails | Mandatory on all model invocations; includes PII filtering, prompt-injection denial, and denied-topic policy | OCR text is untrusted input and may contain prompt-injection attempts. No invocation may bypass the guardrail without an operator-approved break-glass workflow. | Optional guardrails — creates a bypass surface that could expose PII or allow injection. |
| D27 | Stage 4 latency budget | P50 ≤20s / P95 ≤60s / P99 ≤90s from `EXTRACTION_QUEUED` to terminal chunk state | After Stage 3 consumes up to 180s at P95, Stage 4 must fit within ~60s to leave time for merge, validation, storage, and notification. | No explicit budget — prevents backpressure decisions before the 5-minute SLA is missed. |
