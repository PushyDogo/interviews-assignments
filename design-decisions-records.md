# Design Decision Records

This file captures the major architecture choices for the Document AI case study. Each decision records the context, selected option, tradeoffs, and follow-up questions so the final artifacts can explain not just what we designed, but why.

## DDR-001 - Direct S3 Upload for Large Document Ingestion

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Use API Gateway + Lambda to create the job and return a short-lived S3 presigned POST policy. The client uploads the PDF directly to S3 instead of sending the file through the API layer.

**Context**

Documents can be up to 300 MB. The ingestion layer must support 10,000+ documents per day, bursts up to 50,000 documents, and should avoid tying up API compute while customers upload large files.

**Why this choice**

- Keeps the tenant-facing API lightweight and fast.
- Avoids API Gateway/Lambda payload-size and timeout constraints.
- Lets S3 handle large-object upload behavior, retry, durability, and Transfer Acceleration.
- Makes ingestion cost mostly storage/request based instead of compute-duration based.

**Tradeoffs**

- The platform must handle a two-step customer workflow: create job, then upload document.
- Customers need to calculate and provide a checksum before upload.
- The system must protect against abandoned jobs where a presigned POST policy is issued but no upload arrives.

**Operational controls**

- 30-minute presigned POST policy TTL.
- S3 Transfer Acceleration enabled by default in v1.
- Job expiry for uploads that are not completed in time.
- Object key scoped by tenant and job ID.
- Required checksum, file size, content type, and page-count validation before downstream processing starts.

**Future refinement**

- In v2, make Transfer Acceleration tenant-configurable or geography-aware so nearby customers do not pay for acceleration they do not need.

## DDR-002 - S3 Event-Triggered Upload Validation Before Starting Processing

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Start validation from an S3 `ObjectCreated` event after the customer uploads the file to the platform-generated key. The validation Lambda checks the object metadata, checksum, size, tenant/job mapping, schema ownership, and quota before accepting the job and starting the processing workflow.

**Context**

The case study requires sub-5-minute processing latency, but the clock should start after the file is uploaded and accepted, not when a presigned POST policy is issued. Starting Step Functions when the upload policy is created would create false latency pressure and processing failures for objects that do not exist yet. Since S3 already knows when the object arrives, using the object-created event keeps the v1 customer workflow simpler.

**Why this choice**

- Defines a clear `accepted_at` timestamp for latency measurement.
- Avoids requiring a second customer API call after upload.
- Gives the platform a deterministic validation step after object creation.
- Avoids accidentally processing incomplete, wrong, or abandoned uploads.
- Keeps the customer integration simple: create job, upload file, poll or wait for webhook.

**Tradeoffs**

- S3 events are asynchronous, so validation may start a few seconds after object creation.
- The platform must handle duplicate S3 events idempotently.
- If an event is delayed or missed, a reconciliation job must detect uploaded-but-not-validated objects.

**Operational controls**

- Idempotent validation Lambda keyed by tenant ID and job ID.
- Reconciliation job for uploaded-but-not-validated and created-but-not-uploaded jobs.
- Conditional DynamoDB updates to prevent duplicate workflow starts.
- Step Functions execution name derived from job ID.

**Future refinement**

- In v2, optionally add `POST /v1/jobs/{job_id}/complete-upload` as a manual finalization API for customers that want deterministic client-side confirmation or for integrations where S3 events are not the desired acceptance trigger.

## DDR-003 - DynamoDB as the Ingestion Job State and Client Request Store

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Use DynamoDB On-Demand for job state, `client_request_id` records, upload validation state, tenant quota counters, and lightweight cost-tracking fields during ingestion.

**Context**

The ingestion path needs low-latency conditional writes, retry-safe job creation, burst tolerance, and durable status lookup. It also needs to reject duplicate submissions and prevent the same `client_request_id` from being reused for different request content.

**Why this choice**

- Conditional writes are a good fit for idempotent job creation and one-time workflow start.
- On-Demand capacity absorbs bursty traffic without early capacity planning.
- DynamoDB integrates cleanly with streams, EventBridge Pipes, Lambda, and API Gateway-backed services.
- Cost is small relative to OCR and LLM usage at the expected job volume.

**Tradeoffs**

- Requires careful partition key design for tenant fairness and operational queries.
- Complex reporting should not be run directly from the operational table.
- Strong consistency should be used only where needed to control cost.

**Operational controls**

- Partition by tenant and job ID.
- Store request hash with `client_request_id`.
- Retain `client_request_id` records for 24 hours.
- Use TTL for abandoned upload records and short-lived retry records where appropriate.
- Emit state transitions to the audit pipeline.

**Retention rationale**

- A 24-hour retention window is enough for normal client retries, but not for week-later duplicate detection.
- Long-term audit should come from audit records and job state, not the retry/idempotency table.

## DDR-004 - Required Customer-Supplied Checksum

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Require the customer to provide a checksum during job creation and require the uploaded S3 object to match that checksum before the job is accepted.

**Context**

The platform must avoid sending corrupted, truncated, or accidentally mismatched files into OCR and LLM stages. Those downstream stages are the largest cost drivers, so ingestion must catch integrity problems early.

**Why this choice**

- Detects upload corruption and client-side file mismatches before processing.
- Makes retries and support investigations clearer.
- Reduces the chance of billing customers for work on the wrong file.

**Tradeoffs**

- Customer integration is slightly more complex because clients must calculate a checksum before upload.
- Very large files require client-side streaming checksum calculation.

**Operational controls**

- Store the expected checksum in the job record.
- Validate the S3 object checksum or metadata before acceptance.
- Reject checksum mismatches with `UPLOAD_REJECTED`.
- Do not consume quota for checksum-rejected uploads.

## DDR-005 - Consume Tenant Quota on Upload Acceptance

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Reserve or consume tenant processing quota only after the uploaded file passes ingestion validation and the job transitions to `ACCEPTED`.

**Context**

Creating a job and issuing an upload URL is cheap and may never lead to a real upload. Charging quota at job creation would penalize abandoned upload attempts and make customer billing harder to explain.

**Why this choice**

- Aligns quota usage with a real processing job.
- Avoids charging for abandoned upload URLs.
- Keeps cost and latency metrics anchored to `accepted_at`.

**Tradeoffs**

- A tenant could create many pending upload intents unless we separately rate-limit job creation.
- Quota availability must be checked again at acceptance time because quota may have changed since job creation.

**Operational controls**

- Rate-limit `POST /v1/jobs`.
- Check declared quota at creation for early rejection, but consume quota only at acceptance.
- If a job is cancelled or fails after acceptance, bill for the partial pipeline usage already consumed.
- Track partial usage by stage so cancellation and failure billing are explainable.

## DDR-006 - Pre-Registered Webhook Endpoints for v1

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

For v1, require tenants to use pre-registered webhook endpoints. A job may reference a `webhook_id`, but it should not submit an arbitrary callback URL per request.

**Context**

Per-job callback URLs are flexible, but they increase validation, SSRF, secret management, audit, and support complexity. Pre-registered endpoints are simpler and safer for v1.

**Why this choice**

- Reduces SSRF and data exfiltration risk.
- Makes webhook signing secrets easier to manage.
- Simplifies tenant onboarding and audit evidence.
- Keeps the ingestion API smaller.

**Tradeoffs**

- Customers have less per-job routing flexibility.
- Some customers may need multiple registered endpoints for different workflows.

**Future refinement**

- In v2, support both pre-registered webhook endpoints and per-job callback URLs, but only after adding strict allowlisting, domain verification, endpoint health checks, and per-endpoint signing keys.

## DDR-007 - Customer-Facing `client_request_id` Instead of `Idempotency-Key`

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Expose retry-safe job creation through a required `client_request_id` field in the `POST /v1/jobs` request body instead of asking customers to understand an `Idempotency-Key` header.

**Context**

Customers may retry `POST /v1/jobs` after network timeouts or uncertain API responses. The platform needs to return the original job instead of creating duplicates, but the API should remain intuitive.

**Why this choice**

- `client_request_id` is easier to explain than idempotency terminology.
- It can be generated by the customer or by our SDK.
- It keeps duplicate-submission protection explicit in the job creation payload.
- It avoids incorrectly using file checksum as the deduplication key, because the same file may be processed multiple times with different schemas or workflows.

**Operational controls**

- Scope by `tenant_id + client_request_id`.
- Store request hash with the record.
- Same tenant + same `client_request_id` + same request returns the original job.
- Same tenant + same `client_request_id` + different request returns a conflict.
- Retain records for 24 hours.

## DDR-008 - Rejected Upload Retention

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Retain rejected uploads for 7 days for audit and debugging, then move them to cold storage for another 21 days, then delete them completely.

**Context**

Rejected uploads can be useful for customer support, abuse investigation, compliance evidence, and debugging validation failures. Keeping them indefinitely conflicts with data-minimization and storage-cost goals.

**Why this choice**

- Gives operators a practical debugging window.
- Keeps customer support possible after failed uploads.
- Limits long-term data exposure and storage cost.
- Aligns with lifecycle-managed S3 storage.

**Operational controls**

- Store rejected uploads under a dedicated rejected/failed prefix.
- Apply lifecycle policy: 7 days hot/warm, 21 days cold, delete at 28 days.
- Keep audit metadata even after object deletion where compliance allows.
- Do not consume processing quota for rejected uploads.

## DDR-009 - API Key Plus Signed JWT Authentication for v1

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Use an API key plus a signed JWT bearer token for customer authentication in v1.

**Context**

The API is public in v1 and protected by WAF. We still need tenant identification, request authentication, token expiry, and authorization claims without building a full private networking model in the first version.

**Why this choice**

- API key identifies the tenant/application and supports quota/rate limiting.
- Signed JWT provides short-lived authorization claims.
- Works cleanly with API Gateway and Lambda authorizers.
- Is simpler than PrivateLink for v1 while remaining production-credible.

**Tradeoffs**

- Customers must manage both API key and token generation/refresh.
- Public API exposure still requires WAF, rate limits, monitoring, and abuse controls.

**Future refinement**

- Add PrivateLink in v2 for enterprise/tier-based tenants that require private connectivity.

## DDR-010 - Public API plus WAF in v1, PrivateLink in v2

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Expose ingestion through a public API protected by WAF in v1. Defer PrivateLink/private ingestion to v2 when tenant tiers and enterprise requirements are introduced.

**Context**

The case study requires a strong architecture, but v1 should keep the operational surface manageable. PrivateLink is valuable for enterprise tenants, but it adds onboarding, networking, DNS, cost allocation, and support complexity.

**Why this choice**

- Keeps v1 simpler and faster to operate.
- Public API plus WAF is sufficient for the baseline platform.
- Leaves a clear enterprise upgrade path.

**Operational controls**

- WAF managed rules.
- Per-tenant API throttling.
- API key and signed JWT authentication.
- CloudWatch and CloudTrail monitoring.
- GuardDuty/security monitoring where enabled.

## DDR-011 - Per-Stage Usage and Cost Ledger

**Status:** Accepted

**Stage:** Stage 1 - Ingestion, applies across all stages

**Decision**

Maintain a per-job usage and cost ledger that is updated at each pipeline stage. If a job is cancelled or fails after upload acceptance, the user is billed for the billable usage incurred up to that point.

**Context**

The platform must maintain real-time usage and costing. The case study has a strict cost target, and cancellation/failure semantics must be explainable to customers.

**Why this choice**

- Makes cost per job visible while the job is running.
- Supports fair billing for partial pipeline usage.
- Helps detect cost overruns early.
- Provides data for cost-per-stage optimization.

**Tradeoffs**

- Requires every stage to emit accurate usage events.
- Estimated cost during execution may need later reconciliation against AWS billing data.
- Billing logic must clearly distinguish platform errors, customer cancellations, validation rejections, and successful partial work.

**Operational controls**

- Ledger keyed by `tenant_id` and `job_id`.
- Track stage, elapsed time, service, usage quantity, usage unit, estimated cost, and billable flag.
- Initialize ledger at upload acceptance.
- Reconcile estimated cost with actual cloud/provider billing where needed.

## DDR-012 - Malware Scan Gate Before Job Acceptance

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Use GuardDuty Malware Protection for S3 to scan uploaded objects. Do not accept a job or start PDF parsing until the object has a clean malware scan result.

**Context**

Uploaded PDFs are untrusted customer content. Without malware scanning, the platform could accept malicious files into the processing pipeline and expose downstream workers, operators, or customer result consumers to risk.

**Why this choice**

- Uses an AWS-native malware scanning control for newly uploaded S3 objects.
- Keeps malware detection close to the ingestion boundary.
- Prevents malicious files from reaching OCR, LLM, or PDF parsing workers.
- Provides security events that can be audited and alerted on.

**Tradeoffs**

- Adds asynchronous wait time before job acceptance.
- Malware scan failures need a clear operational path.
- The job should not count against processing quota until the scan is clean.

**Operational controls**

- Track `MALWARE_SCAN_PENDING`, `MALWARE_REJECTED`, and `SECURITY_SCAN_FAILED`.
- Quarantine malicious objects.
- Alert on malware detections and scan failures.
- Do not consume processing quota for malware-rejected uploads.

