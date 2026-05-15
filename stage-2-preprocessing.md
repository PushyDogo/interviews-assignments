# Stage 2 - Pre-processing Design

## 1. Goal

The pre-processing stage turns one accepted PDF into a set of safe, ordered, cost-aware processing units for OCR and AI extraction.

It does not perform OCR or LLM extraction itself. Its job is to inspect the document, classify pages, decide which pages need OCR or image enhancement, split the document into bounded chunks, and write a chunk manifest that the next stages can process in parallel.

Stage 2 starts only after Stage 1 has accepted the upload:

- Malware scan is clean.
- File size is within 300 MB.
- Page count is within 1,000 pages.
- Tenant/schema authorization is valid.
- Object is encrypted with the tenant KMS key.
- Tenant quota has been consumed for an accepted processing job.

## 2. Requirements Covered

Stage 2 covers these requirements from `requirements.md`:

- Split large PDFs into 20-50 page chunks.
- Keep chunks constrained by token budget and document structure.
- Detect orientation, skew, and rotation.
- Enhance image quality only for scanned or low-quality pages.
- Avoid expensive rasterization for native digital PDFs.
- Classify pages as digital-text, scanned, or hybrid.
- Route pages toward the cheapest reliable downstream path.
- Preserve document order, page numbers, and metadata needed for source citations.
- Keep latency, cost, security, observability, and replay requirements intact.

## 3. Recommended Flow

### 3.1 Start Pre-processing

The Stage 1 validation Lambda starts the parent Step Functions execution after the job reaches `ACCEPTED`.

The first workflow task is `PREPROCESSING_STARTED`. It loads:

- Job metadata from DynamoDB.
- Tenant configuration.
- Schema configuration.
- Expected document type from `schema_id`.
- Tenant KMS key reference.
- Input S3 object pointer.
- Page count from Stage 1.
- Current usage ledger.

The workflow then emits an immutable audit event:

```json
{
  "event_type": "PREPROCESSING_STARTED",
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "page_count": 742,
  "timestamp": "2026-05-15T10:00:00Z"
}
```

Audit events remain PII-free: no filename, document text, or extracted content.

### 3.2 Document Profiling

The profiler inspects the PDF without converting every page to an image.

Use a small adapter layer over the chosen PDF library (PyMuPDF or pdfplumber/Poppler) so the implementation can be swapped without changing the pipeline contract. Library selection and licensing must be confirmed before build.

It captures:

- Page count from Stage 1.
- Per-page dimensions.
- Whether each page has a digital text layer.
- Text density per page.
- Image coverage per page.
- Basic layout hints such as tables, forms, headers, and repeated page furniture when cheaply detectable.
- Rotation metadata.
- Whether the page appears scanned, digital, or hybrid.

