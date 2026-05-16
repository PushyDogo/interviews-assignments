# Interview Preparation Guide: High-Throughput Hybrid Document AI Platform

## 1. Opening Talk Track

This architecture is an AWS-native, event-driven Document AI platform for large asynchronous PDF processing. It is designed for documents up to 1,000 pages and 300 MB, with 10,000+ documents per day sustained throughput, 50,000-document burst intake, P95 processing latency under 5 minutes after acceptance, 99.9% availability, and a target cost below $0.10 per 100-page document for a representative document mix.

The main design principle is: use deterministic code wherever possible, and use OCR or language models only where deterministic processing is insufficient. That principle controls cost, latency, quality, and auditability.

The customer never uploads the file through the API runtime. The API creates a job, validates tenant policy, and returns a presigned S3 upload target with Transfer Acceleration. After upload, the platform validates the object, malware-scans it, accepts the job, and starts a Step Functions workflow. The PDF is then profiled, chunked by both page count and token budget, processed in parallel, selectively OCRed, extracted through schema-constrained LLM calls, merged deterministically, validated, redacted, stored, and made available through a result API and pointer-only webhook.

The v1 inference path is managed Bedrock, using Llama 3.1 8B for default extraction and 70B only for critical-field recovery. A self-hosted EKS/vLLM path is included as a future optimization once volume and benchmark data justify the operational cost.

## 2. Assignment Requirements Covered

The case study asks for four artifacts: proposed architecture, operational strategy, security and compliance posture, and risk mitigation. Your design covers all four:

| Requirement area | Design answer |
|---|---|
| Serverless and event-driven | API Gateway, Lambda, S3 events, Step Functions, Distributed Map, SQS, EventBridge, Textract callbacks, and async webhook delivery. |
| Large PDF support | Direct S3 upload, Transfer Acceleration, Fargate path for heavy preprocessing, page and chunk manifests. |
| 20-50 page chunking | Chunking honors page range and token budget, with 25K target and 30K hard input cap. |
| OCR and structure | Digital text bypasses OCR; scanned or hybrid pages use async Textract; Forms/Tables are schema-gated. |
| LLM extraction | Bedrock Converse tool-use produces schema-constrained JSON with required citations. |
| Merging | Schema-defined deterministic merge policies, with bounded synthesis only when required. |
| Post-processing | Deterministic schema validation, normalization, business validation, redaction, and packaging. |
| Storage | Final outputs in S3, job state in DynamoDB, signed URL retrieval, pointer-only webhooks. |
| MLOps | CDK Python, GitHub Actions, 80% unit coverage, CDK synth, model eval, canaries, rollback. |
| Security | WAF, API key + JWT, per-tenant KMS keys, PII-safe logging, Object Lock audit, GDPR erasure. |
| Observability | CloudWatch, X-Ray, cost metrics, token metrics, TTFT, queue age, DLQs, GPU metrics for self-hosted path. |
| Risk | Backpressure, idempotency, poison queues, cost stop-loss, manual inspection, circuit breakers. |

## 3. End-to-End Architecture Walkthrough

### 3.1 API, Authentication, and Admission

The entry point is API Gateway protected by WAF. Authentication uses an API key plus a short-lived JWT. The API key identifies the tenant application; the JWT carries authorization scopes such as document submission, schema access, and upload permission.

The customer calls `POST /v1/jobs`. The platform validates tenant identity, schema selection, quotas, and policy. It creates a durable job record and returns a presigned S3 POST target. The upload is direct to S3 using Transfer Acceleration, so API Gateway and Lambda never sit in the 300 MB file path.

Important decisions:

- Direct S3 upload avoids API Gateway and Lambda payload limits.
- The SLA clock starts at `accepted_at`, not at upload start, because customer network upload time is outside platform control.
- Admission is separated into job creation, pending upload count, accepted-per-minute rate, and active job concurrency.
- Tenant backpressure returns `429` with `Retry-After` instead of accepting unlimited work.

Security points to mention:

- WAF protects edge traffic.
- API key alone is not enough for sensitive multi-tenant workflows, so JWT scopes provide explicit authorization.
- Upload keys are platform-generated and tenant/job-scoped, preventing customer-controlled S3 key paths.

### 3.2 Upload Validation and Malware Gate

After upload, S3 emits an object-created event to a validation Lambda. Validation checks object ownership, checksum, content type, file size, page count, KMS encryption, upload TTL, and GuardDuty Malware Protection result.

Only after validation and malware scanning pass does the job move to `ACCEPTED`, consume quota, and start the processing workflow.

Important decisions:

- Failed uploads do not consume expensive processing quota.
- GuardDuty Malware Protection is used instead of pushing a ClamAV scanner into the data path.
- Validation is idempotent because S3 events can be duplicated.

Tradeoff:

- GuardDuty is simpler and managed, but creates a dependency on AWS feature availability and scanning latency.
- A custom scanner gives more control, but adds patching, signatures, scaling, and operational burden.