## DDR-013 - Presigned POST with Upload Policy Guardrails

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Use presigned S3 POST policies rather than unconstrained upload URLs so the platform can enforce upload conditions at S3 before the object is stored.

**Context**

The upload boundary must reject oversized files, wrong content types, missing encryption headers, wrong KMS keys, and missing tenant/job metadata as early as possible.

**Why this choice**

- Enforces `content-length-range` before upload is accepted by S3.
- Requires exact `Content-Type`.
- Requires platform-generated object key.
- Requires server-side encryption with the tenant KMS key.
- Requires tenant/job/checksum metadata.

**Tradeoffs**

- Client upload implementation is slightly more involved than a simple PUT URL.
- POST policy generation must be carefully tested because field names and conditions must match exactly.

**Operational controls**

- 30-minute policy TTL.
- 1 byte to 300 MB `content-length-range`.
- Required `application/pdf` content type.
- Required `aws:kms` SSE and tenant KMS key ID.
- Required metadata for tenant, job, and checksum.

## DDR-014 - Per-Tenant KMS CMK for Document Data

**Status:** Accepted

**Stage:** Stage 1 - Ingestion, applies across all data stages

**Decision**

Use a per-tenant KMS CMK for document data and tenant-scoped artifacts. At minimum, use tenant-specific data keys with envelope encryption where direct per-tenant CMKs are not practical.

**Context**

Requirement 7.5 requires tenant scoping for data and keys. A single platform CMK creates an IAM-only tenant boundary. For a multi-tenant SaaS handling sensitive documents and GDPR erasure workflows, a cryptographic tenant boundary is stronger.

**Why this choice**

- Provides tenant-level cryptographic separation.
- Supports stronger access controls and audit evidence.
- Makes erasure and tenant offboarding easier to reason about.
- Limits blast radius if a key policy or role is misconfigured.

**Tradeoffs**

- More KMS keys and key policies to manage.
- Higher KMS request and monthly key cost as tenant count grows.
- Key deletion must be coordinated with retention and legal-hold obligations.

**Operational controls**

- Store tenant KMS key ARN in tenant configuration.
- Require the tenant key in the presigned POST policy.
- Validate object encryption before acceptance.
- Use encryption context with `tenant_id` and `job_id` where supported.
- Scope IAM decrypt permission by tenant/job execution context.

## DDR-015 - Immutable Audit Path with S3 Object Lock

**Status:** Accepted

**Stage:** Stage 1 - Ingestion, applies across all stages

**Decision**

Send structured audit events through EventBridge and Firehose into an S3 audit bucket with Object Lock in Compliance mode.

**Context**

CloudWatch Logs and CloudTrail alone do not satisfy immutable application audit requirements. SOC 2 evidence needs tamper-resistant records of job creation, upload, validation, rejection, acceptance, and workflow start.

**Why this choice**

- Gives the platform immutable audit records.
- Separates audit records from operational logs.
- Supports Athena/Lake Formation querying without modifying records.
- Makes state transitions explainable during audits and incidents.

**Tradeoffs**

- Object Lock retention must be chosen carefully because Compliance mode cannot be shortened by normal administrators.
- Audit schema must avoid storing raw document text or presigned URLs.

**Operational controls**

- S3 Object Lock Compliance mode.
- Versioning enabled.
- Bucket policy denies deletes and retention reduction.
- Platform-managed audit KMS key separate from tenant document keys.
- Alert on audit delivery failures.

## DDR-016 - Tenant Fairness and Admission Control at Ingestion

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Enforce tenant-level rate limits and admission control before accepting work into the pipeline. Return `429 Too Many Requests` with `Retry-After` when a tenant exceeds allowed burst or sustained rates.

**Context**

The platform must handle 50,000-document bursts without allowing one tenant to starve others or flood expensive downstream services.

**Why this choice**

- Protects shared platform capacity.
- Gives customers an explicit retry contract.
- Keeps expensive stages from being overloaded by a single tenant.
- Makes tenant-tier controls possible in v2.

**Tradeoffs**

- Requires rate-limit state and careful counter design.
- Tenants doing bulk uploads may need retry/backoff logic.

**Operational controls**

- Global WAF/API throttles.
- Tenant token buckets.
- Limits for job creation, pending uploads, acceptance rate, and active jobs.
- Sharded counters to avoid DynamoDB hot partitions.
- `429` response with `Retry-After`.

## DDR-017 - DynamoDB High-Cardinality Keys and Sharded Indexes

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Do not key the primary job table only by `tenant_id`. Use high-cardinality primary keys such as `JOB#<job_id>`, separate `CLIENTREQ#<tenant_id>#<client_request_id>` records, and sharded tenant/quota indexes for tenant listing and rate-limit counters.

**Context**

During a 50,000-document burst, a tenant-based partition key can create a hot partition if one tenant uploads a large batch.

**Why this choice**

- Distributes job writes across high-cardinality job IDs.
- Keeps retry-safe create lookups direct.
- Allows tenant listing through sharded GSIs.
- Supports high-write quota/rate counters without concentrating all writes in one partition.

**Tradeoffs**

- Tenant timeline queries must fan out across shards.
- The table model is more complex than a simple tenant-keyed design.

**Operational controls**

- Main job item: `PK=JOB#<job_id>`, `SK=META`.
- Client request item: `PK=CLIENTREQ#<tenant_id>#<client_request_id>`, `SK=REQUEST`.
- Tenant timeline GSI: `TENANT#<tenant_id>#SHARD#<n>`.
- Quota counters: `QUOTA#<tenant_id>#<window>#SHARD#<n>`.

## DDR-018 - Page-Count Validation Before Quota Consumption

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Validate page count during ingestion after malware scan is clean and before the job is accepted or quota is consumed.

**Context**

The platform limit is 1,000 pages. If quota were consumed before page count was known, a 5,000-page PDF could be accepted and then rejected in pre-processing, creating confusing billing and capacity behavior.

**Why this choice**

- Keeps the page-count requirement in the ingestion stage.
- Prevents over-limit documents from consuming processing quota.
- Avoids sending known-invalid files into OCR or LLM stages.

**Tradeoffs**

- Requires lightweight PDF parsing during ingestion.
- Parsing is done only after malware scan is clean to reduce risk.

**Operational controls**

- Use a Lambda/container image with `qpdf` or `pdfinfo`.
- Reject documents over 1,000 pages.
- Record page count in job metadata.
- Use page count for downstream cost estimation and quota decisions.

## DDR-019 - Automatic PII Scrubbing and Log Leakage Detection

**Status:** Accepted

**Stage:** Stage 1 - Ingestion, applies across all stages

**Decision**

Use emit-time structured log scrubbing plus CloudWatch Logs subscription-based PII leakage detection. The prevention control runs before logs are emitted; the subscription path detects and alerts if PII still leaks.

**Context**

Simply telling developers not to log PII is not a control. Filenames, metadata, error messages, and customer-supplied values can contain PII.

**Why this choice**

- Reduces PII leakage before logs are stored.
- Detects mistakes in logging policy or new fields.
- Provides evidence that log controls are working.

**Tradeoffs**

- Comprehend PII detection can be expensive if run on every log line.
- CloudWatch Logs subscriptions cannot rewrite already-ingested logs, so prevention must happen in the application logging layer.

**Operational controls**

- Structured logging library with denylisted fields and regex masking.
- Hash or sanitize filenames before logging.
- Subscription filter routes suspicious log events to a PII inspection Lambda.
- Inspection uses regex first and Comprehend PII detection selectively.
- Alert and create a security finding when PII leakage is detected.

## DDR-020 - JWT Trust Chain and Signing Key Rotation

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Use API keys for tenant/application identification and signed JWT bearer tokens for caller authentication and authorization. JWTs are issued by the platform identity service or an approved OIDC-compatible identity provider. Public verification keys are exposed through JWKS, and private signing keys are stored in KMS or Secrets Manager with controlled rotation.

**Context**

The previous design said "signed JWT" but did not define who issues it, where keys live, how callers verify trust, or how keys rotate.

**Why this choice**

- Gives a clear trust chain for the public ingestion API.
- Separates tenant/application identification from short-lived caller authorization.
- Supports key rotation without downtime.
- Fits API Gateway plus Lambda authorizer patterns.

**Operational controls**

- Validate `iss`, `aud`, `exp`, `nbf`, `iat`, `kid`, tenant claim, and scopes.
- Use short token lifetimes, typically 5-15 minutes for machine-to-machine calls.
- Use asymmetric signing such as `RS256` or `ES256`.
- Publish public keys through JWKS.
- Store private signing keys in KMS or Secrets Manager.
- Rotate signing keys on a fixed schedule and on compromise.
- Keep overlapping old/new public keys during token expiry windows.
- Never trust request-body `tenant_id` unless it matches the API key and JWT tenant claim.

## DDR-021 - PII-Free Immutable Audit for GDPR and SOC 2 Reconciliation

**Status:** Accepted

**Stage:** Stage 1 - Ingestion, applies across all stages

**Decision**

Keep immutable audit records PII-free. Store only operational identifiers and hashed references in the Object Lock audit stream, while customer content and PII remain in tenant-encrypted objects that can be deleted or crypto-shredded.

**Context**

SOC 2 requires tamper-resistant audit evidence. GDPR requires deletion of personal data when applicable. These goals conflict only if immutable audit records contain PII.

**Why this choice**

- Preserves immutable audit evidence for SOC 2.
- Keeps customer content deletable for GDPR erasure.
- Avoids storing filenames, document text, extracted fields, emails, SSNs, or presigned URLs in immutable records.
- Makes the erasure model explainable during an interview or audit.

**Operational controls**

- Audit records contain tenant ID, job ID, event type, timestamps, state transitions, actor IDs, hashed object keys, hashed client request IDs, and validation outcomes.
- Raw document content and extracted data are stored only in tenant-encrypted data stores.
- Erasure deletes input, intermediate, output, and indexed customer content.
- Tenant-key deletion or disabling is used only after retention/legal-hold checks.
- If needed, long-lived audit analytics can use HMAC-derived tenant references instead of direct tenant IDs.

## DDR-022 - Transport Security Enforcement

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Enforce TLS for all API and S3 ingestion traffic. API Gateway custom domains must require TLS 1.2 or newer, and S3 buckets must deny requests where `aws:SecureTransport` is false.

**Context**

SOC 2 expects transport encryption to be enforced, not just assumed. Public APIs and direct S3 uploads need explicit policy controls.

**Why this choice**

- Prevents accidental plaintext access to S3.
- Enforces modern TLS on customer-facing APIs.
- Provides clear evidence for SOC 2 controls.

**Operational controls**

- API Gateway security policy requires TLS 1.2+.
- S3 bucket policy denies `aws:SecureTransport=false`.
- S3 Block Public Access remains enabled.
- CloudTrail captures denied insecure attempts.
- WAF and API Gateway metrics alert on suspicious access patterns.

## DDR-023 - Deterministic Page Classification Before OCR

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Classify pages using deterministic PDF and image signals before deciding whether OCR is required. Avoid LLM calls during pre-processing.

**Context**

The cost target is tight. Sending every page through OCR or an LLM would quickly exceed the budget. Pre-processing must cheaply identify direct-text, scanned, hybrid, and low-quality pages.

**Why this choice**

- Keeps pre-processing cheap and fast.
- Reduces unnecessary Textract usage.
- Creates explainable routing decisions.
- Avoids using expensive model calls before we know they are needed.

**Tradeoffs**

- Heuristics can misclassify ambiguous pages.
- Ambiguous pages default to OCR in v1 to protect recall.

**Future refinement**

- Add a lightweight classifier in v2 once labeled production examples prove it reduces OCR cost without hurting recall.

## DDR-024 - Token-Aware Chunk Manifest

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Create a deterministic chunk manifest using both page-count limits and estimated token limits. The manifest becomes the contract for OCR and extraction stages.

**Context**

The case study asks for 20-50 page chunks, but page count alone does not protect downstream LLM context windows. Legal contracts and reports may be token-dense and need smaller chunks.

**Why this choice**

- Keeps downstream OCR and extraction parallelizable.
- Protects LLM context budgets.
- Makes replay deterministic.
- Gives operators a clear view of chunk count and route mix.

**Tradeoffs**

- Token estimates are imperfect before OCR.
- More chunks increase orchestration overhead.

**Decision details**

- For contracts and corporate reports, use a 25K estimated-token soft cap and a 30K hard cap per chunk in v1.
- Allow natural section boundaries up to the hard cap.
- Split sections above the hard cap and preserve cross-chunk references in the manifest.
- Keep token caps fixed in v1.
- Defer tenant-configurable or premium-tier higher caps to v2.
- Tune caps using production telemetry for token usage, extraction quality, latency, and cost.

## DDR-025 - Lambda First, Fargate for Heavy Image Processing

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Use Lambda for lightweight PDF profiling and page classification. Include ECS/Fargate CPU workers in v1 for heavy image enhancement, large page-range rendering, or documents likely to exceed Lambda timeout/memory/tmp-storage limits.

**Context**

Most documents should not need heavy image processing. But 1,000-page low-quality scans can exceed Lambda's practical runtime and temporary-storage envelope.

