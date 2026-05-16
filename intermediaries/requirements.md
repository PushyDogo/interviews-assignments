# Requirements - High-Throughput Hybrid Document AI Platform

## 1. Mission

Design an AWS-based, serverless, API-driven Document AI platform for asynchronous processing of large PDFs, including invoices, legal contracts, forms, and corporate reports.

The platform must process documents up to 1,000 pages and 300 MB, sustain 10,000+ documents per day, absorb bursts up to 50,000 documents, complete large jobs within a P95 latency of 5 minutes, maintain 99.9% availability, and keep processing cost below $0.10 per 100-page document.

## 2. Functional Requirements

### 2.1 Ingestion

- Accept PDF uploads up to 300 MB.
- Use S3 Transfer Acceleration for all upload endpoints to mitigate ingestion bottlenecks for large files.
- Generate a globally unique job ID for every submitted document.
- Validate tenant identity, schema selection, file type, file size, checksum, and page-count limits.
- Persist the original document in S3.
- Track job state in a durable database from submission through terminal status.
- Start the processing workflow only after upload completion is confirmed.

### 2.2 Pre-processing

- Split large PDFs into 20-50 page chunks, with chunking also constrained by token budget and document structure.
- Detect page orientation, skew, and rotation.
- Enhance image quality for scanned or low-quality pages before OCR.
- Avoid expensive rasterization or image enhancement for native digital PDFs when text can be extracted directly.
- Classify pages as digital-text, scanned, or hybrid to route them through the cheapest reliable path.

### 2.3 OCR and Structure Extraction

- Execute OCR asynchronously for scanned or hybrid pages.
- Support text, forms, and tables extraction where required by the document type or tenant tier.
- Prefer raw OCR plus downstream model extraction for the cost-controlled default path.
- Treat advanced structured OCR APIs as selective or premium paths because their page-level costs can exceed the target budget.
- Preserve page numbers, reading order, layout hints, table boundaries, and source spans for downstream validation.

### 2.4 AI Extraction

- Apply SLMs or LLMs for document classification, entity extraction, and schema-constrained JSON generation.
- Support invoices, contracts, forms, and corporate reports with document-type-specific extraction policies.
- Enforce schema-valid outputs.
- Include source citations or spans for extracted fields.
- Support chunk-level extraction plus a document-level synthesis pass for cross-page reasoning in contracts and reports.
- Keep prompts token-capped to stay within context-window, latency, and cost constraints.

### 2.5 Merging

- Merge chunk outputs into one ordered document representation.
- Deduplicate repeated fields across chunks.
- Reconcile cross-page fields such as totals, clauses, parties, dates, and signatures.
- Preserve field-level provenance from final output back to source pages and spans.
- Track which chunks completed and which failed; surface partial results with a clear status distinguishing full success, partial success, and failure.

### 2.6 Post-processing

- Validate extracted data against tenant schemas.
- Normalize dates, currencies, addresses, names, identifiers, totals, and tax fields.
- Apply deterministic validators for high-value fields.
- Assign field-level confidence scores.
- Route low-confidence or unverifiable critical fields to manual review or stronger-model reprocessing.
- Produce structured JSON output suitable for API retrieval and downstream customer workflows.

### 2.7 Storage and Retrieval

- Store original inputs, intermediate artifacts, final JSON outputs, and audit events separately.
- Persist final results to S3.
- Update durable job status on every major state transition.
- Provide an API to retrieve job status and final result.
- Support webhook notification on terminal job states without pushing large result payloads directly to customer endpoints.

## 3. Non-Functional Requirements

### 3.1 Scale and Throughput

- Sustain at least 10,000 documents per day.
- Support burst intake of 50,000 documents without overloading OCR, LLM, database, or webhook delivery services.
- Use asynchronous queues and event routing to decouple ingestion from downstream processing.
- Use S3 Transfer Acceleration for upload throughput at scale.
- Bound per-stage concurrency to avoid exhausting AWS service quotas.
- Support elastic fan-out for chunk-level processing.

### 3.2 Latency

- Meet P95 end-to-end processing latency below 5 minutes for large documents after upload acceptance.
- Avoid synchronous waits in API request paths.
- Process chunks in parallel where service quotas and cost controls allow.
- Maintain stage-level latency budgets for ingestion, chunking, OCR, extraction, merge, validation, and notification.

### 3.3 Availability and Resilience

- Provide 99.9% platform availability.
- Use durable state machines, queues, and storage to survive transient failures.
- Implement retries with exponential backoff and jitter.
- Use DLQs for failed asynchronous work.
- Support automated replay for recoverable failures.
- Provide manual inspection workflows for poison-pill documents that repeatedly fail.
- Ensure all processing steps are idempotent.