### 3.3 State, Storage, and Orchestration

The platform uses S3 for input, intermediate, output, and audit artifacts; DynamoDB for job state and idempotency records; KMS for per-tenant encryption; Step Functions Standard for orchestration; Distributed Map for chunk fan-out; and SQS/EventBridge for buffering and decoupling.

Every stage emits a durable, versioned artifact. Downstream stages consume the previous stage artifact instead of relying on in-memory state. This makes replay possible at the document, chunk, OCR unit, model invocation, merge, post-processing, and webhook levels.

Important decisions:

- Step Functions gives durable lifecycle state and visibility.
- SQS absorbs burst pressure and isolates downstream throttling.
- DynamoDB uses high-cardinality `JOB#<job_id>` primary keys and write-sharded tenant indexes to avoid hot partitions.
- Every async boundary has an idempotency key that includes tenant, job, input artifact hash, and relevant policy/model/schema versions.

### 3.4 Pre-processing

Stage 2 profiles the PDF and creates the page and chunk manifests. Small documents use Lambda; larger documents use ECS/Fargate because 1,000-page or 300 MB PDFs can exceed Lambda timeout, memory, and `/tmp` constraints.

The preprocessor classifies pages as digital-text, scanned, or hybrid. It uses deterministic signals such as text density, image coverage, dictionary-word ratio, language confidence, rotation, skew, and quality. Ambiguous pages go to OCR because missing a critical field is worse than paying for unnecessary OCR.

Chunking is both page-aware and token-aware:

- Page target: 20-50 pages.
- Input token target: 25K.
- Hard input cap: 30K.
- This keeps the LLM well below the 128K context limit after adding schema, instructions, citations, examples, and repair prompts.

MLOps and quality point:

- A lightweight learned classifier is deferred until production-labeled data exists. That is a deliberate maturity decision.

### 3.5 OCR and Structure Extraction

Stage 3 uses the cheapest reliable text source:

- Digital-text pages bypass OCR.
- Scanned or hybrid pages use async Textract.
- Structured Textract Forms/Tables are enabled only when the tenant schema requires them.

Textract async jobs write results to tenant-scoped S3 prefixes using Textract `OutputConfig` with the tenant KMS key. SNS/SQS callbacks are validated by expected topic ARN, signature, job ID, tenant/chunk match, and freshness window.

The normalized `ocr-structure.json` preserves page numbers, reading order, block IDs, bounding boxes, confidence scores, and character spans. Those spans become the citation anchors for downstream extraction.

Important decisions:

- Async Textract is chosen over synchronous OCR because chunk-level OCR is long-running.
- Structured OCR is schema-gated because Forms extraction can cost several times more per page than text extraction.
- OCR output must preserve provenance, not just raw text.

### 3.6 AI Extraction

Stage 4 uses Bedrock Llama 3.1 8B through Converse tool-use. The tool schema is generated from the tenant JSON Schema, and the model must call the `submit_extraction` tool exactly once. Free-form model text is ignored.

Every non-null extracted field must include `source_citations` pointing to page, block, and character span from Stage 3. If a value has no valid citation, it is rejected and marked `not_found`.

Recovery path:

1. Run default 8B extraction.
2. If JSON, schema, or citation validation fails, retry once with a compact repair prompt.
3. If critical fields still fail, route only the affected chunk to Llama 3.1 70B.
4. If that still fails, route to manual inspection.

Important decisions:

- Model output is treated as a proposal, not truth.
- Citations are the main hallucination control.
- 70B is used only for critical-field recovery because using it for every extraction would break the cost target.
- Bedrock Guardrails are mandatory for prompt injection and sensitive data controls.

### 3.7 Merge and Document-Level Synthesis

Stage 5 merges chunk outputs using schema-defined policies. Each field defines cardinality, criticality, merge strategy, and conflict behavior.

The default merge is deterministic: collect candidates, deduplicate, normalize, compare citations, and apply field-specific rules. Examples:

- Invoice totals must reconcile deterministically.
- Critical singleton conflicts route to manual inspection.
- Uncited critical fields are rejected.

Document-level synthesis is bounded. It is used only when required by schema or when Stage 4 flags a cross-chunk dependency. The synthesis prompt contains only relevant candidates and source spans, never the full document. It is capped at 10 questions per document.

Important decisions:

- Merging after extraction is more practical than the case study order because chunk-level field candidates must exist before the platform can merge them.
- Synthesis is a controlled exception, not the default merge strategy.

### 3.8 Post-processing and Redaction

Stage 6 is deterministic and does not call a model. It validates the merged result against the tenant schema, normalizes dates/currencies/identifiers, runs business validators, applies redaction policy, and produces three artifacts:

- `final-result.json`: full internal access-controlled result.
- `customer-result.json`: customer-facing redacted output.
- `validation-report.json`: operator/internal validation evidence.

Redaction happens after validation, never before, because validators need real values for invoice totals, dates, IDs, and reconciliation. If redaction fails, delivery is blocked.