**Why this choice**

- Keeps the common path serverless and cheap.
- Provides a safe path for heavy scanned documents.
- Avoids forcing all documents through a more expensive worker model.

**Tradeoffs**

- Two execution paths add operational complexity.
- Fargate cold starts and scaling need monitoring.

**Decision details**

- ECS/Fargate is included in v1 for the heavy image path.
- Lambda remains the common path for lightweight profiling.
- Fargate usage is recorded in the per-job usage ledger.

## DDR-026 - Enhanced Image Retention Window

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Retain enhanced page images for 24 hours, then delete them.

**Context**

Enhanced images are intermediate artifacts used to improve OCR on scanned or low-quality pages. They can contain customer content and should not live longer than needed for retries.

**Why this choice**

- Supports short retry/replay windows.
- Aligns with the 24-hour retry/idempotency posture from Stage 1.
- Reduces storage cost and sensitive-data exposure.

**Tradeoffs**

- Operators have a shorter debugging window for enhancement artifacts.
- Reprocessing after 24 hours must regenerate enhanced images from the source PDF.

**Operational controls**

- Store enhanced images under a dedicated intermediate prefix.
- Apply S3 lifecycle deletion at 24 hours.
- Keep only non-PII metadata in the manifest after image deletion.

## DDR-027 - Schema Consistency Validation in Pre-processing

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Stage 2 validates that the uploaded document profile is broadly consistent with the `schema_id` selected in Stage 1.

**Context**

Stage 1 validates that the tenant is allowed to use the schema, but it cannot know whether the actual uploaded document matches the chosen schema without inspecting document structure.

**Why this choice**

- Catches obvious mismatches before OCR and LLM spend.
- Protects extraction quality.
- Creates a clear early warning when customers submit the wrong document type.

**Tradeoffs**

- This is a sanity check, not full classification.
- Strict mismatch handling could reject unusual but valid documents.

**Operational controls**

- Use cheap profile signals: layout markers, expected keywords, page-count shape, and document-family hints.
- If clearly mismatched, route to `MANUAL_INSPECTION_REQUIRED` by default.
- If ambiguous, continue but mark the manifest with a schema-consistency warning.

## DDR-028 - Stage 2 P95-Tail Handling for Large Low-Quality Scans

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Treat 1,000-page low-quality scanned documents as expected P95-tail workloads. Continue designing toward the overall sub-5-minute target, but do not optimize the median path around the worst-case scanned-document profile.

**Context**

The case study requires sub-5-minute P95 latency. A 1,000-page low-quality scan is the heaviest realistic document for pre-processing because it may require page rendering, orientation detection, image enhancement, and OCR preparation across many pages.

**Why this choice**

- Keeps the normal digital/mixed path fast and cost-efficient.
- Acknowledges that worst-case scanned documents will consume the slowest part of the latency distribution.
- Forces capacity planning for the heavy path without overcomplicating the common path.

**Operational controls**

- Parallelize page-range enhancement workers.
- Cap per-tenant worker concurrency to preserve fairness.
- Track P95/P99 latency separately by document profile.
- Alert when low-quality scanned jobs begin to degrade the broader P95 SLO.
- Use the usage ledger to show the additional cost of heavy image processing.

## DDR-029 - Separate Page and Chunk Manifests

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Write both `page-manifest.json` and `chunk-manifest.json`. The page manifest contains per-page routing, quality, enhancement, and criticality details. The chunk manifest references the page manifest and includes exact page lists and per-route page numbers for each chunk.

**Context**

An aggregate chunk summary is not enough for Stage 3. OCR needs to know exactly which pages need OCR, which can bypass OCR, which need enhanced images, and which are unsupported.

**Why this choice**

- Makes the Stage 2 to Stage 3 contract executable.
- Prevents accidental page drops.
- Preserves page-level provenance for downstream citation and merge logic.
- Allows unsupported-page policy to operate by criticality and field coverage.

**Operational controls**

- Store deterministic manifest hashes in DynamoDB.
- Include exact `page_routes` in each chunk.
- Include `unsupported_page_policy` per chunk.
- Version manifests by `preprocessing_algorithm_version`.

## DDR-030 - Conservative Token Estimation in Stage 2

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Use a model-agnostic conservative token estimator in Stage 2 and allow Stage 3 to recalculate exact model-specific prompt tokens for the selected inference path.

**Context**

Stage 2 must create chunks before OCR and before the final inference model path is fully applied. A model-specific tokenizer can be more accurate, but it may not apply across Bedrock and future self-hosted models.

**Why this choice**

- Keeps the chunk manifest portable across managed and hybrid inference paths.
- Avoids coupling Stage 2 to one model provider.
- Uses conservative estimates to reduce context-window risk.
- Lets Stage 3 perform exact token accounting when prompts are assembled.

**Operational controls**

- Use normalized character count for embedded text pages.
- Use layout density and tenant/schema averages for OCR-bound pages.
- Apply a 20% safety margin.
- Store raw and margin-adjusted estimates.
- Store `token_estimation_method` and `token_estimator_version`.

## DDR-031 - Stage 3 Fan-out Through Step Functions Distributed Map

**Status:** Accepted

**Stage:** Stage 2 to Stage 3 boundary

**Decision**

Launch Stage 3 OCR and structure extraction through Step Functions Distributed Map over the ordered chunk list in `chunk-manifest.json`.

**Context**

The case study requires elastic chunk-level processing. A hand-waved transition from pre-processing to OCR is not enough; the architecture must explicitly fan out chunks while controlling concurrency.

**Why this choice**

- Provides AWS-native elastic fan-out.
- Makes per-chunk retries and failure isolation easier.
- Supports concurrency limits to protect Textract, Bedrock, DynamoDB, and tenant fairness.
- Keeps the parent workflow durable and auditable.

**Operational controls**

- Distributed Map input is `chunk-manifest.json`.
- Each item includes chunk ID, page list, per-page routes, manifest URI, and flags.
- Bound global and per-tenant concurrency.
- Emit chunk-level state records before fan-out.

## DDR-032 - Schema Consistency Score Thresholds

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Use deterministic schema consistency scoring with explicit thresholds: continue at `>=0.70`, continue with warning at `0.40-0.69`, and route to `MANUAL_INSPECTION_REQUIRED` at `<0.40`.

**Context**

"Clearly mismatched" and "ambiguous" are not implementable without thresholds and signals. Stage 2 needs a reproducible policy for schema/document mismatch.

**Why this choice**

- Makes schema mismatch handling deterministic.
- Avoids hard failure for uncertain cases.
- Provides downstream stages with a warning when confidence is not strong.

**Operational controls**

- Score expected keyword/section match, layout/document-shape match, page-count range fit, text-density/table/form hints, and tenant historical profile fit.
- Record score and action in the manifest.
- Route high-confidence mismatch to manual inspection by default.

## DDR-033 - Unsupported Page Policy at Chunk Boundary

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Keep unsupported pages in the page and chunk manifests. Mark chunks with tolerated non-critical unsupported pages as `DEGRADED`; route unsupported critical pages or tolerance breaches to manual inspection.

**Context**

Unsupported pages cannot be dropped silently. Merge and validation need to know whether missing content affects critical fields, signatures, totals, clauses, or other business-important pages.

**Why this choice**

- Preserves field-coverage semantics for later stages.
- Treats critical pages differently from non-critical pages.
- Prevents a chunk success count from hiding missing important content.

**Operational controls**

- Default tolerance: zero unsupported critical pages.
- Default tolerance: up to two unsupported non-critical pages per chunk.
- Default tolerance: up to 1% unsupported non-critical pages per document.
- Store unsupported pages in `unsupported_page_policy`.

## DDR-034 - Lambda vs. Fargate Profiling Cutoff

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Use Lambda for profiling only when `page_count <= 250`, `file_size <= 100 MB`, and the document appears direct-text or mixed. Use ECS/Fargate for larger documents, broad rendering/enhancement, or retries after Lambda resource pressure.

**Context**

Lambda has real timeout, memory, and temporary-storage limits. A 1,000-page / 300 MB PDF can exceed the practical safety envelope even when it is mostly digital text.

**Why this choice**

- Keeps Lambda as the fast path without pretending it handles the upper bound safely.
- Gives large documents a bounded worker environment.
- Reduces timeout/retry waste.

**Operational controls**

- Route profiling to Fargate for `page_count > 250` or `file_size > 100 MB`.
- Route all broad image rendering/enhancement to Fargate.
- Retry Lambda resource failures on Fargate with smaller page ranges.
- Record selected worker type in metrics and usage ledger.

## DDR-035 - PDF Profiling and Rasterization Library Choices

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Use an adapter-based PDF profiling layer. Prefer PyMuPDF for profiling only if commercial licensing is approved; otherwise use pdfplumber for profiling. Use `pdf2image` with Poppler in ECS/Fargate for rasterization and image enhancement.

**Context**

The Stage 2 design needs concrete implementation choices. PDF library behavior, performance, and licensing matter for production readiness.

**Why this choice**

- PyMuPDF is fast and practical for metadata/text/render probes, but its AGPL/commercial licensing must be handled deliberately.
- pdfplumber is MIT-licensed and useful for layout/table inspection.
- Poppler-based rasterization is a standard, reliable path for page rendering in container workers.
- An adapter avoids locking the architecture to one library.

**Operational controls**

- Legal approval required before using PyMuPDF under commercial terms.
- Keep PDF operations behind an interface.
- Package Poppler in the ECS/Fargate worker image.
- Track parser/rasterizer version in the document profile.

## DDR-036 - Embedded Text Quality Gate Before DIRECT_TEXT Routing

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Do not trust an embedded PDF text layer based only on character count. Require text-quality sanity checks before routing a page to `DIRECT_TEXT`.

**Context**

PDFs often contain garbage text layers from poor OCR, broken encodings, invisible overlays, or copy-protection artifacts. Treating these as reliable direct text would create extraction errors.

**Why this choice**

- Improves extraction quality.
- Avoids false OCR bypass.
- Keeps routing decisions explainable.

**Operational controls**

- Check dictionary-word ratio.
- Check language detection confidence where applicable.
- Check garbage-text ratio: replacement characters, control characters, excessive single-character tokens.
- Use small rendered-sample comparison for suspicious pages when cheap.
- Route suspicious pages to `HYBRID` or `FULL_OCR`.

## DDR-037 - Enhanced Image Output Standard

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing

**Decision**

Use PNG at 300 DPI grayscale as the v1 default enhanced-image output. Use binary PNG for clean black-and-white forms, RGB only when color is semantically meaningful, and 400 DPI only for critical pages where profiling predicts 300 DPI will underperform.

**Context**

Enhanced image format affects OCR quality, S3 object size, upload time, and downstream latency. Textract page pricing is flat, but larger images still affect operational performance.

**Why this choice**

- PNG avoids JPEG compression artifacts.
- 300 DPI is a strong default for OCR quality without excessive image size.
- Grayscale usually preserves enough detail for text extraction.
- Avoiding default 600 DPI controls latency and storage.

**Tradeoffs**

- PNG can be larger than JPEG.
- Some documents may benefit from higher DPI or color.

**Operational controls**

- Default: PNG, 300 DPI, grayscale.
- Use binary PNG only when thresholding is safe.
- Use RGB only for color-dependent forms.
- Avoid TIFF as the default handoff format.
- Retain enhanced images for 24 hours.

## DDR-038 - Replay Billing Policy for Pre-processing

**Status:** Accepted

**Stage:** Stage 2 - Pre-processing, applies across all stages

**Decision**

Do not bill tenants for platform-initiated replays caused by platform failures, deployment defects, infrastructure errors, or idempotent recovery from partial writes. Bill tenants for customer-requested reprocessing, changed inputs, changed schema choices, or retries after customer-side cancellation where new processing work is intentionally requested.

**Context**

Stage 1 established that post-acceptance work is generally billable, but replay needs a carve-out. Without it, customers could be charged for platform recovery work.

**Why this choice**

- Keeps billing fair and explainable.
- Separates platform reliability recovery from customer-requested work.
- Supports the usage ledger without overcharging on retries.

**Operational controls**

- Usage ledger records `billable` and `billing_reason`.
- Replay requests include `replay_initiator`: platform, operator, or customer.
- Platform-initiated recovery replays are marked non-billable.
- Customer-requested reprocesses are billable.

## DDR-039 - Schema-Gated Text, Forms, and Tables Extraction

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Support text, forms, and tables in v1, but select the OCR/analysis API through tenant and schema policy. Use embedded text or raw Textract OCR as the default text path. Use structured Textract APIs such as Forms and Tables only for selected pages or regions where the schema requires stronger structure signals.

**Context**

The requirements explicitly call for text, forms, and tables extraction. At the same time, the case study targets less than `$0.10` per 100-page document. Textract raw OCR is already `$0.0015/page` for the first 1M pages/month, and structured OCR APIs can be 10x to 40x more expensive per page.

**Why this choice**

- Covers the functional requirement for text, forms, and tables.
- Keeps the cost-controlled path viable.
- Lets Stage 4 perform schema extraction with SLMs/LLMs instead of paying Textract structured API prices by default.
- Makes structured OCR an explicit schema and tenant-policy decision.
- Avoids silent cost cliffs during burst processing.