### 3.4 Cost

- Keep processing cost below $0.10 per 100-page document.
- Track cost per job, per tenant, per document type, and per pipeline stage.
- Minimize OCR cost by bypassing OCR for digital-text pages.
- Minimize LLM cost through token compression, page routing, prompt caching where available, smaller models where acceptable, and schema-specific prompts.
- Define a token budget per 100-page document before claiming the cost target is met.
- Prevent cost overruns with tenant quotas, per-job limits, model-routing policies, and spend alerts.

## 4. Technical Challenges to Address

### 4.1 Large Documents and Lambda Limits

- Documents can reach 1,000 pages and 300 MB, which creates timeout, memory, and temporary-storage risks.
- Chunking must be robust, resumable, and token-aware.

### 4.2 LLM Context Limits

- The case study calls out 128K-token context constraints.
- A 1,000-page document cannot be sent to an LLM as one prompt.
- The architecture must use chunk-level extraction, compressed summaries, citations, and a bounded document-level synthesis step.

### 4.3 Burst Protection

- The platform must absorb 5x peak bursts without throttling downstream AI services.
- SQS, EventBridge, and Step Functions concurrency controls should buffer and shape traffic.
- Backpressure must be visible and tied to admission control or tenant-level fairness.

### 4.4 Service Quotas

- OCR async job limits, Step Functions map concurrency, Lambda concurrency, Bedrock token/request limits, DynamoDB throughput, S3 request rates, and webhook delivery throughput must be modeled.
- Quota increase requests should be part of launch readiness.
- The system must degrade predictably when quotas are reached.

### 4.5 Idempotency and Replay

- Idempotency must cover submission, upload finalization, chunk processing, OCR callbacks, LLM extraction, merge, billing, audit events, and webhook delivery.
- Idempotency keys must be scoped by tenant.
- Reuse of the same idempotency key with different request content must be rejected.
- Replay should be possible from the document, chunk, or stage level without duplicating billing or audit events.

### 4.6 Poison-Pill Handling

- Documents or chunks that repeatedly fail must be isolated from normal queues.
- Failed items should carry failure reason, retry count, stage, tenant, and artifact pointers.
- Operators need a manual inspection and replay path.
- Poison-pill handling must avoid infinite retries and uncontrolled cost growth.

### 4.7 Model Hallucination

- LLM outputs are probabilistic and can produce plausible but incorrect field values, fabricated citations, or schema-valid JSON with wrong data.
- Mitigate through confidence scoring on extracted fields, source-span grounding requirements, and schema validation that rejects outputs lacking required citations.
- Route low-confidence or ungrounded critical fields to manual review or reprocessing with a stronger model rather than passing them silently to output.
- Track hallucination-related manual-review rates as a quality signal and regression indicator across model versions and document types.
- Include hallucination risk explicitly in the Risk Mitigation Plan deliverable alongside service quotas and cost overruns.

### 4.8 Managed vs. Self-Hosted Inference

- The platform must support a managed-first path using AWS managed AI services.
- The design must also cover a self-hosted LLM option on EKS using models such as Llama or Mistral served through vLLM or TGI.
- GPU orchestration must cover node provisioning, workload scheduling, scale-to-zero or warm-pool policy, and tenant isolation.
- Karpenter can manage GPU node lifecycle; KEDA can scale inference deployments from queue depth, token backlog, or custom metrics.
- A break-even model is required to decide when EKS GPU inference is cheaper than Bedrock token pricing.

## 5. Cost and Reliability Standards

### 5.1 Cost Controls

- Maintain a cost ledger for every job.
- Attribute cost across OCR, LLM input tokens, LLM output tokens, embeddings, orchestration, storage, network transfer, and observability.
- Enforce tenant-level quotas before expensive work starts.
- Use per-page routing to avoid unnecessary OCR and LLM calls.

### 5.2 Reliability Controls

- Store job state durably and update it at every transition.
- Make every transition safe to retry.
- Use DLQs and replay tooling for OCR, extraction, merge, and webhook delivery.
- Define terminal states including completed, failed, partial failed, manual review required, and deletion requested.

## 6. MLOps Operational Standards

### 6.1 Infrastructure as Code

- Use AWS CDK with Python for all infrastructure stacks.
- Keep application, data, observability, security, and inference infrastructure reproducible through code.
- Require CDK synthesis validation in CI.

### 6.2 CI/CD

- Use GitHub Actions for build, test, lint, security checks, and deployment workflows.
- Require at least 80% unit test coverage.
- Validate CDK synth output before deployment.
- Add integration tests for core job lifecycle paths.
- Block deployment when critical tests, synth, or policy checks fail.