Important decisions:

- No model calls in Stage 6 keeps final validation cheap, fast, and auditable.
- Bounded reprocess prevents infinite retry loops.

### 3.9 Storage, Retrieval, and Webhooks

Stage 7 writes the final customer artifact to S3, updates terminal state in DynamoDB, and triggers webhook delivery.

The result API returns a short-lived signed URL, not the inline JSON body. Webhooks are pointer-only and HMAC-signed. They include job status, result hash, and retrieval metadata, but no extracted values, raw text, PII, or presigned S3 URL.

Important decisions:

- Webhook failure does not roll back a completed job.
- Signed URL retrieval keeps API response size consistent.
- Pointer-only webhooks reduce accidental PII exposure.

## 4. Operational Strategy

The operational strategy is based on four feedback loops:

| Loop | Purpose |
|---|---|
| Reliability | Detect retries, DLQs, poison documents, provider outages, and stuck jobs. |
| Latency | Track P50/P95/P99, queue age, slowest chunk, and critical path. |
| Cost | Track OCR pages, model tokens, retries, storage, and per-tenant spend. |
| Quality | Track precision, recall, F1, citation validity, manual-review rate, and model drift. |

Key SLOs:

- Platform availability: 99.9%.
- End-to-end processing latency: P95 under 5 minutes after `accepted_at`.
- Stage budgets: preprocessing <=30s, OCR P95 <=180s, extraction P95 <=60s, merge P95 <=30s, post-processing <=10s, storage/notification <=10s.
- Job success rate tracked by document type and schema.
- Critical-field quality cannot materially regress during canaries.
- GDPR deletion workflow must be tracked and alerted.

Dashboards:

- Executive health: jobs, latency, success rate, cost, backlog, provider status.
- Pipeline operations: stage latency, queue age, DLQs, retries, stuck jobs, tenant throttling.
- Model operations: Bedrock calls, tokens, TTFT, validation failures, citation failures, guardrails.
- Cost: cost per job, per tenant, per document type, per stage, and exception rate.
- GPU: utilization, memory, vLLM queue, tokens/sec, TTFT, node provisioning, spot interruptions.

Alerting philosophy:

- Alert on customer impact and leading indicators, not every metric.
- Queue age is a primary latency risk signal.
- Cost spikes and token spikes are treated as production incidents.
- PII detected in logs is a security incident.

## 5. MLOps Practices

Infrastructure and release:

- All infrastructure is AWS CDK in Python.
- GitHub Actions runs format, lint, unit tests with >=80% coverage, security checks, CDK synth, IAM/policy checks, integration tests, and deployment.
- Prompts, schemas, merge policies, post-processing policies, and model routes are versioned.
- Rollback supports application code, infrastructure, prompts, schemas, merge policies, and model versions.

Model evaluation:

- Evaluate at document level, field level, and critical-field level.
- Track precision, recall, F1, citation validity, schema validation failure rate, manual-review rate, and cost per field/document.
- Slice quality by document type, tenant schema, page quality, OCR route, model version, and prompt version.

Canary policy:

- 10% traffic for 1 hour.
- Roll back on critical-field F1 regression, schema validation spike, citation failure spike, P95 latency regression, cost spike, provider throttling, manual-review spike, or PII leakage.

Replay:

- Replay the smallest unit that can fix the issue.
- Examples: OCR unit for missed callback, model invocation for invalid JSON, Stage 5 merge for merge bug, webhook event for delivery failure.
- Idempotency prevents duplicate OCR charges, duplicate model calls, duplicate billing events, and duplicate webhooks.

## 6. Security and Compliance Posture

Security principles:

- Tenant isolation everywhere.
- Customer content encrypted with tenant-scoped KMS keys.
- Model output is untrusted.
- Audit is immutable but PII-free.
- GDPR erasure and SOC 2 evidence are reconciled by not putting PII in immutable audit.

Controls:

- API edge: WAF, TLS 1.2+, API key, JWT, authorizer.
- Storage: S3 Block Public Access, SecureTransport deny, tenant/job prefixes, tenant KMS keys.
- Processing: least-privilege IAM, KMS encryption context, tenant/job prefix checks.
- Model: Bedrock Guardrails, prompt injection defense, schema-constrained output, citation validation.
- Logs: emit-time PII scrubbing and CloudWatch subscription leakage detection.
- Audit: EventBridge to Firehose to S3 Object Lock compliance mode.
- Delivery: authenticated signed URL retrieval and HMAC-signed pointer webhooks.

GDPR:

- Job artifacts are mapped by tenant/job for access and deletion.
- Deletion removes input, intermediate, output, retrieval index, and content-bearing delivery records.
- Crypto-shredding through tenant KMS key deletion is possible where contract and legal hold allow.
- PII-free immutable audit remains because it contains operational facts, hashes, and status codes, not customer content.

SOC 2:

- Change management evidence from CI/CD, tests, synth, canary, rollback records.
- Access evidence from IAM, API authorizer logs, operator access audit.
- Availability evidence from SLO dashboards, runbooks, DLQs, replay records.
- Processing integrity from artifact hashes, versioned schemas/prompts/models, and citation validation.

## 7. Risk Mitigation Summary

| Risk | Mitigation |
|---|---|
| Lambda limits | Use Lambda only for smaller preprocessing; use Fargate for large PDFs. |
| LLM context overflow | Token-aware chunking with 25K target and 30K hard cap. |
| 50,000-document burst | Admission control, SQS buffering, Step Functions concurrency, tenant token buckets. |
| Textract/Bedrock throttling | Per-stage concurrency caps, queue age alerts, circuit breakers, backpressure. |
| Cost overrun | OCR bypass, schema-gated Forms/Tables, token budgets, 70B only for recovery, stop-loss thresholds. |
| Hallucination | Tool-use JSON, schema validation, required citations, deterministic validators, manual inspection. |
| Poison documents | DLQs, poison queues, reason codes, bounded retries, replay tooling. |
| PII leakage | No raw text in logs, emit-time scrubbing, leakage detector, pointer-only webhooks. |
| Provider outage | Stage circuit breakers, queued accepted work, admission backpressure, evaluated fallback routes. |
| Manual review backlog | Separate critical/manual/degraded queues, backlog SLOs, priority rules. |
| Self-hosted GPU underutilization | Benchmark first, keep Bedrock fallback, track cost per 1M tokens, use KEDA/Karpenter. |

## 8. Managed vs. Self-Hosted Inference

The architecture is managed-first. Bedrock is the v1 default because it lowers operational complexity, avoids GPU capacity management, gives faster launch, and integrates better with AWS security and service controls.

The self-hosted option becomes attractive only when sustained volume is high enough and GPU utilization is proven. The proposed path is EKS with vLLM, Karpenter for GPU nodes, KEDA for model-serving scale, Prometheus/Grafana for GPU and inference metrics, and Bedrock as fallback.

Break-even logic:

```text
cost_per_1M = H_g / (E * 3600 / 1,000,000)

Where:
H_g = hourly GPU node cost
E = effective sustained tokens/sec
P_b = Bedrock price per 1M tokens

Self-hosted wins when:
H_g / (E * 3600 / 1,000,000) < P_b
```

Using the planning numbers:

- Reserved `g5.xlarge`: about $0.634/hr.
- Bedrock 8B price: about $0.22 per 1M tokens.
- Required effective throughput: about 801 tokens/sec.
- At 80% utilization, raw throughput needs to be around 1,001 tokens/sec.
- Recommended transition point: around 10,000 docs/day, after production-like benchmarks.

Interview nuance:

- Do not claim EKS is always cheaper. It is cheaper only if utilization, throughput, and operational overhead work out.
- Long-context prompts may lower vLLM throughput, so benchmarks must use production-like 18K-25K token prompts.
- 70B recovery can remain managed even if 8B moves to self-hosted.

## 9. Major Tradeoffs and Alternatives

### 9.1 Direct S3 Upload vs API Proxy Upload

Chosen: direct presigned S3 upload with Transfer Acceleration.

Why:

- Supports 300 MB files without API Gateway/Lambda payload issues.
- Removes compute from the hot upload path.
- Allows S3-native encryption, lifecycle, and eventing.

Alternative:

- API Gateway + Lambda proxy upload.

Why not:

- Payload size and timeout constraints.
- More expensive and less reliable for large PDFs.

### 9.2 Step Functions vs Custom Queue Orchestrator

Chosen: Step Functions Standard with Distributed Map plus SQS/EventBridge.

Why:

- Durable state machine, visual execution trace, retries, and controlled fan-out.
- Good fit for a multi-stage workflow with replayable artifacts.

Alternative:

- Pure SQS workers with custom orchestration state.

Tradeoff:

- SQS-only can be cheaper and more flexible at extreme scale, but requires custom state management, replay semantics, visibility, and failure handling.

### 9.3 Lambda vs Fargate for Pre-processing

Chosen: Lambda for smaller files, Fargate for large/heavy PDFs.

Why:

- Lambda is fast and cheap for small deterministic work.
- Fargate handles 1,000-page/300 MB edge cases with better CPU, memory, and runtime control.

Alternative:

- Lambda-only.

Why not:

- Timeout, memory, and `/tmp` limitations make it risky at the upper bound.

Alternative:

- Fargate-only.

Why not:

- Slower startup and unnecessary cost for small PDFs.

### 9.4 Async Textract vs Synchronous OCR

Chosen: async Textract with SNS/SQS callbacks.

Why:

- Long-running OCR should not block Lambda or API requests.
- Callbacks are durable and replayable.

Alternative:

- Synchronous Textract.

Why not:

- Only suitable for small/single-page work, not chunked large PDFs.

### 9.5 Textract Forms/Tables Everywhere vs Schema-Gated Use