The profiler writes a compact profile artifact:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/preprocess/document-profile.json
```

This artifact is encrypted with the tenant KMS key and retained under the intermediate-artifact lifecycle policy.

### 3.3 Page Classification

Each page receives one primary route:

| Route | Meaning | Downstream implication |
|---|---|---|
| `DIRECT_TEXT` | Page has reliable embedded text and low image dependence | Bypass OCR; send normalized text/layout hints downstream |
| `FULL_OCR` | Page is image-only or scanned | OCR required |
| `HYBRID` | Page has text plus important image/table/form regions | Selective OCR or layout extraction required |
| `LOW_QUALITY_SCAN` | Page is scanned and likely needs enhancement | Enhance before OCR |
| `UNSUPPORTED` | Page is unreadable, corrupt, or structurally unsafe | Route to failed/inspection path |

The classifier should be deterministic first:

- Embedded text character count.
- Text density by page area.
- Dictionary-word ratio.
- Language detection confidence.
- Garbage-text ratio, including replacement characters, control characters, and excessive single-character tokens.
- Optional spot comparison between a small rendered sample and the embedded text signal for suspicious pages.
- Image area ratio.
- Font/object metadata.
- Rotation/skew hints.
- Page renderability at low resolution.

Embedded text is not automatically trusted. A page can have an embedded text layer that is garbage due to bad OCR, broken encoding, invisible text overlays, or copy-protection artifacts.

`DIRECT_TEXT` requires:

- Sufficient character density.
- Dictionary-word ratio above the configured threshold for the expected language.
- Language detection confidence above threshold, when language is detectable.
- Low garbage-text ratio.
- No strong layout/image signal suggesting important content exists only in the rendered page.

If text quality is suspicious, route the page to `HYBRID` or `FULL_OCR` instead of `DIRECT_TEXT`.

Ambiguous pages default to OCR in v1. This is more expensive than a perfect classifier, but it avoids silently dropping content that may contain critical fields. A lightweight classifier can be added in v2 once we have labeled production examples and can prove it reduces OCR cost without hurting recall.

Unsupported pages are kept in the manifest. They are never silently dropped.

Unsupported-page policy:

- If an unsupported page is critical, route the job or chunk to `MANUAL_INSPECTION_REQUIRED`.
- If unsupported pages are non-critical and below the configured tolerance, mark the chunk as `DEGRADED` and continue.
- If unsupported pages exceed the tolerance, route to manual inspection.
- Downstream merge and validation must see unsupported pages so field coverage can be judged by business importance, not only by chunk success count.

Default tolerance for v1:

- `0` unsupported critical pages.
- Up to `2` unsupported non-critical pages per chunk.
- Up to `1%` unsupported non-critical pages per document.

### 3.4 Orientation and Image Quality Detection

Pre-processing detects whether a page needs:

- Rotation correction.
- Deskew.
- Contrast normalization.
- Denoising.
- DPI normalization.
- Binarization.

The system should not enhance every page. Enhancement is reserved for `FULL_OCR` and `LOW_QUALITY_SCAN` pages, and for `HYBRID` regions where OCR quality would otherwise be poor.

This is a major cost and latency control. Rasterizing a 1,000-page digital PDF just because a few pages need OCR would waste CPU, storage, and time.

V1 default output: **PNG, 300 DPI, grayscale**. Use 400 DPI only for critical pages where 300 DPI is predicted to underperform. Avoid JPEG (compression artifacts hurt OCR), TIFF (complicates S3/Textract handoff), RGB (unless color carries semantic meaning), and 600 DPI by default (increases size and latency without guaranteed accuracy gain).

### 3.5 Token-Aware Chunking

The case study asks for 20-50 page chunks, but page count alone is not enough. A 50-page legal contract can exceed a downstream token budget, while 50 invoice pages may be cheap.

Chunking policy:

- Target 20-50 pages per chunk.
- Hard cap 50 pages per chunk.
- Also cap estimated text tokens per chunk.
- Prefer natural boundaries: sections, invoice boundaries, form boundaries, report headings, or page groups.
- Keep chunk order stable.
- Allow smaller chunks for dense legal/report pages.
- Avoid splitting tables or multi-page sections when detectable.

Recommended initial token budgets:

| Document type | Target pages/chunk | Estimated token cap/chunk | Notes |
|---|---:|---:|---|
| Invoice batch | 25-50 | 12K-20K | Usually repetitive, shorter fields |
| Forms | 20-40 | 12K-20K | Layout matters more than text volume |
| Legal contract | 10-20 | 20K-25K | Section boundaries matter; keep room for citations and synthesis |
| Corporate report | 10-20 | 20K-25K | Tables and cross-page context matter |
| Unknown | 20 | 15K | Conservative default |

Recommended v1 approach for contracts and reports: use a fixed **25K estimated-token soft cap** and a fixed **30K hard cap** per chunk. If a natural section boundary would slightly exceed the soft cap, allow it up to the hard cap. If it exceeds the hard cap, split the section and preserve cross-chunk references in the manifest. Tenant-configurable higher caps are deferred to v2/premium tiers.

This keeps chunks far below the model's maximum context window. The remaining context budget is needed for the extraction schema, tool definitions, instructions, examples, citations, validation hints, and merge/synthesis overhead. These are pre-processing estimates, not final LLM prompts. Later stages still compress, cite, and validate.

Token estimation method:

- For pages with embedded text, estimate from normalized extracted text.
- For OCR-bound pages, estimate from page profile: expected words per page from layout density, page type, and historical tenant/schema averages.
- Use a conservative fallback of `1 token ~= 4 characters` for English-heavy documents.
- Add a 20% safety margin to estimated tokens before applying chunk caps.
- Store both `estimated_tokens_raw` and `estimated_tokens_with_margin`.
- Record `token_estimation_method` and `token_estimator_version` in the manifest.

Tokenizer policy:

- V1 uses a model-agnostic conservative estimator for chunking because the same manifest should work for managed Bedrock and future self-hosted models.
- Stage 3 may recalculate exact prompt tokens using the model-specific tokenizer for the selected inference path.
- If Stage 3 finds a chunk exceeds the real model budget, it can request a manifest replay with a lower token cap or split the chunk using the manifest's page-level details.

### 3.6 Page and Chunk Manifests

Pre-processing writes two related manifests that become the contract with OCR and extraction stages:

- `page-manifest.json`: one record per page with its route, quality signals, enhancement requirements, and criticality hints.
- `chunk-manifest.json`: ordered chunk definitions that reference the page manifest and list the exact page numbers included in each chunk.

The chunk manifest must not only include aggregate counts. Stage 3 needs the exact page routing decisions to know which pages need OCR, which pages can bypass OCR, and which pages are unsupported.

### 3.6.1 Page Manifest

```json
{
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "manifest_type": "page_manifest",
  "preprocessing_algorithm_version": "preprocess-v1.0",
  "page_count": 742,
  "pages": [
    {
      "page_number": 1,
      "route": "DIRECT_TEXT",
      "criticality": "normal",
      "digital_text_available": true,
      "ocr_required": false,
      "enhancement_required": false,
      "rotation_degrees": 0,
      "skew_detected": false,
      "text_density": 0.82,
      "image_coverage": 0.05,
      "estimated_tokens_raw": 640,
      "estimated_tokens_with_margin": 768,
      "token_estimation_method": "normalized_chars_div_4_plus_20pct",
      "quality_warnings": []
    },
    {
      "page_number": 7,
      "route": "LOW_QUALITY_SCAN",
      "criticality": "critical",
      "digital_text_available": false,
      "ocr_required": true,
      "enhancement_required": true,
      "enhanced_image_uri": "s3://.../preprocess/enhanced/page-0007.png",
      "rotation_degrees": 90,
      "skew_detected": true,
      "estimated_tokens_raw": 1100,
      "estimated_tokens_with_margin": 1320,
      "token_estimation_method": "layout_density_schema_average_plus_20pct",
      "quality_warnings": ["low_contrast", "rotated"]
    }
  ]
}
```

Page manifest location:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/preprocess/page-manifest.json
```