### 6.3 Model Evaluation

- Track precision, recall, and F1 scores for extraction quality.
- Evaluate at document level, field level, and critical-field level.
- Track accuracy by document type, tenant schema, page quality, and model version.

### 6.4 Deployment and Release

- Support 10% traffic canary rollouts over a 1-hour soak period.
- Track latency, error rate, extraction quality, token usage, cost per job, and manual-review rate during canaries.
- Support rollback for application code, prompts, schemas, and model versions.
- Version prompts, schemas, and model configurations.

### 6.5 Operations

- Define SLOs for availability, P95 latency, job success rate, extraction quality, and webhook delivery.
- Provide operational dashboards for managed inference and EKS-hosted inference.

## 7. Security and Compliance Requirements

### 7.1 Data Protection

- Encrypt data at rest using customer-managed KMS keys.
- Encrypt data in transit for all service-to-service and customer-facing traffic.
- Apply a default 365-day retention policy for customer data unless tenant policy requires otherwise.
- Separate input, intermediate, output, and audit storage.
- Apply least-privilege IAM policies to every service role.

### 7.2 PII Protection

- Automatically detect and mask sensitive data such as SSNs and email addresses in logs.
- Prevent raw document text and unmasked PII from being written to application logs.
- Support redacted outputs where required by tenant policy.
- Keep PII detection cost controlled by filtering log payloads and avoiding blanket scans of every raw artifact unless required.

### 7.3 GDPR Alignment

- Support the right to access and erasure where applicable.
- Support deletion workflows for completed jobs, including removal of stored inputs, intermediate artifacts, outputs, and audit-exempt data.
- Ensure final outputs and retained artifacts can be mapped to the tenant and job for retrieval or deletion.
- Provide explainability through source spans and audit trails for automated extraction decisions.

### 7.4 SOC 2 Alignment

- Maintain immutable audit records for security-relevant events and job state transitions.
- Enforce change management through CI/CD, review, test evidence, and canary deployments.
- Capture access logs for customer data retrieval and operator access.
- Maintain incident response and operational runbooks.

### 7.5 Multi-Tenant Isolation

- Scope all data, keys, quotas, schemas, prompts, and jobs by tenant.
- Prevent cross-tenant access through IAM, database partitioning, storage prefixes, and application authorization checks.
- Attribute cost, quality, and operational metrics per tenant.

## 8. Observability Requirements

### 8.1 Distributed Tracing

- Use AWS X-Ray for end-to-end tracing across API, orchestration, queue, OCR, extraction, merge, storage, and webhook stages.
- Propagate correlation IDs, tenant IDs, job IDs, chunk IDs, and trace IDs across asynchronous boundaries.

### 8.2 Metrics

- Track P50/P95/P99 end-to-end latency.
- Track per-stage latency and queue age.
- Track throughput, backlog, success rate, failure rate, retry count, and DLQ depth.
- Track token usage, time to first token, model latency, model throttles, and model error rate.
- Track cost per job, cost per 100 pages, and cost per tenant.
- Track OCR page count, OCR latency, OCR error rate, and OCR service throttling.
- Track manual-review rate and field-level confidence distribution.

### 8.3 Logs

- Use structured logs with job ID, tenant ID, stage, chunk ID, correlation ID, and error reason.
- Mask or omit PII and raw document content.
- Keep logs operationally useful without turning them into a secondary document store.

### 8.4 Alerts

- Alert on SLO burn rate, queue backlog, DLQ growth, provider throttling, cost spikes, token spikes, OCR failures, webhook failures, and canary degradation.
- Alert on security events such as unauthorized access attempts, KMS failures, unusual tenant access patterns, and policy violations.

### 8.5 GPU Observability for Self-Hosted Inference

- Use Prometheus and Grafana for EKS and GPU metrics.
- Track GPU utilization, GPU memory, request queue depth, batch size, tokens per second, TTFT, model latency, error rate, pod restarts, node provisioning time, and cost per GPU hour.
- Compare GPU utilization and cost per token against Bedrock managed-model metrics.

## 9. Open Questions and Discussion Points

- Is the $0.10 target per submitted document or per 100-page document? The case study states both forms; cost modeling should normalize to cost per 100 pages.
- Does the 5-minute latency target apply to all 1,000-page documents or to large-document P95 within a realistic document mix?
- What percentage of documents are scanned, digital-text, or hybrid?
- Are advanced Textract Forms/Tables required for all customers, or can the default path use raw OCR plus LLM extraction?
- Which data residency regions are required?
- What are acceptable manual-review thresholds for low-confidence critical fields?
- What are the required retention periods for input, intermediate, output, and audit data?
- What tenant fairness policy should apply during 50,000-document bursts?