Chosen: schema-gated structured OCR.

Why:

- Structured OCR is more expensive.
- Many pages only need text plus LLM extraction.

Alternative:

- Run Forms/Tables for every page.

Why not:

- It can consume the full cost budget before LLM extraction starts.

### 9.6 Bedrock Managed Models vs Self-Hosted EKS

Chosen: Bedrock first, EKS/vLLM later.

Why:

- Faster launch.
- Lower operational burden.
- Better for uncertain volume.
- Native AWS managed security and availability.

Alternative:

- Self-hosted LLMs from day one.

Why not:

- GPU autoscaling, patching, utilization, model serving, availability, and benchmarking add significant operational complexity.

### 9.7 8B Default Model vs 70B Default Model

Chosen: 8B default with 70B recovery.

Why:

- 8B is cost-effective for schema-constrained chunk extraction.
- 70B is more expensive and reserved for critical failures.

Alternative:

- 70B for all documents.

Why not:

- Better quality may not justify cost across every chunk and can break the $0.10 target.

### 9.8 LLM Merge vs Deterministic Merge

Chosen: deterministic merge first, bounded synthesis only when needed.

Why:

- Deterministic merge is cheaper, faster, testable, and auditable.
- Most field conflicts can be handled with schema rules and validators.

Alternative:

- Put all chunk outputs into another LLM prompt and ask for final answer.

Why not:

- Higher cost, harder provenance, harder validation, and more hallucination risk.

### 9.9 Inline Results vs Signed URL Retrieval

Chosen: signed URL retrieval.

Why:

- Works for small and large results.
- Keeps API latency stable.
- Avoids large payload limits.

Alternative:

- Return result inline from API.

Why not:

- Poor fit for large outputs and harder to control response payload size.

### 9.10 Webhook With Full Result vs Pointer-Only Webhook

Chosen: pointer-only webhook.

Why:

- Reduces PII leakage risk.
- Keeps authorization in the retrieval API.
- Customer endpoint outage does not block completed jobs.

Alternative:

- Send extracted JSON in webhook.

Why not:

- Pushes sensitive data outside controlled access paths.

### 9.11 Per-Tenant KMS Keys vs Single Platform Key

Chosen: per-tenant KMS keys.

Why:

- Stronger cryptographic tenant isolation.
- Enables crypto-shredding for deletion workflows.

Alternative:

- Single platform CMK.

Why not:

- Simpler but weaker isolation and less flexible erasure.

### 9.12 PII-Free Immutable Audit vs Full Audit Payloads

Chosen: immutable audit with operational facts only.

Why:

- SOC 2 needs tamper-resistant evidence.
- GDPR erasure requires customer content to be deletable.

Alternative:

- Store full job details or extracted values in immutable audit.

Why not:

- Creates conflict with erasure obligations.

## 10. Likely Interview Questions and Strong Answers

### Q1. Walk me through the architecture end to end.

Answer:

The architecture starts with API Gateway and WAF. A tenant creates a job and receives a presigned S3 upload target. The PDF goes directly to S3 using Transfer Acceleration. An S3 event triggers validation: checksum, type, size, page count, tenant key, upload TTL, and malware scan. Once accepted, Step Functions starts the pipeline.

Preprocessing profiles the PDF and emits page and chunk manifests. Digital pages bypass OCR; scanned and hybrid pages go through async Textract. The normalized OCR output preserves spans and layout. Bedrock then extracts schema-valid chunk JSON using tool-use, and every field requires citations back to OCR spans. Chunk outputs are merged deterministically, with bounded synthesis only for cross-chunk questions. Post-processing validates, normalizes, redacts, and produces final artifacts. Stage 7 stores the customer output in S3, updates DynamoDB, and sends pointer-only webhooks.

The important cross-cutting ideas are durable artifacts, idempotency at every async boundary, tenant-scoped encryption, queue-based backpressure, and observability on latency, cost, quality, and security.

### Q2. Why does the design start the 5-minute SLA clock at `accepted_at`?

Answer:

Because upload time for a 300 MB file is dependent on the customer's network, geography, and retry behavior. The platform can optimize upload with S3 Transfer Acceleration, but it cannot control the client's network. Starting the SLA after upload validation and malware scan gives a fair processing SLA for work the platform actually controls.

I would still measure upload latency separately as a product metric, but I would not include it in the processing SLA.

### Q3. How do you meet the 5-minute P95 target for 1,000-page documents?

Answer:

The design avoids serial processing of the full document. It profiles the document, chunks it, and fans out chunk-level OCR/extraction work with bounded concurrency. Digital pages bypass OCR, which is the biggest latency and cost win. Async Textract avoids blocking compute while OCR runs. Stage 4 extraction happens per chunk with token-capped prompts. Merge and post-processing are deterministic and intentionally fast.

The design also treats queue age as a leading indicator. If OCR or model queues threaten the stage budget, admission control slows new accepted work before the end-to-end SLA is already breached.