**Tradeoffs**

- Some forms and tables may require structured OCR to reach quality targets.
- Selective structured OCR requires better page/region routing and cost estimation.
- Stage 4 still has responsibility for schema extraction and validation on the default path.

**Operational controls**

- OCR policy object per tenant/schema.
- Usage ledger records raw OCR pages, structured OCR pages, and structured OCR regions separately.
- Stop-loss check before starting structured OCR.
- Audit event for every structured OCR decision.

## DDR-040 - Chunk-Level Async OCR with Page-Level Provenance

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Run OCR asynchronously at chunk or contiguous page-range granularity, while preserving source page numbers, page-route decisions, and source spans in the output artifact.

**Context**

The system needs elastic fan-out, but one OCR job per page would create high orchestration overhead and callback volume. At the same time, downstream extraction needs page-level citations and confidence.

**Why this choice**

- Reduces orchestration overhead compared with per-page OCR jobs.
- Keeps callbacks manageable under large bursts.
- Preserves page-level provenance for citations and validation.
- Allows direct-text pages to bypass OCR even inside mixed chunks.

**Tradeoffs**

- Temporary OCR input artifacts may be required for non-contiguous OCR page subsets.
- The system must maintain a page-mapping file from OCR input pages back to original document pages.
- Chunk-level retries need careful idempotency so they do not duplicate paid OCR work.

**Operational controls**

- OCR unit idempotency key includes tenant, job, chunk, OCR unit, policy version, and input artifact hash.
- Store OCR input page to source page mapping.
- Reuse existing Textract job IDs on retry.
- Keep temporary OCR input artifacts encrypted with the tenant KMS key.

## DDR-041 - Separate Callback Handling from OCR Result Normalization

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Use the Textract completion callback only to validate and record OCR completion. Retrieve, normalize, and write OCR results in a separate worker task.

**Context**

OCR callbacks should be fast, idempotent, and resilient to duplicate delivery. Heavy result retrieval and artifact construction can exceed callback handler time budgets and make duplicate callbacks harder to reason about.

**Why this choice**

- Keeps callback processing short and reliable.
- Makes duplicate callback handling simple.
- Allows normalization retries without starting another paid OCR job.
- Separates provider-notification correctness from artifact-generation correctness.

**Operational controls**

- Callback handler validates Textract job ID against chunk state.
- Duplicate callbacks are acknowledged without reprocessing.
- Unknown callbacks go to DLQ.
- Normalization worker reuses completed Textract result pointers.

## DDR-042 - Stage 3 Output as a Normalized OCR/Structure Artifact

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Write one normalized `ocr-structure.json` artifact per chunk. The artifact includes ordered page blocks, text, confidence, bounding boxes, source spans, route/source metadata, warning flags, and usage counts.

**Context**

Stage 4 needs a stable contract that works whether a page came from embedded text, raw OCR, hybrid merge, or a premium structured API. Without normalization, every downstream stage would need provider-specific logic.

**Why this choice**

- Gives Stage 4 a consistent input.
- Preserves page citations and source spans.
- Keeps provider-specific OCR responses out of AI extraction code.
- Supports replay and auditing through artifact hashes and policy versions.

**Tradeoffs**

- Requires a normalization layer.
- Some provider-specific details may need to be carried as optional metadata.

**Operational controls**

- Artifact is encrypted with the tenant KMS key.
- Artifact URI and hash are stored in chunk state.
- Raw OCR text is never logged.
- Artifact version, OCR policy version, and source manifest hashes are recorded.

## DDR-043 - OCR Concurrency Guardrails Before Paid Work

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Enforce global, tenant-level, and per-job OCR concurrency limits before starting paid OCR jobs.

**Context**

The platform must absorb 50,000-document bursts without allowing one tenant or one large job to consume all OCR capacity. Textract quotas and callback throughput are finite, and uncontrolled fan-out would harm latency and cost predictability.

**Why this choice**

- Preserves tenant fairness.
- Protects Textract quotas.
- Keeps queue growth visible and manageable.
- Supports predictable degradation when downstream services throttle.

**Tradeoffs**

- Some chunks may wait even when the parent workflow has fanned out.
- Admission control must consider Stage 3 backlog, not just Stage 1 API traffic.

**Operational controls**

- Initial global active OCR unit limit of `100`.
- Initial base tenant active OCR unit limit of `5`.
- Initial per-job active OCR chunk limit of `2`.
- Initial structured OCR active unit limit of `1` per tenant.
- Fixed v1 limit of `50` queued-or-active structured OCR pages per base tenant.
- Return `429 Too Many Requests` with `Retry-After` at job creation/acceptance when projected structured OCR work would exceed the tenant limit.
- Keep already accepted Stage 3 chunks queued when capacity is temporarily unavailable.
- Queue-age metrics and SLO burn alerts.
- Circuit breaker during provider throttling or elevated error rates.

**Future refinement**

Tune limits after load testing against approved Textract service quotas, callback throughput, Step Functions concurrency, DynamoDB write capacity, queue age, and cost metrics. V1 has only base-level tenants; tier-based OCR concurrency is deferred to v2.

## DDR-044 - Region-Level OCR Preferred for Hybrid Pages

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Prefer region-level OCR for hybrid pages when Stage 2 provides trustworthy region coordinates and the number of regions is small enough to reduce cost or improve quality. Fall back to full-page OCR when region coordinates are missing, unreliable, or when many region crops would cost more than one page-level OCR call.

**Context**

Hybrid pages can contain reliable embedded text plus image-only sections such as stamps, signatures, scanned tables, and handwritten notes. Full-page OCR is simple but may duplicate text and add cost. Region-level OCR is more targeted but depends on Stage 2 region quality.

**Why this choice**

- Minimizes OCR cost where only a small part of the page needs OCR.
- Reduces duplicate OCR text on mostly digital pages.
- Preserves important image-only regions.
- Keeps a safe fallback when region detection is not reliable.

**Tradeoffs**

- Requires coordinate mapping and merge logic.
- Multiple crops from one page can create more OCR units than full-page OCR.
- Bad region coordinates can miss content, so Stage 2 confidence matters.

**Operational controls**

- Use region-level OCR only when `region_confidence >= 0.80`.
- When separate signals are available, require `bbox_confidence >= 0.85` and `content_classification_confidence >= 0.75`.
- Cap region crops per page.
- Use full-page OCR when crop count or uncertainty exceeds the policy.
- Preserve provenance for embedded, region OCR, and full-page OCR blocks.

## DDR-045 - Critical OCR Confidence Escalation Ladder

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

When OCR confidence is low on a critical page, use a three-tier escalation ladder: re-run OCR with different settings, route the affected chunk to the stronger extraction model or OCR/analysis policy available in v1, then send the chunk to manual inspection if automated recovery fails.

**Context**

A critical page cannot be treated like a non-critical warning. At the same time, failing the whole job immediately wastes work from other chunks and creates poor operational behavior.

**Why this choice**

- Gives the system an automated recovery path before manual inspection.
- Keeps other chunks moving where possible.
- Makes cost escalation explicit and auditable.
- Routes unresolved critical uncertainty to a human rather than returning a false result.

**Operational controls**

- Tier 1: alternate enhancement, rotation, DPI, or full-page OCR after region OCR underperforms.
- Tier 2: stronger extraction model or structured OCR policy by default for all base tenants in v1.
- Tier 3: manual inspection queue with page, region, confidence, and artifact pointers.
- Usage ledger marks whether escalation is billable and why.

**Future refinement**

Make stronger-model recovery tenant-configurable in v2.

## DDR-046 - Short-Lived Normalized OCR Artifacts

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Retain normalized OCR artifacts only until final extraction output is accepted and the retry/replay window closes. Do not keep OCR artifacts for the full 365-day customer document retention period by default.

**Context**

OCR artifacts can contain the same PII and sensitive business data as the source document. Keeping them after extraction acceptance increases storage cost and privacy exposure without adding much customer value.

**Why this choice**

- Supports GDPR data minimization.
- Reduces storage and leakage risk.
- Keeps final accepted extraction output as the long-lived customer-facing artifact.
- Still allows short-window retries and manual inspection.

**Tradeoffs**

- Later customer-requested reprocessing may need to replay OCR from the retained source document.
- Debugging older extraction issues relies on hashes, audit records, and final outputs rather than raw OCR artifacts.

**Operational controls**

- Delete `ocr-structure.json` after final extraction acceptance and replay-window closure.
- Keep only non-PII hashes, counts, policy versions, timings, and cost records in audit/job state.
- Customer-requested reprocessing after deletion is billable when it requires new OCR work.

## DDR-047 - Representative-Mix Cost Target for Stage 3 OCR

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Treat the `$0.10/100-page` target as a representative document-mix target, not as a guarantee for every scanned-heavy document. Use an explicit v1 modeling assumption of 60% digital-text pages, 30% hybrid pages, and 10% fully scanned pages unless later workload evidence changes it.

**Context**

A fully scanned 100-page document costs about `$0.15` in raw Textract OCR before LLM, orchestration, storage, and observability. That means the platform cannot honestly claim the target for every scanned-heavy workload.

**Why this choice**

- Makes the unit economics defensible.
- Avoids hiding a known cost exception in a footnote.
- Forces scanned-heavy documents into an explicit policy path.
- Keeps cost claims tied to document mix instead of worst-case wishful thinking.

**Operational controls**

- Stage 2 estimates scanned-page ratio.
- Stage 3 calculates projected OCR cost before paid work starts.
- Cost policy result is recorded as `WITHIN_TARGET`, `MIX_EXCEPTION`, `CUSTOMER_APPROVAL_REQUIRED`, or `REJECTED_BY_COST_POLICY`.
- Scanned-heavy documents can require approval, manual inspection, or different pricing.

## DDR-048 - Tenant-KMS Textract OutputConfig

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Every Textract async request must pass `OutputConfig.S3Bucket`, tenant-scoped `OutputConfig.S3Prefix`, and `OutputConfig.KMSKeyId` set to the tenant CMK.

**Context**

Textract async jobs can write service output outside the platform's normalized artifact path. If the platform does not control the output bucket and KMS key, raw OCR output can fall outside the per-tenant CMK isolation model established in Stage 1.

**Why this choice**

- Preserves tenant KMS isolation for raw OCR output.
- Keeps raw Textract output in tenant-scoped S3 prefixes.
- Supports GDPR deletion and crypto-shredding.
- Keeps security posture consistent across original, intermediate, and normalized artifacts.

**Operational controls**

- Validate tenant KMS key before OCR start.
- Pass tenant-scoped S3 output prefix on every async Textract call.
- Bucket policy denies writes without expected SSE-KMS context where supported.
- Normalization worker reads only from the expected tenant/job/chunk prefix.

## DDR-049 - Verified OCR Callback and Bounded Wait

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Verify SNS callback integrity before accepting Textract completion messages and enforce bounded callback waits. Use a 3-minute poll fallback and a 5-minute extended ceiling only for large-scan P95-tail jobs.

**Context**

Validating that a Textract job ID exists is necessary but not enough. The callback path must verify message integrity and avoid waiting indefinitely when callbacks are delayed or lost.

**Why this choice**

- Prevents forged or replayed completion messages.
- Avoids stuck `OCR_CALLBACK_WAITING` chunks.
- Provides a deterministic fallback when callbacks are delayed.
- Keeps the Stage 3 latency budget visible.

**Operational controls**

- Verify SNS message signature and expected topic ARN.
- Reject stale callback messages outside the freshness window.
- Poll Textract status once after 3 minutes.
- Route timed-out OCR units to DLQ after the final wait ceiling.

## DDR-050 - OCR Poison-Pill Isolation

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Move OCR units or chunks into `OCR_POISON` after bounded retry attempts or deterministic failures. Isolate poison work in DynamoDB, an OCR poison DLQ, and a tenant/job-scoped diagnostics artifact.

**Context**

Requirement 4.6 requires poison-pill handling. Naming an `OCR_POISON` state is not sufficient; the platform needs explicit triggers, cost bounds, operator actions, and replay paths.

**Why this choice**

- Prevents infinite retries and uncontrolled OCR spend.
- Keeps bad chunks from blocking unrelated chunks.
- Gives operators enough information to replay, inspect, or fail the affected work.
- Preserves auditability without logging raw OCR content.

**Operational controls**

- Default retry budget: one initial paid OCR attempt plus up to two automated recovery attempts.
- Immediate poison for deterministic invalid input and confirmed KMS/IAM denial.
- OCR poison DLQ stores compact non-PII failure messages.
- Diagnostics artifact stores IDs, hashes, counts, policy versions, attempt history, and artifact pointers.
- Operator tool supports replay normalization, replay OCR, alternate settings, manual inspection, or terminal failure.

## DDR-051 - Explicit Stage 3 Latency Budget

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Allocate Stage 3 a v1 latency budget of P50 `<= 60s`, P95 `<= 180s`, and P99 `<= 240s`, measured from `OCR_QUEUED` to a Stage 3 terminal state.

**Context**

