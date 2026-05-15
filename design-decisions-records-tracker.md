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