I would be careful with the exact claim: a pathological 1,000-page low-quality scanned PDF may not fit the representative P95. The system should surface that as an exception, not hide it.

### Q4. Why use both Step Functions and SQS/EventBridge?

Answer:

They solve different problems. Step Functions owns the durable workflow and makes stage transitions, retries, and execution visibility clear. Distributed Map gives controlled fan-out over chunks. SQS and EventBridge provide buffering, decoupled handoffs, DLQs, and independent replay for stages such as callbacks and webhooks.

Using only Step Functions can make some asynchronous external handoffs awkward. Using only SQS would require building a custom workflow engine. The combination gives orchestration plus backpressure.

### Q5. Why not send the entire 1,000-page document to an LLM with a 128K context window?

Answer:

A 1,000-page document can exceed 128K tokens, especially legal contracts or dense reports. Even if it fits sometimes, full-document prompting creates unpredictable cost, latency, and failure modes. It also makes citation validation and retries coarse-grained.

Chunk-level extraction gives bounded prompts, parallelism, localized retries, and better cost control. For true cross-document questions, the system uses bounded synthesis over selected candidates and source spans, not the full raw document.

### Q6. How do you prevent hallucinations?

Answer:

The model is not trusted as an authority. It must produce schema-constrained output through tool-use, and every non-null field must include source citations pointing to page/block/span in the OCR structure. A resolver checks that citations exist and match the current artifact. Missing or invalid citations reject the field.

Critical fields also go through deterministic validators, such as invoice total reconciliation or date ordering. Conflicts route to stronger-model recovery or manual inspection. This makes hallucination a controlled failure mode instead of silently shipping fabricated data.

### Q7. Why use Llama 3.1 8B as the default and 70B only for recovery?

Answer:

The extraction task is schema-constrained and citation-grounded, so an 8B model can be cost-effective for most chunks. The 70B model is more capable but materially more expensive. If we used 70B for all chunks, the LLM stage could consume too much of the $0.10 per 100-page budget.

The recovery route uses 70B only when critical fields fail validation or citation checks. That preserves quality for high-value failures without making the entire pipeline expensive.

### Q8. Why is the merge deterministic instead of LLM-based?

Answer:

Merging is a place where we want auditability and repeatability. Many merge cases are rule-based: deduplicate repeated invoice numbers, reconcile totals, choose highest-confidence cited candidate, or route singleton conflicts to review.

An LLM-first merge would be harder to test, more expensive, and easier to hallucinate. The design still supports synthesis, but only for bounded cross-chunk questions where deterministic rules are not enough.

### Q9. What is your idempotency strategy?

Answer:

Every async boundary has a scoped idempotency key. Job creation uses tenant, client request ID, and request hash. OCR includes tenant, job, chunk, OCR unit, policy version, and input artifact hash. AI extraction includes tenant, job, chunk, schema version, prompt version, model route, and input artifact hash. Merge includes ordered chunk artifact hashes and merge policy version. Webhooks include event ID and final result hash.

If a retry arrives with the same key and same input, it is a no-op or returns the existing result. If the same key arrives with different content, it is rejected as a conflict. This prevents duplicate processing, duplicate charges, and duplicate webhook delivery.

### Q10. How do you handle poison-pill documents?

Answer:

Retries are bounded with exponential backoff and jitter. If a document or chunk repeatedly fails with deterministic errors, it moves to a poison queue with tenant, job, stage, attempt count, reason code, and artifact pointers. It stops blocking healthy traffic.

Operators can inspect the failure, replay the smallest safe unit, change policy if needed, or route to manual review. The key is to isolate repeated failures and avoid infinite retry loops or cost explosions.

### Q11. How do you control cost below $0.10 per 100 pages?

Answer:

Cost control starts before paid work begins. The system classifies pages and bypasses OCR for digital text. It uses text OCR by default and gates expensive Forms/Tables extraction by schema. It caps token budgets per chunk, uses an 8B model by default, uses 70B only for critical recovery, and limits synthesis questions.

It also maintains a cost ledger per job, tenant, document type, and stage. Soft and hard stop-loss thresholds stop optional work or require review when a job becomes too expensive. I would state the target is for a representative mix, not for every possible fully-scanned or form-heavy document.

### Q12. What happens during a 50,000-document burst?

Answer:

The API does not convert the burst into 50,000 simultaneous OCR or model calls. Admission control gates accepted work. SQS and EventBridge buffer handoffs. Step Functions Distributed Map has max concurrency. OCR and model stages have global, per-tenant, and per-job limits.

If queue age threatens the SLA, the system returns `429 Retry-After` for new work and preserves fairness for already accepted jobs. Tenant token buckets prevent one tenant from starving others.

### Q13. What are the main service quota risks?

Answer:

The main quotas are Textract async job limits, Bedrock request/token limits, Step Functions map concurrency and state transitions, Lambda concurrency, DynamoDB write capacity or hot partitions, S3 request rates, and webhook throughput.