The platform has a 5-minute end-to-end P95 target after upload acceptance. Since async OCR can take from seconds to minutes, Stage 3 needs an explicit budget so downstream stages still have time for AI extraction, merge, validation, storage, and notification.

**Why this choice**

- Makes the SLA allocation defensible.
- Gives admission control a concrete queue-age threshold.
- Forces OCR-heavy workloads to be identified as P95-tail risk.
- Leaves roughly 90 seconds for later stages inside the end-to-end P95 target.

**Operational controls**

- Track `ocr_stage_latency_ms` and `ocr_queue_age_ms`.
- Use admission backpressure when projected Stage 3 latency exceeds budget.
- Use 3-minute poll fallback before the P95 budget is exhausted.
- Mark scanned-heavy jobs as P95-tail risk.

## DDR-052 - Quantified OCR Confidence Thresholds

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Use quantified default confidence thresholds for low-confidence handling. On normalized `0.0-1.0` confidence, critical required field blocks below `0.90`, critical page average below `0.85`, more than `5%` critical-page blocks below `0.90`, or table/form structural confidence below `0.85` trigger the critical-page escalation ladder.

**Context**

The design previously referenced low OCR confidence without defining what low meant. Textract confidence is numeric, and production behavior must be deterministic and tunable.

**Why this choice**

- Makes escalation implementable.
- Lets schemas tune quality requirements by field importance.
- Avoids treating all OCR uncertainty as equal.
- Provides clear metrics for manual-review and quality monitoring.

**Operational controls**

- Store confidence thresholds in schema policy.
- Normalize Textract `0-100` confidence to `0.0-1.0` in Stage 3 artifacts.
- Emit low-confidence page and block metrics.
- Defer tenant-specific threshold overrides to v2.

## DDR-053 - Single-Region Textract in v1 with Outage Backpressure

**Status:** Accepted

**Stage:** Stage 3 - OCR and Structure Extraction

**Decision**

Use Textract in the platform's primary AWS region for v1. Do not enable automatic cross-region OCR failover in v1. During sustained Textract errors or outages, open the Stage 3 circuit breaker, stop starting new OCR jobs, queue accepted chunks where possible, and return `429` for new OCR-heavy work.

**Context**

The platform targets 99.9% availability, but Textract is a managed regional dependency. Cross-region failover would improve resilience but adds data residency, tenant KMS, quota, callback routing, and audit complexity.

**Why this choice**

- Keeps v1 operationally testable and compliant with the tenant CMK model.
- Avoids unplanned cross-region data movement.
- Makes the outage dependency explicit.
- Preserves a clean normalized artifact contract for future fallback providers.

**Tradeoffs**

- A prolonged Textract regional outage can delay OCR-heavy jobs in v1.
- Availability for OCR-heavy workloads depends on AWS regional Textract health.

**Operational controls**

- Stage 3 circuit breaker on sustained Textract error/throttle rate.
- Admission backpressure and `429 Retry-After` for new OCR-heavy work.
- Provider-outage runbook and tenant notification for long outages.
- V2 evaluates cross-region Textract, secondary OCR provider, or self-hosted OCR fallback.

## DDR-054 - Managed-First AI Extraction with Bedrock Llama 3.1

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Use Amazon Bedrock as the default v1 inference path for chunk-level AI extraction. Use `meta.llama3-1-8b-instruct-v1:0` as the default text model, and use `meta.llama3-1-70b-instruct-v1:0` for critical-field recovery.

**Context**

`meta.llama3-2-11b-instruct-v1:0` is a vision/multimodal model, not the right text-only default for extraction. Llama 3.1 8B Instruct is cheaper, text-focused, supports batch pricing, and provides a cleaner direct substitute when the platform later moves to self-hosted inference. The case study also requires explaining a self-hosted EKS option, but managed inference is simpler to operate for v1.

**Why this choice**

- Reduces v1 operational complexity.
- Aligns managed and future self-hosted model families.
- Reduces migration risk when moving from Bedrock to vLLM/TGI.
- Keeps self-hosted inference as a deliberate optimization rather than an early dependency.

**Tradeoffs**

- `aws-pricing.md` uses the us-east-1 Bedrock Llama 3.1 8B rate of `$0.22/$0.22 per 1M input/output tokens`; batch is `$0.11/$0.11` and is only for non-interactive workloads.
- 70B recovery pricing remains a planning estimate until verified.
- Llama extraction quality must be validated against golden datasets before launch.
- Model availability and quotas remain managed-service dependencies.
- Self-hosted GPU economics may become better at high utilization.

**Operational controls**

- Record model ID, route, prompt version, schema version, input tokens, output tokens, and cost for every chunk.
- Use Llama 3.1 70B only for recovery paths in v1.
- Use Llama 3.1 8B batch only for offline replay, backfill, and evaluation workloads.
- Compare managed and self-hosted model quality before production routing.

## DDR-055 - Schema-Constrained Chunk Extraction Artifact

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Stage 4 writes one schema-valid `chunk-extraction.json` artifact per chunk. Each extracted field includes value, confidence, validation status, and source citations back to Stage 3 page/block/source-span metadata. V1 gets structured output through Bedrock Converse tool use: declare one `submit_extraction` tool whose input schema is generated from the tenant/schema JSON Schema, and accept only a valid tool call.

**Context**

Stage 5 needs stable chunk outputs to merge and reconcile document-level meaning. If Stage 4 emits free-form or uncited output, downstream validation cannot reliably explain or correct results.

**Why this choice**

- Keeps Stage 5 provider-agnostic.
- Preserves provenance from final fields back to source pages.
- Makes partial success and manual review field-aware.
- Allows deterministic validators to operate before and after merge.

**Tradeoffs**

- Requires strict schema validation and repair handling.
- Some document-level fields cannot be resolved within a single chunk.

**Operational controls**

- JSON parse and JSON Schema validation before accepting output.
- Bedrock Converse tool-use output with exactly one `submit_extraction` call.
- Citation validation for extracted fields.
- `requires_document_synthesis` marker for contracts/reports and cross-chunk fields.
- Store source OCR artifact hash and prompt/model/schema versions.

## DDR-056 - Exact Token Counting Before Model Invocation

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Calculate exact model-specific input tokens before every model invocation. Do not call the model if the prompt exceeds the hard token cap; request sub-splitting or manifest replay instead.

**Context**

Stage 2 uses conservative token estimates for chunking, but Stage 4 owns the actual prompt and model route. Token overruns can cause model failures, high latency, or cost spikes.

**Why this choice**

- Prevents avoidable model failures.
- Keeps cost estimates honest.
- Makes prompt compression and chunk splitting deterministic.
- Protects the 5-minute end-to-end latency target.

**Operational controls**

- Target input cap `25K` and hard input cap `30K` per chunk in v1.
- Target output cap `1.5K` and hard output cap `2K` per chunk in v1.
- Store actual token counts and tokenizer/model version.
- Fail closed when estimated and actual counts diverge materially.

## DDR-057 - Prompt and Schema Versioning for MLOps

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Version prompts, schemas, extraction policies, model routes, few-shot retrieval policy, and evaluation datasets. Record these versions in every chunk artifact and usage ledger entry.

**Context**

Extraction quality depends on prompt, schema, model, OCR quality, and document type. The case study requires model evaluation, canary rollouts, and rollback support.

**Why this choice**

- Makes quality regressions traceable.
- Enables prompt/schema/model rollback.
- Supports 10% canary over a 1-hour soak.
- Allows field-level precision/recall/F1 analysis by model and schema version.

**Operational controls**

- CI validates schema and prompt packages.
- Canary tracks latency, error rate, quality, token usage, cost, and manual-review rate.
- Golden datasets cover invoices, contracts, forms, and reports.
- Deployments block on critical eval failures.

## DDR-058 - Prompt Injection Treated as Data-Plane Risk

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Treat OCR text and document content as untrusted data. Keep extraction instructions outside document text, validate outputs deterministically, and never allow model output to control platform actions.

**Context**

Documents may contain text that attempts to instruct the model. Since Stage 4 turns OCR text into prompts, prompt injection is a practical security and correctness risk.

**Why this choice**

- Prevents document text from overriding extraction policy.
- Keeps model output from affecting IAM, S3 paths, webhook destinations, or billing.
- Supports SOC 2-style control evidence for AI behavior.

**Operational controls**

- System/schema instructions are separated from document text.
- Prompt explicitly says document text is data, not instructions.
- JSON Schema and citation validation are mandatory.
- Platform decisions are made by deterministic code, not model output.

## DDR-059 - Few-Shot Retrieval in v1

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Enable few-shot retrieval in v1 for Stage 4 extraction. Retrieved examples must match the same `schema_id`, document type, and prompt major version, and are capped at three examples and 4K input tokens per chunk.

**Context**

The quality of LLM extraction and summaries matters for the interview case study. Schemas vary by document type and tenant workflow, so short schema-matched examples can improve field formatting, citation behavior, and JSON consistency.

**Why this choice**

- Improves extraction quality for schema-specific fields.
- Helps smaller Llama models follow expected output patterns.
- Keeps the example budget bounded.
- Provides a clear evaluation lever in canaries.

**Operational controls**

- Prefer tenant-approved examples; otherwise use platform-curated examples with no tenant PII.
- Never retrieve examples from another tenant's private data.
- Store example IDs and retrieval policy version in the chunk artifact.
- Disable examples for a schema if canary results show token cost without quality lift.

## DDR-060 - Stage 4 Latency and Cost Stop-Loss

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Use a Stage 4 v1 latency budget of P50 `<= 20s`, P95 `<= 60s`, and P99 `<= 90s`. Use a model-cost target of `<= $0.025/100 pages`, soft stop-loss above `$0.035/100 pages`, and hard stop-loss above `$0.050/100 pages`.

**Context**

Stage 3 can consume up to 180 seconds at P95, so Stage 4 cannot have an unbounded model/repair loop. The full platform target is `$0.10/100 pages`, and OCR can consume most of that for scanned-heavy documents.

**Why this choice**

- Preserves the end-to-end 5-minute P95 budget.
- Keeps model spend bounded while Llama 3.1 pricing and quality are validated.
- Gives admission control concrete queue-age and projected-cost thresholds.
- Leaves room for Stage 5 merge, post-processing, storage, and notification.

**Operational controls**

- Track `extraction_stage_latency_ms`, queue age, model latency, TTFT, and token counts.
- Apply backpressure if queue age plus projected model latency exceeds P95 budget.
- Do not naively add per-stage P95 budgets; chunks fan out and stages overlap across chunks, so critical-path latency and queue age drive the end-to-end SLA.
- Initial concurrency: global active Bedrock calls `100`, base tenant active calls `5`, per-job active extraction chunks `5`, stronger recovery calls per tenant `1`, batch jobs per tenant `1`.
- Bedrock quota-increase requests for selected Llama 3.1 models are part of launch readiness.
- Pause or require approval when hard cost stop-loss is crossed.
- Tune budgets by schema/model after production telemetry.

## DDR-061 - Non-Billable Stronger-Model Recovery in v1

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Do not bill tenants separately for stronger-model recovery in v1. Treat it as platform quality recovery. In v2, make stronger-model recovery customer opt-in and billable by tier or tenant policy.

**Context**

Stage 4 may need a larger Llama 3.1 model to recover critical-field extraction failures. In v1 there are only base-level tenants, and charging for recovery would make billing harder to explain before tiered controls exist.

**Why this choice**

- Keeps v1 billing simple.
- Avoids surprising customers with recovery charges.
- Lets the platform collect recovery-rate and cost data before pricing it.
- Preserves v2 room for tiered quality/cost options.

**Operational controls**

- Usage ledger still records recovery model cost and `billable=false`.
- Track recovery rate by schema, document type, OCR quality, and model version.
- Alert on recovery cost spikes.
- V2 introduces opt-in stronger recovery policy.

## DDR-062 - Bounded Context Window Strategy

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Do not try to fit full large documents into the Llama 3.1 128K context window. Keep Stage 4 chunk prompts capped at 30K input tokens, reserve context for schema, examples, output, and safety margin, and defer cross-chunk reasoning to Stage 5 hierarchical synthesis.

**Context**

The case study explicitly challenges how the system handles documents whose content plus prompt exceed the model context window. A 1,000-page document can easily exceed 128K tokens even before schema instructions and examples are added.

**Why this choice**

- Prevents context-window failures.
- Keeps latency and cost bounded.
- Preserves source citations by avoiding silent truncation.
- Separates chunk extraction from document-level synthesis.

**Operational controls**

- Exact-token count before model invocation.
- Drop repeated furniture and compact tables before sub-splitting.
- Trim few-shot examples before source content.
- Request Stage 2/3 sub-splits when a chunk exceeds the hard cap.
- Mark cross-chunk fields with `requires_document_synthesis=true`.
- Stage 5 uses hierarchical synthesis over chunk outputs and relevant cited source spans.

## DDR-063 - Managed Model Outage and Bake-Off Policy

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Define a managed-model outage policy and a v1 model bake-off list. If Bedrock Llama 3.1 is throttled or unavailable, Stage 4 uses bounded retries, opens a circuit breaker on sustained errors, queues accepted chunks where possible, and returns `429 Retry-After` for new extraction-heavy work. The v1 bake-off candidates are Llama 3.1 8B, Llama 3.1 70B, and Mistral 7B Instruct.