### 3.6.2 Chunk Manifest

```json
{
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "manifest_type": "chunk_manifest",
  "preprocessing_algorithm_version": "preprocess-v1.0",
  "document_profile_uri": "s3://.../document-profile.json",
  "page_manifest_uri": "s3://.../page-manifest.json",
  "chunks": [
    {
      "chunk_id": "chunk_0001",
      "page_start": 1,
      "page_end": 25,
      "pages": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25],
      "page_routes": {
        "DIRECT_TEXT": [1, 2, 3, 5, 6, 8, 9, 10, 11, 12, 13, 14, 16, 17, 18, 19, 20, 21],
        "FULL_OCR": [4, 15],
        "HYBRID": [22, 23, 24],
        "LOW_QUALITY_SCAN": [7, 25],
        "UNSUPPORTED": []
      },
      "route_summary": {
        "direct_text_pages": 18,
        "full_ocr_pages": 4,
        "hybrid_pages": 3,
        "unsupported_pages": 0
      },
      "estimated_tokens_raw": 12340,
      "estimated_tokens_with_margin": 14808,
      "token_estimator_version": "token-estimator-v1.0",
      "ocr_required": true,
      "enhancement_required": true,
      "chunk_status": "READY",
      "critical_page_hints": ["signature", "total"],
      "unsupported_page_policy": {
        "critical_unsupported_pages": [],
        "non_critical_unsupported_pages": [],
        "action": "CONTINUE"
      }
    }
  ]
}
```