The mitigation is launch readiness quota requests, bounded concurrency, sharded DynamoDB indexes, high-cardinality S3 keys, queue-age alarms, and circuit breakers. I would load-test the default caps before production because quota behavior varies by account and region.

### Q14. Why per-tenant KMS keys?

Answer:

Per-tenant keys provide cryptographic isolation, not only IAM isolation. If there is an application bug or policy misconfiguration, the tenant key boundary still matters. It also supports crypto-shredding in deletion workflows when contract and legal hold permit.

The tradeoff is more key management complexity and KMS policy management, but for sensitive multi-tenant document processing, the isolation benefits justify it.

### Q15. How do you reconcile SOC 2 immutable audit with GDPR deletion?

Answer:

The audit trail is immutable but PII-free. It stores operational facts such as actor, tenant/job ID, event type, status, timestamps, hashes, and reason codes. It does not store raw document text, filenames, extracted values, presigned URLs, or PII.

GDPR deletion removes customer content from input, intermediate, output, indexes, and content-bearing delivery records. The PII-free audit record remains because it is security evidence and not customer content.

### Q16. How do you protect PII in logs?

Answer:

The primary control is to avoid logging raw document text, extracted values, filenames, or presigned URLs. Structured logging scrubs emails, SSN-like values, phone numbers, tokens, and URLs at emit time. A CloudWatch Logs subscription then runs leakage detection with pattern rules and Comprehend-backed inspection.

If leakage is detected, it is treated as a security incident. The offending release is stopped or rolled back, logs are contained according to policy, and secrets are rotated if any were exposed.

### Q17. Why are webhooks pointer-only?

Answer:

Webhook endpoints are outside the platform's direct access-control boundary. Sending extracted values in webhooks would push sensitive customer data to an endpoint that may be misconfigured, unavailable, or logged insecurely.

Pointer-only webhooks notify the customer that a result is ready. The customer retrieves it through the authenticated API, which can enforce authorization, sign URLs, audit access, and apply expiration.

### Q18. How do you operate model quality over time?

Answer:

Prompts, schemas, model routes, and merge policies are versioned. Evaluation tracks precision, recall, F1, citation validity, schema failures, manual-review rate, and critical-field accuracy by document type, schema, OCR route, page quality, and model version.

Any extraction-affecting change runs evaluation before deployment and canary after deployment. Canary rollback triggers include critical-field F1 regression, citation failures, schema validation spike, cost spike, latency regression, or manual-review spike.

### Q19. What does canary release mean in this system?

Answer:

Canary applies not only to application code but also prompts, schemas, merge policies, post-processing policies, and model routes. The rollout sends 10% of traffic through the new version for a 1-hour soak.

During the soak we compare latency, error rate, token usage, cost, citation validity, schema validation, critical-field F1, and manual-review rate. Rollback needs to be version-aware because prompt/schema/model changes can affect stored artifacts and replay behavior.

### Q20. Why not build self-hosted LLM inference from day one?

Answer:

Self-hosted inference can be cheaper at high sustained utilization, but it adds GPU provisioning, autoscaling, model serving, patching, security, capacity planning, and observability work. At early or uncertain volume, Bedrock is simpler and often cheaper once operational overhead is included.

The design includes a future EKS/vLLM path with Karpenter and KEDA, but it should be adopted only after production-like benchmarks show enough effective tokens/sec and utilization to beat Bedrock.

### Q21. Explain the self-hosted break-even calculation.

Answer:

The formula compares hourly GPU cost against effective token throughput:

```text
cost_per_1M = H_g / (E * 3600 / 1,000,000)
```

For a reserved `g5.xlarge` at about $0.634/hr and Bedrock at about $0.22 per 1M tokens, the GPU needs about 801 effective tokens/sec to match Bedrock token price. If we target 80% useful utilization, raw serving throughput needs to be around 1,001 tokens/sec.

That is before adding EKS control plane, observability, engineering, and operational risk. So the recommendation is to switch around 10,000 docs/day only after benchmarks confirm the throughput.

### Q22. How would Karpenter and KEDA work together?

Answer:

KEDA scales the vLLM deployment based on application signals such as SQS extraction queue depth and vLLM `num_requests_waiting`. When KEDA needs more pods than current nodes can host, Karpenter provisions GPU nodes. Karpenter handles instance selection, spot/on-demand/reserved capacity, and consolidation.

KEDA scales pods; Karpenter scales nodes. Prometheus and Grafana track GPU utilization, memory, TTFT, queue depth, tokens/sec, node provisioning time, and cost per 1M tokens.

### Q23. What would you monitor first in production?

Answer:

I would monitor five things from day one:

1. End-to-end P95 latency and per-stage queue age.
2. Textract and Bedrock throttles/errors.
3. Cost per job and per 100 pages by tenant/document type.
4. Citation validation failure rate and manual-review rate.
5. DLQ/poison queue growth.

Queue age is especially important because it tells us before customers experience SLA misses.