**Context**

The default model is accepted only if evaluation confirms field-level quality. The system also needs a provider-outage story for 99.9% availability and operational runbooks.

**Why this choice**

- Avoids pretending one model is guaranteed to meet quality and availability requirements.
- Gives launch readiness a concrete evaluation set.
- Keeps model outage behavior predictable.
- Preserves a fallback path without changing the Stage 4 artifact contract.

**Operational controls**

- Evaluate candidates on golden datasets for field-level F1, critical-field F1, citation validity, schema validation success, latency, cost per 100 pages, and manual-inspection rate.
- Stage 4 circuit breaker on sustained Bedrock model errors or throttling.
- Accepted chunks queue where possible; new extraction-heavy work gets `429 Retry-After`.
- V2 can add self-hosted Llama or secondary managed provider fallback behind the same artifact contract.

## DDR-064 - Bedrock Guardrails Required in v1

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Enable Bedrock Guardrails for all v1 Stage 4 model invocations. Configure PII filtering, prompt-injection/jailbreak denial, and tenant-configured denied-topic policies. Do not allow normal model invocations to bypass guardrails.

**Context**

Stage 4 sends OCR text and extracted content through model prompts. In a multi-tenant GDPR/SOC 2 system, guardrails are a baseline data-plane control, not an optional enhancement.

**Why this choice**

- Adds a managed control layer for prompt and output safety.
- Supports SOC 2 evidence for AI interaction controls.
- Reduces PII and prompt-injection risk.
- Makes tenant topic restrictions enforceable at model boundary.

**Operational controls**

- Guardrail ID/version recorded in every chunk artifact.
- Break-glass bypass requires operator approval and immutable audit event.
- Guardrail block/intervention rates are monitored by schema/model version.
- Guardrail configuration changes follow prompt/schema canary process.

## DDR-065 - Prompt Management and Schema Registry Split

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Use Bedrock Prompt Management for versioned prompt templates and prompt variants. Store tenant schema definitions, JSON Schemas, extraction policies, selector expressions, and field-criticality rules in a tenant-scoped schema registry backed by DynamoDB metadata and versioned S3 documents.

**Context**

Prompt templates and tenant schemas have different lifecycles. Bedrock Prompt Management is useful for prompt rollout/versioning, but tenant schema definitions and extraction policies still need tenant-scoped storage, auditability, and application-level validation.

**Why this choice**

- Uses Bedrock-native prompt versioning where it fits.
- Keeps schemas under tenant-scoped governance.
- Supports independent rollback of prompts, schemas, and model routes.
- Makes prompt/schema versions visible in every extraction artifact.

**Operational controls**

- Record `prompt_version`, prompt ARN, `schema_version`, and extraction policy version.
- CI validates prompt references and JSON Schemas.
- Schema documents are encrypted with tenant KMS where tenant-specific.
- Canary prompt/schema changes independently.

## DDR-066 - Local Llama Token Counting

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Compute prompt token counts locally using the Meta Llama 3.1 tokenizer, for example HuggingFace `tokenizers` with `meta-llama/Llama-3.1-8B-Instruct` or an approved equivalent BPE artifact. Do not claim Bedrock provides a count-only token endpoint for Llama.

**Context**

Stage 4 requires exact-enough token accounting before invocation, but Bedrock does not expose a count-only endpoint for Llama. Token counting must therefore be implemented and versioned in the worker.

**Why this choice**

- Makes token preflight implementable.
- Prevents context-window failures before paid calls.
- Keeps tokenizer behavior auditable and reproducible.
- Supports drift detection against provider usage metrics.

**Operational controls**

- Pin tokenizer version and checksum in the worker image.
- Store tokenizer name, version, and hash in chunk artifacts.
- Compare local counts with provider usage metrics after invocation.
- Fail closed when divergence exceeds the configured threshold.

## DDR-067 - Few-Shot Retrieval Through Bedrock Knowledge Bases

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Use Bedrock Knowledge Bases over OpenSearch Serverless for v1 few-shot retrieval. Use Titan Embeddings V2, cosine similarity, `top_k=3`, and mandatory filters for tenant, schema, document type, prompt major version, and `pii_safe=true`.

**Context**

The earlier design enabled few-shot retrieval but did not specify the architecture. Retrieval must be tenant-safe, bounded, and cost-controlled.

**Why this choice**

- Uses managed AWS retrieval infrastructure.
- Keeps few-shot examples schema-specific and tenant-safe.
- Bounds retrieval cost and prompt growth.
- Provides a path to evaluate quality lift per schema.

**Operational controls**

- Never retrieve private examples across tenants.
- Store retrieved example IDs and retrieval policy version.
- Cap examples at three and 4K input tokens.
- Disable retrieval for schemas where canaries show no quality lift.

## DDR-068 - Citation-Gated Non-Null Fields

**Status:** Accepted

**Stage:** Stage 4 - AI Extraction

**Decision**

Reject every extracted non-null field value that lacks a valid non-empty source citation. Mark the field as `not_found` with `rejection_reason="MISSING_CITATION"` instead of accepting uncited values.

**Context**

Schema-valid JSON alone does not prevent hallucination. The system needs field-level provenance from extracted values back to Stage 3 source spans.

**Why this choice**

- Prevents uncited hallucinated values from entering Stage 5.
- Makes manual review and customer explanations traceable.
- Preserves field coverage semantics for partial success.
- Keeps final outputs defensible in regulated workflows.

**Operational controls**

- Citation resolver validates page/block/source span against Stage 3 artifact hash.
- Required critical fields marked `not_found` route to stronger recovery or manual inspection.
- Stage 5 can only merge cited values or explicit `not_found` markers.

## DDR-069 - Deterministic Merge Before Document Synthesis

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Merge Stage 4 chunk extraction artifacts with deterministic schema-driven rules before using any model-based document synthesis. Every schema field must declare a merge rule such as first valid by document order, last valid by document order, highest confidence with conflict check, append with dedupe, aggregate numeric, document synthesis required, or manual inspection required.

**Context**

Stage 4 emits schema-valid, citation-backed chunk artifacts. Stage 5 needs to produce one document-level result without turning merge into another broad extraction pass over the full document.

**Why this choice**

- Keeps merge behavior auditable and reproducible.
- Avoids unnecessary model cost and latency.
- Prevents the model from guessing critical conflict resolution.
- Makes field-level partial success and manual inspection easier to explain.

**Operational controls**

- Merge policy is versioned in the schema registry.
- Missing merge rules fail validation before customer-facing output.
- Merge decisions and conflicts are recorded in the document artifact.
- Canary tests measure document-level and critical-field F1 after merge.

## DDR-070 - Bounded Document-Level Synthesis

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Use model-based document synthesis only for fields explicitly marked as cross-chunk or synthesis-required. The synthesis prompt must contain chunk-level candidates, source-backed summaries, conflict records, and only relevant cited source spans. It must not contain the full document text.

**Context**

Large contracts and reports can exceed the Llama 3.1 128K context window, and a 1,000-page document cannot be safely summarized by placing the whole source into one prompt.

**Why this choice**

- Preserves the hierarchical context-window strategy.
- Keeps synthesis cost and latency bounded.
- Supports cross-page reasoning without losing provenance.
- Prevents silent truncation of source evidence.

**Operational controls**

- Target synthesis input `<= 20K` tokens; hard cap `30K`.
- Maximum `10` synthesis questions per document in v1.
- Use Bedrock Converse tool use with `submit_document_synthesis`.
- Reject uncited synthesized values as `not_found`.
- Include document-level summaries in v1 only when projected synthesis cost stays within the Stage 5 cost budget.
- If summaries breach the hard stop-loss, mark summary fields `not_found` with `rejection_reason="COST_POLICY_DEFERRED"` and defer summary-heavy workflows to v2/self-hosted inference.

## DDR-071 - Field-Coverage Based Partial Success

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Determine document outcome from field coverage, field criticality, page criticality, and unresolved conflicts rather than chunk success count. Default v1 requires `100%` critical-field coverage, `100%` critical-page coverage, required-field coverage of at least `95%`, and zero unresolved critical conflicts. Degraded outputs are not customer-visible in v1; they must be manually evaluated before webhook delivery.

**Context**

A document can have many successful chunks and still be unusable if a critical signature, invoice total, or required clause failed. Conversely, a document can miss non-critical optional content and still provide useful extracted data.

**Why this choice**

- Matches the case study requirement that partial success depends on business importance.
- Avoids false success when critical fields fail.
- Allows useful degraded results when only non-critical fields are missing, while keeping customer delivery gated by review.
- Gives manual review a precise reason code.

**Operational controls**

- Coverage metrics are stored in `document-extraction.json`.
- Critical/required/optional field sets come from the schema registry.
- Degraded outputs include missing-field and conflict reason codes, then route to manual evaluation before delivery.
- Tenant-specific thresholds are deferred to v2 because v1 has only base-level tenants.

## DDR-072 - Citation-Gated Final Document Artifact

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Require every non-null final document field to carry valid citations back to Stage 4 candidates and Stage 3 page/block/source spans. Final merged or synthesized values without valid citations are rejected and marked `not_found`.

**Context**

Stage 4 already requires citations for chunk-level fields. Stage 5 can still introduce risk during dedupe, conflict resolution, and synthesis unless the final document artifact repeats the citation requirement.

**Why this choice**

- Preserves end-to-end provenance.
- Prevents synthesized hallucinations from becoming final fields.
- Makes customer review and audit defensible.
- Keeps Stage 6 validation grounded in source evidence.

**Operational controls**

- Citation resolver validates final citations against source chunk artifact hashes.
- Merge output stores `source_candidates` and `source_citations`.
- Critical uncited values route to manual inspection.
- Immutable audit records store only hashes and counts, not PII values.

## DDR-073 - Stage 5 Idempotency and Replay Boundary

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Use `tenant_id + job_id + schema_version + merge_policy_version + ordered_chunk_artifact_hashes` as the Stage 5 idempotency key. Reuse a completed merge artifact for the same key. Treat a changed schema, merge policy, or chunk artifact hash set as a new merge attempt linked to the previous attempt.

**Context**

Merge can be replayed after transient failures, schema changes, or operator review. Replays must not duplicate billing or overwrite audit history.

**Why this choice**

- Makes merge replay deterministic.
- Prevents accidental reuse after upstream artifacts change.
- Supports validation-only and synthesis-only replay.
- Keeps audit evidence append-only.

**Operational controls**

- Store ordered chunk artifact hashes in the merge attempt record.
- Platform retries with identical inputs are non-billable.
- Customer-requested or schema-changing replays are billable according to the usage policy.
- Prior immutable audit events are never mutated.
- Validate final citations against the current Stage 4 chunk artifacts referenced by hash.
- Re-merge and re-validate citations when manual review or Stage 4 replay replaces a chunk artifact.

## DDR-074 - Manual Verification for Invoice Total Conflicts

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Route invoice total conflicts to manual verification in v1. Deterministic arithmetic validation can identify the conflict and provide evidence, but it must not automatically select the final total when header total, subtotal, tax, line-item total, or payable amount disagree.

**Context**

Invoice totals are critical business fields. Even when arithmetic validation suggests one value is more likely, choosing incorrectly can create downstream financial or compliance errors.

**Why this choice**

- Avoids silently accepting high-impact financial mistakes.
- Makes total conflicts explainable to reviewers.
- Keeps v1 conservative while labeled review data is still limited.
- Preserves customer trust for critical monetary fields.

**Operational controls**

- Total conflict records include candidate values, normalized amounts, arithmetic checks, citations, and confidence.
- Conflict audit events contain only IDs, hashes, counts, and non-PII reason codes.
- Manual review result is stored as a separate reviewed correction artifact.
- V2 can revisit auto-resolution only after measured precision on reviewed invoices is acceptable.

## DDR-075 - Cost-Gated Document Summaries

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Include document-level summaries for contracts and reports in v1 only when the schema declares `summary_required: true`, the projected synthesis cost fits within the Stage 5 budget, and the summary does not materially threaten the representative `$0.10/100-page` pipeline target. If summary generation would breach the hard stop-loss, skip the summary field in v1 and mark it as cost-deferred.

**Context**

Document summaries are useful for contracts and corporate reports, but summary-heavy workloads can consume many model tokens. The v1 system has only base-level tenants and must protect the shared cost target.

**Why this choice**

- Provides summaries when they are economically safe.
- Avoids overbuilding expensive summary behavior into the default v1 path.
- Keeps the cost story defensible for the interview case study.
- Creates a clear v2 reason for self-hosted inference.

**Operational controls**

- Preflight synthesis tokens and projected model cost before summary generation.
- Emit summaries only for schemas with `summary_required: true`.
- Skip summaries when projected Stage 5 cost exceeds the hard stop-loss.
- Use `rejection_reason="COST_POLICY_DEFERRED"` for skipped summary fields.
- V2 evaluates self-hosted Llama on EKS/vLLM or TGI for high-volume summary workloads.