Chunk manifest location:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/preprocess/chunk-manifest.json
```

The manifest must be deterministic and replayable: rerunning pre-processing on the same input and same configuration should produce the same chunk IDs and page ranges unless the algorithm version changes.

### 3.7 Transition to OCR/Extraction

After the manifest is written:

- DynamoDB job state becomes `PREPROCESSING_COMPLETED`.
- Chunk-level records are created for each chunk.
- Usage ledger records elapsed time, Lambda/ECS/Fargate usage, S3 writes, and estimated cost.
- Audit event is emitted to immutable audit storage.
- The parent Step Functions workflow launches Stage 3 with a Step Functions **Distributed Map** over the chunk manifest.

The Distributed Map input is the ordered chunk list from `chunk-manifest.json`. Each map item includes:

- `tenant_id`
- `job_id`
- `chunk_id`
- `page_manifest_uri`
- `page numbers`
- `page_routes`
- `ocr_required`
- `enhancement_required`
- `estimated_tokens_with_margin`
- `chunk_status`

Distributed Map provides elastic chunk-level fan-out while preserving concurrency controls. Concurrency is bounded globally and per tenant so Stage 3 cannot overwhelm Textract, Bedrock, DynamoDB, or downstream queues.

## 4. AWS Components

| Component | Role |
|---|---|
| Step Functions Standard | Owns the durable pre-processing workflow |
| Lambda profiler | Fast metadata extraction, profile generation, direct-text checks for bounded documents |
| ECS/Fargate CPU worker | Required v1 path for heavier PDF rendering, image-quality detection, or enhancement |
| S3 input bucket | Source PDF, encrypted with tenant CMK |
| S3 intermediate bucket | Document profile, page manifest, chunk manifest, page artifacts, enhanced page images |
| DynamoDB | Job state, chunk records, replay markers, usage ledger |
| EventBridge | Stage events and audit fan-out |
| Firehose -> S3 Object Lock | Immutable audit records |
| CloudWatch Logs/Metrics | Operational logs and metrics |
| X-Ray | Trace workflow, worker, S3, and DynamoDB operations |
| Per-tenant KMS CMK | Encrypts intermediate artifacts |

## 5. State Model

| State | Meaning | Owner |
|---|---|---|
| `PREPROCESSING_QUEUED` | Job accepted and waiting for pre-processing capacity | Step Functions |
| `PREPROCESSING_STARTED` | Profile and classification have started | Step Functions / worker |
| `DOCUMENT_PROFILED` | PDF profile artifact written | Profiler |
| `PAGE_CLASSIFICATION_COMPLETED` | Page routes are known | Profiler |
| `IMAGE_ENHANCEMENT_COMPLETED` | Required enhancement artifacts are written | CPU worker |
| `CHUNKING_COMPLETED` | Chunk manifest written | Chunker |
| `PREPROCESSING_COMPLETED` | Job is ready for OCR/extraction | Step Functions |
| `PREPROCESSING_FAILED_RETRYABLE` | Transient failure; safe to retry | Step Functions |
| `PREPROCESSING_FAILED_FINAL` | Non-recoverable failure | Step Functions |
| `MANUAL_INSPECTION_REQUIRED` | Document is structurally unusual or unsafe to automate | Step Functions |

Chunk-level records should be created after `CHUNKING_COMPLETED`, not before. This prevents downstream stages from seeing incomplete or unstable chunk definitions.

## 6. Idempotency and Replay

Pre-processing is idempotent at the job and artifact level.

Rules:

- Use `job_id + preprocessing_algorithm_version` as the replay boundary.
- Write profile and manifest artifacts to deterministic S3 keys.
- Use conditional DynamoDB updates for state transitions.
- Do not create duplicate chunk records if the same manifest already exists.
- Store `manifest_hash` in DynamoDB.
- Store `page_manifest_hash` and `chunk_manifest_hash` in DynamoDB.
- If the algorithm changes, write a new manifest version and record the version in job metadata.

Replay modes:

| Replay mode | Scope | Use case | Tenant billable? |
|---|---|---|---|
| Full pre-processing replay | Entire job | Customer-requested reprocess with same input/config | Yes |
| Platform full replay | Entire job | Platform bug fix, failed deployment, or internal recovery | No |
| Manifest replay | Chunk records only | DynamoDB write failure after manifest was written | No |
| Enhancement replay | Selected pages | CPU worker failure on enhanced image generation | No, unless customer changed settings/input |
| Inspection replay | Operator-approved retry | Document previously routed to manual inspection | Depends: No for platform error; Yes for customer-requested retry |

Billing rule: tenants are not billed for platform-initiated replays caused by platform failures, deployment defects, infrastructure errors, or idempotent recovery from partial writes. Tenants are billed for customer-requested reprocessing, changed inputs, changed schema choices, or retries after customer-side cancellation where new processing work is intentionally requested.

## 7. Security and Compliance

Security controls from Stage 1 continue here:

- Read input only from the tenant-scoped S3 key.
- Decrypt with the tenant KMS key only in the context of the current job.
- Write all intermediate artifacts with the tenant KMS key.
- Keep audit records PII-free.
- Do not write document text, filenames, or page images to logs.
- Use structured log scrubbing and PII leakage detection.
- Use least-privilege IAM for profiler and worker roles.
- Deny cross-tenant reads through IAM, S3 prefix conditions, and application checks.

Intermediate artifact retention:

- Document profile and chunk manifest: retain with job metadata until output retention expires.
- Enhanced page images: retain for 24 hours, then delete.
- Failed-page artifacts: retain for the rejected/failed artifact window, then transition/delete according to policy.

## 8. Observability

### 8.1 Metrics

Track:

- `preprocess.jobs_started`
- `preprocess.jobs_completed`
- `preprocess.jobs_failed`
- `preprocess.duration_ms`
- `preprocess.pages_profiled`
- `preprocess.pages_direct_text`
- `preprocess.pages_full_ocr`
- `preprocess.pages_hybrid`
- `preprocess.pages_low_quality_scan`
- `preprocess.pages_enhanced`
- `preprocess.chunks_created`
- `preprocess.estimated_tokens_total`
- `preprocess.estimated_tokens_per_chunk_p95`
- `preprocess.worker_retries`
- `preprocess.manual_inspection_required_count`
- `preprocess.cost_estimated_usd`

Important dimensions:

- Environment.
- Tenant tier, not raw tenant ID for high-cardinality metrics.
- Document type.
- Worker type: Lambda or Fargate.
- Failure reason.

### 8.2 Logs

Logs should include:

- `tenant_id`
- `job_id`
- `stage`
- `state`
- `correlation_id`
- `preprocessing_algorithm_version`
- `page_count`
- `chunk_count`
- `failure_reason`

Logs must not include raw document text, page images, extracted text, filenames, or presigned URLs.

### 8.3 Tracing

Trace:

- Step Functions execution.
- Profiler worker.
- Optional image-enhancement worker.
- S3 reads/writes.
- DynamoDB state transitions.
- Usage ledger writes.

### 8.4 Alerts

Alert on:

- Pre-processing error-rate spikes.
- Pre-processing latency SLO burn.
- Fargate/Lambda timeout spikes.
- Chunk-count anomalies.
- Unexpected rise in `FULL_OCR` pages.
- Manual-inspection spikes.
- Intermediate S3 write failures.
- KMS decrypt/encrypt failures.
- PII detected in logs.

## 9. Cost and Latency Controls

Pre-processing can quietly become expensive if every page is rendered or enhanced. The main control is selective work.

Cost controls:

- Do not rasterize pages that have reliable digital text.
- Do not enhance pages unless OCR quality needs it.
- Use Lambda only for metadata-only and lightweight profiling below the Lambda cutoff.
- Use ECS/Fargate for large-document profiling, CPU-heavy image operations, or documents likely to hit Lambda limits.
- Store enhanced images only for pages that need OCR.
- Delete enhanced images quickly.
- Use page classification to reduce Textract calls in the next stage.
- Update the usage ledger with pages profiled, pages enhanced, CPU duration, S3 bytes written, and estimated downstream OCR pages avoided.

Latency controls:

- Keep document profiling sequential only where PDF libraries require it.
- Parallelize page-quality checks and enhancement by page range.
- Cap worker concurrency per tenant to preserve fairness.
- Avoid blocking the full job on non-critical enhancement failures; route affected pages to retry or inspection.

Lambda vs. Fargate cutoff:

| Condition | Worker |
|---|---|
| `page_count <= 250` and `file_size <= 100 MB` and estimated direct-text/mixed profile | Lambda profiler |
| `page_count > 250` or `file_size > 100 MB` | ECS/Fargate profiler |
| Any document requiring broad page rendering or enhancement | ECS/Fargate CPU worker |
| Retry after Lambda timeout/memory/tmp-storage pressure | ECS/Fargate CPU worker |

This avoids pretending Lambda is safe for the 1,000-page / 300 MB upper bound. Lambda remains the fast path; Fargate is the bounded path for large or heavy documents.

## 10. Approximate Cost and Latency

Pre-processing cost is dominated by image enhancement; profiling and manifest writes are negligible.

| Step | Approx cost | Notes |
|---|---|---|
| Document profiling (Lambda path, ≤250 pages / ≤100 MB) | `<$0.001` | Fast metadata-only path; no rasterization. |
| Document profiling + enhancement (Fargate path) | Workload-dependent | Billed per vCPU/memory second; variable with page count and scan quality. |
| Manifest writes, state transitions, audit events | `<$0.001` combined | S3, DynamoDB, EventBridge — negligible. |

Latency target: under 30 seconds for typical digital or mixed PDFs. A 1,000-page low-quality scan is the expected P95-tail workload — capacity-planned for, but not the design centre.

## 11. Failure Handling

| Failure | Handling |
|---|---|
| PDF library cannot parse structure | Retry with alternate parser; if still failing, `MANUAL_INSPECTION_REQUIRED` or `PREPROCESSING_FAILED_FINAL` |
| Worker timeout | Retry page range with smaller batch size or Fargate worker |
| KMS decrypt failure | Fail fast, alert, do not continue |
| S3 read/write failure | Retry with backoff; DLQ/replay if persistent |
| Enhancement fails for non-critical page | Mark page for OCR without enhancement or retry once |
| Enhancement fails for critical page | Retry; then manual inspection if still failing |
| Chunk contains unsupported non-critical pages within tolerance | Mark chunk `DEGRADED`; continue with unsupported pages visible to Stage 3 and merge |
| Chunk contains unsupported critical page | Route chunk/job to `MANUAL_INSPECTION_REQUIRED` |
| Unsupported pages exceed tolerance | Route job to `MANUAL_INSPECTION_REQUIRED` |
| Chunk manifest write succeeds but DynamoDB write fails | Replay manifest registration idempotently |
| Duplicate Step Functions retry | Reuse existing manifest if hash/version match |
| Too many unsupported pages | Route job to manual inspection or fail according to tenant policy |