### Q24. What is the most important architectural decision in your design?

Answer:

The most important decision is to make every stage artifact-driven and idempotent. The pipeline is asynchronous and large documents can fail in many places. Durable artifacts and scoped idempotency make replay safe, prevent duplicate paid work, and allow operators to repair a small unit instead of rerunning an entire document.

The second major decision is deterministic-first processing. It keeps the LLM limited to extraction and bounded synthesis, rather than letting it control orchestration, security, billing, or final validation.

### Q25. Where is the design weakest?

Answer:

The biggest uncertainty is the cost and latency model for pathological workloads: fully scanned, low-quality, 1,000-page documents with complex forms. The architecture can process them, but the $0.10 and 5-minute targets are based on a representative mix. I would make that explicit in the interview.

The second uncertainty is service quota behavior under burst load. The design has concurrency controls and backpressure, but the exact caps need load testing and quota approvals before launch.

### Q26. Why did you invert the case study order of merging before AI extraction?

Answer:

The case study lists merging before AI extraction, but for chunked LLM extraction the practical order is chunk extraction first, then merge. The system needs field candidates, citations, confidence, and per-chunk extraction results before it can merge meaningfully.

What is still preserved is the intent: produce a single ordered document structure. The architecture does that after chunk extraction, with deterministic merge and bounded synthesis.

### Q27. How would you handle manual review without breaking automation?

Answer:

Manual review is a controlled terminal or near-terminal path, not an unbounded fallback. Critical conflicts, uncited critical values, repeated validation failures, and unresolved low-confidence fields route to manual inspection with reason codes and artifact pointers.

Manual review backlog becomes an operational SLO. Reviewed cases feed back into eval datasets, schema rules, and prompt improvements. Non-critical missing optional fields can go to degraded review rather than blocking the entire job.

### Q28. How do you make sure customers are charged fairly?

Answer:

The cost ledger tracks work by job, tenant, stage, and policy. Platform-caused recovery, such as retry after transient model error or stronger-model recovery for quality, can be marked non-billable. Customer-requested replay after input or schema changes can be billable.

Idempotency keys prevent duplicate charges when retries or duplicate events occur. The ledger should separate processing cost from retention/storage cost.

### Q29. How do you handle missed S3 events or Textract callbacks?

Answer:

For S3 events, a reconciliation job can scan pending uploads and object metadata to find uploads that did not trigger validation. For Textract callbacks, the workflow can use a timeout and poll fallback. If Textract completed, normalization can replay from the completed output instead of starting a duplicate paid OCR job.

This is why storing state and artifact pointers matters. The system can recover from missed notifications without assuming the provider callback is perfect.

### Q30. What would you build first as an MVP?

Answer:

I would build the managed path first:

1. API, presigned upload, validation, job state.
2. Preprocessing and chunk manifest.
3. Digital text extraction plus async Textract for scanned pages.
4. Bedrock 8B schema-constrained chunk extraction with citations.
5. Deterministic merge, validation, redaction, and signed URL retrieval.
6. Core observability: latency, queue age, cost, tokens, citation failures, DLQs.

I would defer self-hosted EKS inference, advanced learned page classification, PrivateLink, and complex tenant-specific manual-review workflows until the core pipeline and data shape are validated.

## 11. How to Present This in the Interview

Start with the constraints:

- Large PDFs, high throughput, burst handling, low latency, low cost, and compliance.

Then state the design principle:

- Deterministic first; OCR and LLM only where needed; everything durable, idempotent, and replayable.

Then walk stage by stage:

1. API creates job and presigned upload.
2. S3 validation and malware gate accepts job.
3. Preprocessing creates page/chunk manifests.
4. OCR is selective and async.
5. LLM extraction is schema-constrained and citation-grounded.
6. Merge is deterministic with bounded synthesis.
7. Post-processing validates and redacts.
8. Storage and notification use signed URLs and pointer-only webhooks.

Then emphasize cross-cutting controls:

- Backpressure and queue age for latency.
- Cost ledger and stop-loss for budget.
- KMS and PII-safe logs for security.
- X-Ray, CloudWatch, dashboards, and DLQs for operations.
- CI/CD, evals, canaries, and rollback for MLOps.

Close with tradeoffs:

- Managed Bedrock first for speed and lower ops burden.
- EKS/vLLM only after benchmarked break-even.
- Structured OCR only when schema requires it.
- Deterministic merge instead of LLM-first merge.
- Per-tenant KMS for stronger isolation despite more complexity.

## 12. Concise Closing Statement

The architecture is intentionally not just an AI pipeline. It is a production document-processing platform with controlled ingestion, durable state, bounded parallelism, selective OCR, schema-constrained model extraction, deterministic quality gates, secure multi-tenant storage, replayable operations, and measurable MLOps. The key tradeoff is using managed AWS services first to reduce delivery and operational risk, while keeping a quantified path to self-hosted GPU inference once volume justifies it.