## DDR-076 - Separate Degraded Review Queue

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Route v1 degraded outputs to a separate degraded-review queue before customer delivery. Do not send degraded outputs directly through the webhook path. Keep this queue separate from the manual inspection queue used for critical failures, unsupported critical pages, invoice total conflicts, and unresolved critical fields.

**Context**

Degraded outputs can still be useful, but they represent lower-severity quality gaps than critical extraction failures. Mixing them with critical manual inspection would make triage noisier and make operational SLAs harder to reason about.

**Why this choice**

- Keeps critical manual inspection focused on high-risk failures.
- Lets reviewers handle degraded outputs with a lighter workflow.
- Preserves customer quality by gating degraded delivery.
- Makes operational metrics clearer.

**Operational controls**

- `MERGE_DEGRADED` routes to the degraded-review SQS queue.
- Critical states route to the manual inspection SQS queue.
- Review outcome is recorded as a separate reviewed correction or approval artifact.
- Metrics split degraded review backlog from critical inspection backlog.

## DDR-077 - Minimal v1 Document Summary Schema

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Use a minimal `document_summary` schema in v1 when summaries fit within the cost budget. The schema includes status, short summary text, key points, risks or open items, citations, confidence, and rejection reason. Domain experts must validate the schema before production launch.

**Context**

The platform should support document-level summaries for contracts and reports when affordable, but the exact summary shape should not be over-specified before domain review.

**Why this choice**

- Gives the design a concrete artifact contract.
- Keeps v1 summary output simple and reviewable.
- Preserves citation and confidence requirements.
- Leaves room for domain-specific refinement.

**Operational controls**

- Summary text should be short, recommended `<= 150` words.
- Every key point and risk/open item must have source citations.
- Only emit `document_summary` when the schema declares `summary_required: true`.
- Contract summaries initially cover parties, effective date, term/renewal, obligations, governing law, and signatures when present.
- Corporate report summaries initially cover reporting period, entities, key metrics, risks, and notable changes when present.
- Cost-deferred summaries use `status="cost_deferred"` and `rejection_reason="COST_POLICY_DEFERRED"`.

## DDR-078 - Manual Review Re-Merge Boundary

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Start Stage 5 when every required chunk has reached a terminal-or-resumable Stage 4 state. Treat `EXTRACTION_MANUAL_INSPECTION_REQUIRED` as resumable. Stage 5 produces an interim merge artifact and leaves the job in `MERGE_MANUAL_INSPECTION_REQUIRED` until `manual_review.completed` triggers a re-merge against the updated chunk artifact set.

**Context**

Waiting indefinitely for manual review prevents the system from producing a useful internal view of completed fields. Proceeding to delivery would be unsafe when critical fields still require review.

**Why this choice**

- Keeps the pipeline resumable and observable.
- Preserves useful partial merge artifacts for reviewers.
- Prevents unsafe customer delivery before review completion.
- Makes manual review updates deterministic through re-merge.

**Operational controls**

- `manual_review.completed` is emitted through EventBridge with tenant/job/chunk IDs, reviewed artifact hash, review outcome, and non-PII reason codes.
- Re-merge uses a new merge attempt ID and ordered chunk artifact hashes.
- Prior merge artifacts and audit events remain immutable.
- Re-merge can move the job to `MERGE_COMPLETED`, `MERGE_DEGRADED`, `MERGE_DEGRADED_APPROVED`, `MERGE_MANUAL_REVIEW_APPROVED`, or keep it in manual review.
- Stage 6 consumes only `MERGE_COMPLETED`, `MERGE_DEGRADED_APPROVED`, or `MERGE_MANUAL_REVIEW_APPROVED`.

## DDR-079 - Stage 5 PII-Free Audit Event Set

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Emit an explicit Stage 5 audit event set: `MERGE_STARTED`, `MERGE_COMPLETED`, `SYNTHESIS_REQUESTED`, `SYNTHESIS_COMPLETED`, `MERGE_DEGRADED`, `MERGE_MANUAL_INSPECTION_REQUIRED`, `MERGE_POISON`, `REMERGE_STARTED`, `REMERGE_COMPLETED`, and `REMERGE_MANUAL_INSPECTION_REQUIRED`.

**Context**

Earlier stages define explicit operational events. Stage 5 must provide the same auditability for merge, synthesis, manual-review handoff, re-merge, and poison handling.

**Why this choice**

- Gives SOC 2 evidence for merge and review transitions.
- Makes re-merge behavior traceable.
- Separates synthesis activity from deterministic merge in audit trails.
- Keeps incident investigation possible without logging PII.

**Operational controls**

- Events carry tenant/job IDs, merge attempt ID, artifact hashes, counts, timings, policy versions, status, and reason codes.
- Events must not include field values, prompt text, OCR text, summaries, filenames, or raw customer text.
- Events go through the immutable audit path defined in Stage 1.
- Re-merge events link to the prior merge attempt ID.

## DDR-080 - Stage 5 Synthesis Question Cap

**Status:** Accepted

**Stage:** Stage 5 - Merge and Document-Level Synthesis

**Decision**

Limit v1 document synthesis to at most `10` synthesis questions per document. Process questions by schema criticality and priority. Route overflow questions to manual inspection or degraded review with `rejection_reason="SYNTHESIS_QUESTION_LIMIT_EXCEEDED"`.

**Context**

Unbounded synthesis questions can turn Stage 5 into a cost and latency hotspot, especially for large contracts and reports.

**Why this choice**

- Bounds model calls, token usage, and tail latency.
- Keeps synthesis reviewable.
- Fits the minimal v1 summary schema plus a few cross-chunk fields.
- Gives a clear fallback for overflow questions.

**Operational controls**

- Schema declares synthesis question priority.
- Stage 5 preflights question count before model calls.
- Overflow questions are not silently dropped.
- Track overflow count as an observability metric and evaluation signal.

## DDR-081 - Deterministic Post-Processing Gate

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Make Stage 6 a deterministic final validation, normalization, redaction, and packaging stage. It does not broadly re-run OCR or extraction models. When model recovery is needed, Stage 6 emits a bounded replay request to Stage 4 or Stage 5 rather than invoking the model directly.

**Context**

Stage 6 is the final quality gate before storage and webhook notification. Letting it become another extraction stage would blur ownership, increase cost, and weaken auditability.

**Why this choice**

- Keeps the final output reproducible and explainable.
- Prevents unbounded model cost at the end of the pipeline.
- Makes validator failures easier to test and canary.
- Preserves clean stage ownership.

**Operational controls**

- Stage 6 starts only from deliverable Stage 5 states.
- Stronger-model recovery is requested through replay events.
- Each job allows at most one Stage 6-initiated reprocess request per target stage; subsequent same-class failures route to manual inspection.
- Validator and normalizer versions are recorded in final artifacts.
- Model output cannot control validation, billing, retention, or webhook behavior.

## DDR-082 - Canonical and Raw Field Values

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Store canonical normalized values for customer workflows while preserving raw source values when useful. For example, dates normalize to ISO-8601, currencies use ISO-4217 plus decimal strings, and identifiers use format-preserving normalization.

**Context**

Customers need machine-friendly values, but auditability and review often require the original source expression.

**Why this choice**

- Makes downstream API use easier.
- Preserves explainability to the original document.
- Avoids destructive normalization.
- Supports locale-sensitive review.

**Operational controls**

- Ambiguous normalization does not guess.
- Raw values remain cited to source spans.
- Canonical values include type and validator status.
- Money values use decimal strings, never binary floating point.

## DDR-083 - Deterministic Business Validators

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Run deterministic validators after normalization and before redaction. Core v1 validators include JSON Schema validation, citation presence and hash validation, critical field coverage, invoice total reconciliation, currency consistency, date order, signature presence, summary citation validation, and redaction policy validation.

**Context**

Merged outputs can still be schema-shaped but semantically wrong. High-value fields need deterministic checks before customer delivery.

**Why this choice**

- Catches business-critical errors without extra model calls.
- Provides clear reason codes for manual review.
- Makes quality measurable and testable.
- Keeps Stage 7 from storing and notifying invalid outputs.

**Operational controls**

- Validator outcomes are `PASS`, `WARN`, `DEGRADED_REVIEW_REQUIRED`, `MANUAL_INSPECTION_REQUIRED`, `REPROCESS_REQUESTED`, or `FAIL_POISON`.
- Invoice total conflicts route to manual inspection in v1.
- Validator library version is recorded in artifacts and metrics.
- Canary releases track validator false positives and manual-review rate.

## DDR-084 - Redaction After Validation

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Apply redaction and field suppression after validation. Stage 6 writes a full access-controlled `final-result.json` and a policy-shaped `customer-result.json`. If redaction fails, do not deliver the unredacted result.

**Context**

Validators need real values to check totals, identifiers, dates, and business rules. Customer-facing outputs may still need masking or suppression based on tenant policy.

**Why this choice**

- Avoids validating masked values.
- Supports tenant-specific privacy requirements.
- Prevents accidental webhook delivery of unredacted data.
- Separates internal support artifact from customer-facing artifact.

**Operational controls**

- Redaction policy validation runs before customer delivery.
- Redaction failures route to poison or security review.
- Webhooks receive pointers only, not large payloads.
- Logs and audit events never include unmasked field values.

## DDR-085 - Stage 6 Final Artifact Set

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Produce three Stage 6 artifacts: `final-result.json`, `customer-result.json`, and `validation-report.json`. Stage 7 promotes or copies these to durable output storage with retention policy.

**Context**

The system needs separate artifacts for internal validated data, customer-facing policy-shaped output, and validation diagnostics. Combining them would complicate access control and redaction.

**Why this choice**

- Keeps customer output clean and policy-shaped.
- Gives operators a validation report without polluting customer payloads.
- Supports different access controls per artifact.
- Provides a stable Stage 7 handoff contract.

**Operational controls**

- All artifacts are encrypted with the tenant KMS key.
- `customer-result.json` contains only fields allowed by output policy.
- `validation-report.json` avoids raw values unless tenant support policy explicitly allows it.
- Stage 7 handles final retention placement and API retrieval.

## DDR-086 - Stage 6 Latency and Cost Budget

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Set Stage 6 latency budgets at P50 `<= 2s`, P95 `<= 10s`, and P99 `<= 20s`. Target post-processing compute cost `<= $0.001/100 pages`, with a hard stop-loss above `$0.005/100 pages`.

**Context**

Stage 6 sits near the end of the 5-minute P95 pipeline. It must leave time for Stage 7 storage and webhook delivery and should not become a hidden cost center.

**Why this choice**

- Keeps final validation inside the end-to-end SLA.
- Makes retry storms and oversized outputs visible.
- Preserves the overall cost target.
- Gives clear alarms for policy/configuration bugs.

**Operational controls**

- Track queue age, latency, validator counts, redaction counts, and estimated cost.
- Use Fargate worker path for unusually large outputs or arrays.
- Hard stop-loss routes to operator review.
- Alert on P95 latency and cost burn.

## DDR-087 - Redacted Customer Output by Default

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Enable the redaction engine by default for v1 customer-facing output. Stage 6 still validates against full internal values, then writes `customer-result.json` with masked or suppressed fields according to tenant output policy. Fields without an explicit `redaction_policy` or suppression rule flow through unchanged.

**Context**

Stage 6 produces the artifact customers retrieve. Since documents can contain PII and sensitive business data, the safer v1 posture is privacy-preserving by default.

**Why this choice**

- Aligns with GDPR data minimization.
- Reduces accidental exposure through customer workflows.
- Keeps the full result available under stricter access controls.
- Provides a conservative SOC 2 posture.

**Operational controls**

- Redaction runs after validation.
- `customer-result.json` records whether output redaction was applied.
- Redacted fields keep `value` as the customer-visible masked value and include `redaction.applied=true` plus the policy name.
- Fields without explicit redaction/suppression policy are not modified.
- Redaction failures block customer delivery.
- Full `final-result.json` remains tenant-KMS encrypted and access-controlled.

## DDR-088 - Operator-Only Validation Report in v1

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Keep `validation-report.json` operator/support-only in v1. Customers receive a sanitized validation summary in `customer-result.json`, not the full diagnostic report.

**Context**

Validation reports can expose operational reason codes, field IDs, citation IDs, and validator internals. That is useful for support and audit, but not necessary for normal customer consumption.

**Why this choice**

- Minimizes exposure of operational internals.
- Reduces risk of leaking sensitive derived metadata.
- Keeps customer output simpler.
- Leaves room for a governed export/support workflow later.

**Operational controls**

- `validation-report.json` is encrypted with the tenant KMS key.
- Access is limited to operator/support roles with audit logging.
- Customer-facing output includes only sanitized validation summary fields.
- V2 can add a governed customer export flow if needed.

## DDR-089 - JSON-Only Customer Output in v1

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Produce only JSON outputs in v1. Do not generate CSV, XLSX, or other derived export formats during Stage 6.

**Context**

The case study requires structured JSON output suitable for APIs and downstream workflows. CSV/XLSX exports add formatting, locale, quoting, formula-injection, and access-control concerns that are not required for v1.

**Why this choice**

- Keeps the v1 artifact contract simple.
- Avoids spreadsheet formula-injection and formatting risks.
- Reduces implementation and testing scope.
- Defers customer-specific exports to a later workflow.

**Operational controls**

- Stage 6 emits `final-result.json`, `customer-result.json`, and `validation-report.json`.
- Stage 7 APIs return JSON or signed URLs to JSON artifacts.
- Future export jobs must be separate, policy-controlled, and audited.

## DDR-090 - Review-Approved Merge States for Stage 6

**Status:** Accepted

**Stage:** Stage 5 / Stage 6 Boundary

**Decision**

Define review-approved merge states explicitly. `MERGE_DEGRADED` can transition to `MERGE_DEGRADED_APPROVED` through `degraded_review.approved`. `MERGE_MANUAL_INSPECTION_REQUIRED` can transition to `MERGE_MANUAL_REVIEW_APPROVED` after `manual_review.completed` writes reviewed artifacts, Stage 5 re-merges, and coverage/conflict checks pass. Stage 6 starts only from `MERGE_COMPLETED`, `MERGE_DEGRADED_APPROVED`, or `MERGE_MANUAL_REVIEW_APPROVED`.

**Context**

Stage 6 previously referenced approved states that Stage 5 did not define. The review workflow must be explicit for state-machine correctness and auditability.

**Why this choice**

- Removes ambiguity at the Stage 5 to Stage 6 boundary.
- Keeps unresolved degraded/manual-review states from reaching final output.
- Makes review approvals auditable.
- Gives Stage 6 a clean deliverable-state contract.

**Operational controls**

- Review service owns approval events.
- Approval events carry tenant/job IDs, review ID, reviewed artifact hash, reviewer role, outcome, and non-PII reason codes.
- Stage 5 re-validates current artifacts before setting approved states.
- Stage 6 rejects unresolved `MERGE_DEGRADED` and `MERGE_MANUAL_INSPECTION_REQUIRED`.

## DDR-091 - Stage 6 to Stage 7 Durable Handoff

**Status:** Accepted

**Stage:** Stage 6 / Stage 7 Boundary

**Decision**

Use both the parent Step Functions next-task transition and a durable EventBridge-to-SQS handoff for Stage 7. Stage 6 emits `POST_PROCESSING_COMPLETED` with artifact hashes and non-PII status; EventBridge routes it to the Stage 7 queue. Stage 7 consumes idempotently using `tenant_id + job_id + post_processing_attempt_id + final_result_hash`.

**Context**

Stage 6 previously said it handed off final artifact pointers without defining the mechanism. Stage 7 will handle storage promotion, retrieval indexing, and webhook notification, all of which need independent retries.

**Why this choice**

- Keeps the main workflow explicit through Step Functions.
- Decouples Stage 7 retries from Stage 6 workers.
- Provides a replayable durable handoff message.
- Gives webhook/storage failures their own DLQ behavior.

**Operational controls**

- Handoff message contains artifact URIs, hashes, policy versions, status, and reason codes only.
- Stage 7 queue has a DLQ.
- Stage 7 idempotency prevents duplicate storage promotion or webhook sends.
- `POST_PROCESSING_COMPLETED` is also recorded in immutable audit.

## DDR-092 - Customer Redacted Field Shape

**Status:** Accepted

**Stage:** Stage 6 - Post-Processing and Final Validation

**Decision**

Represent redacted fields in `customer-result.json` with `value` set to the customer-visible masked value, `raw_value=null`, and a `redaction` object containing `applied`, `policy`, and `original_value_available`. The unmasked value remains only in `final-result.json`.

**Context**

The document referenced `customer-result.json` but did not specify whether redacted values live in `value`, `masked_value`, or another field.

**Why this choice**

- Keeps customer integrations simple: read `value` for the visible value.
- Makes redaction explicit and auditable.
- Prevents accidental exposure of raw values in customer-facing JSON.
- Preserves full values only under stricter access controls.

**Operational controls**

- `customer-result.json` never includes unmasked `raw_value` for redacted fields.
- Redaction policy name is recorded per field.
- Full values remain in `final-result.json`, encrypted with tenant KMS and access-controlled.
- Redaction failures block delivery.

## DDR-093 - Stage 7 Durable Output Promotion

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Promote Stage 6 JSON artifacts from the intermediate bucket into a tenant-scoped S3 output bucket and write an `output-manifest.json` for the post-processing attempt. The output bucket is the durable source for retrieval and retention.

**Context**

Stage 6 writes artifacts under intermediate storage. Final customer retrieval needs a stable, retention-managed location separate from intermediate artifacts.

**Why this choice**

- Separates intermediate cleanup from durable output retention.
- Gives retrieval APIs a stable manifest contract.
- Supports tenant/job-scoped deletion and access control.
- Keeps output storage independently lifecycle-managed.

**Operational controls**

- Copy/promote only after validating Stage 6 artifact hashes.
- Use tenant KMS encryption.
- Write manifest with artifact URIs, hashes, access flags, retention policy, and webhook status.
- Treat destination hash mismatch as poison/security review.

## DDR-094 - Pointer-Only Customer Webhooks

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Send signed pointer-only webhook payloads. Webhooks include job status, schema/version, result availability, API retrieval URL, artifact hash, and usage summary. They must not include extracted field values, raw document text, PII, validation-report details, or presigned S3 URLs.

**Context**

The case study requires webhook notification without pushing large payloads directly to customer endpoints. Webhooks are also a data-exposure boundary.

**Why this choice**

- Avoids leaking PII into customer webhook infrastructure by default.
- Keeps delivery payloads small and retryable.
- Encourages customers to retrieve results through authenticated APIs.
- Makes webhook failure independent of result availability.

**Operational controls**

- Use HMAC-SHA256 signatures with per-webhook secrets.
- Include event ID, timestamp, and signature headers.
- Store delivery attempts without payload PII.
- Webhook failure changes delivery state only, not completed job state.

## DDR-095 - Stage 7 Retrieval Boundary

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Expose `customer-result.json` through customer APIs in v1. Do not expose `final-result.json` or `validation-report.json` through customer APIs. Always return authenticated metadata plus short-lived signed URLs for result retrieval; do not inline result JSON in v1.

**Context**

Stage 6 produces separate full, customer-facing, and diagnostic artifacts. Stage 7 must enforce that separation at retrieval time.

**Why this choice**

- Keeps support-only diagnostics out of customer APIs.
- Preserves redaction policy for customer-facing output.
- Keeps API response shape consistent across small and large results.
- Avoids payload size pressure on API Gateway/Lambda.
- Maintains a clear SOC 2/GDPR access boundary.

**Operational controls**

- Verify API key, signed JWT, and tenant/job ownership.
- Signed URL TTL defaults to 15 minutes.
- `GET /result` returns a signed URL, not inline JSON, even for small results.
- Record retrieval access in immutable PII-free audit.
- Operator/support access to internal artifacts uses separate audited tooling.

## DDR-096 - Stage 7 Webhook Delivery Policy

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Use bounded webhook retries: up to `8` attempts over a `24h` retry window with exponential backoff and jitter. Treat `2xx` as success, retry timeouts/5xx/429/408, and mark most other 4xx responses as failed. Webhook retry exhaustion does not change the terminal job result.

**Context**

Webhook endpoints are outside platform control. The platform needs reliable delivery without infinite retries or customer-visible result rollback.

**Why this choice**

- Bounds operational cost and queue growth.
- Handles normal customer endpoint outages.
- Keeps result availability independent of webhook delivery.
- Provides clear support behavior for delivery failures.

**Operational controls**

- One logical event ID per terminal state and post-processing attempt.
- Delivery attempts are idempotent by event ID and webhook ID.
- Webhook DLQ captures retry exhaustion.
- Metrics split result completion from webhook delivery health.

## DDR-097 - Retention and GDPR Erasure Mapping

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Apply default `365-day` retention to original inputs and final output artifacts unless tenant policy says otherwise. Move original input PDFs to cold storage after `7 days`. Keep intermediate artifacts short-lived according to earlier stage policies. Maintain job-to-object mappings for GDPR access and erasure, and keep immutable audit records PII-free.

**Context**

The platform must support customer retrieval, replay where allowed, SOC 2 evidence, and GDPR deletion. Those goals require explicit object mapping and separate retention classes.

**Why this choice**

- Supports customer access and deletion workflows.
- Keeps audit records compatible with GDPR because they contain no PII.
- Reduces privacy exposure from intermediates.
- Makes storage cost visible over retention lifetime.

**Operational controls**

- Output manifest records retention policy and expiration.
- Original input lifecycle transitions to cold storage after 7 days.
- Deletion workflow removes input, intermediate, output, and retrieval index records.
- Legal hold blocks deletion with a non-PII reason code.
- Tenant-key crypto-shredding follows retention/legal-hold checks.

## DDR-098 - Stage 7 Idempotency Boundary

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Use `tenant_id + job_id + post_processing_attempt_id + final_result_hash` as the Stage 7 idempotency key. Artifact promotion, manifest writes, terminal job update, retrieval indexing, and webhook event creation must all be safe to retry under this key.

**Context**

Stage 7 receives both a Step Functions transition and an EventBridge/SQS handoff. It must tolerate duplicates without duplicate webhook events or inconsistent terminal state.

**Why this choice**

- Handles duplicate queue messages and worker retries.
- Prevents duplicate logical webhook events.
- Makes terminal state updates deterministic.
- Supports full Stage 7 replay from Stage 6 handoff.

**Operational controls**

- Conditional DynamoDB writes for terminal state and idempotency records.
- Skip S3 copy when destination hash already matches.
- One webhook event ID per terminal state and post-processing attempt.
- Hash mismatch triggers poison/security review.

## DDR-099 - Signed URL Only Retrieval in v1

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Always return a short-lived signed URL for `customer-result.json` in v1. Do not return inline result JSON, even when the result is small.

**Context**

Inline for small results and signed URL for large results creates two client integration paths and makes payload-size behavior harder to reason about.

**Why this choice**

- Keeps API responses consistent.
- Avoids API Gateway/Lambda payload-size edge cases.
- Keeps large and small document retrieval behind the same authorization and audit path.
- Makes customer integration simpler.

**Operational controls**

- Signed URL TTL defaults to 15 minutes.
- Retrieval API responses include metadata, artifact hash, and signed URL.
- Signed URLs point only to `customer-result.json`.
- Every signed URL issuance is audited with PII-free metadata.

## DDR-100 - Webhooks for All Terminal States

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Emit webhook notifications for all customer-visible terminal states in v1, including `COMPLETED`, `COMPLETED_WITH_REVIEW`, `MANUAL_INSPECTION_REQUIRED`, `FAILED`, `CANCELLED`, `DELETION_REQUESTED`, and `DELETED`.

**Context**

Customers need to know when processing failed or requires action, not only when it succeeded.

**Why this choice**

- Reduces polling pressure.
- Gives customers timely failure/manual-review signals.
- Keeps lifecycle events visible to integrations.
- Preserves pointer-only payload safety.

**Operational controls**

- Non-success payloads include status and non-PII reason codes.
- Result pointers are included only when a customer result is available.
- One logical webhook event ID per terminal state and post-processing attempt.
- Webhook delivery failure does not alter job terminal state.

## DDR-101 - Input Cold Storage After Seven Days

**Status:** Accepted

**Stage:** Stage 7 - Storage, Retrieval, and Notification

**Decision**

Retain original input PDFs and final output artifacts for `365 days` by default, but transition original input PDFs to cold storage after `7 days`.

**Context**

Original PDFs are needed for retrieval, support, replay, and compliance during the retention window, but they are larger and accessed less frequently after initial processing completes.

**Why this choice**

- Preserves the 365-day retention requirement.
- Reduces retained-storage cost for large PDFs.
- Keeps final outputs readily available.
- Maintains replay/support ability when policy allows it.

**Operational controls**

- S3 lifecycle policy transitions original inputs after 7 days.
- Output artifacts remain in the output storage class chosen for retrieval latency.
- Retrieval/replay from cold input storage may have restore latency and must be surfaced in operations.
- GDPR erasure still deletes or crypto-shreds input and output objects regardless of storage class, unless legal hold blocks deletion.

## DDR-102 - Stage 1 Async Validation Failure Destination

**Status:** Accepted

**Stage:** Stage 1 - Ingestion

**Decision**

Configure the S3-triggered validation Lambda with an asynchronous failure destination, using EventBridge on-failure or an SQS DLQ, and define bounded exponential backoff with full jitter for retryable validation failures.

**Context**

S3 object-created events invoke validation asynchronously. If the validation Lambda times out or fails after the S3 event is accepted, the platform needs a durable failure record for replay, alerting, and SOC 2 operational evidence.

**Why this choice**

- Prevents silent loss of uploaded-object validation events.
- Satisfies the resilience requirement for DLQs on failed asynchronous work.
- Keeps retry behavior explicit and bounded.
- Preserves idempotency because validation state transitions remain conditional.

**Operational controls**

- Lambda async retry policy uses roughly 1-minute then 2-minute retries with full jitter, capped at two async attempts.
- Exhausted invocations are sent to EventBridge on-failure or an SQS DLQ.
- Failure records carry `tenant_id`, `job_id`, object key hash, failure reason, and retry count.
- DLQ depth and replay counts are monitored and alerted.
