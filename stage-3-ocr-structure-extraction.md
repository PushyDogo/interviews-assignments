# Stage 3 - OCR and Structure Extraction Design

## 1. Goal

Stage 3 turns the ordered page and chunk manifests from Stage 2 into normalized, citation-ready text and layout artifacts for AI extraction.

This stage is deliberately not the main LLM extraction stage. Its job is to collect trustworthy text from the cheapest reliable source for each page:

- Use embedded text for pages Stage 2 marked as `DIRECT_TEXT`.
- Run asynchronous OCR only for pages that need OCR.
- Preserve page numbers, reading order, bounding boxes, table/layout hints, confidence scores, and source spans.
- Produce deterministic chunk artifacts that Stage 4 can feed into schema-constrained AI extraction.

The v1 path must support text, forms, and tables, but it should not treat every page as needing expensive structured OCR. The cost-controlled default is:

- Text: use embedded text or raw OCR.
- Forms and tables: use schema-guided extraction from raw OCR/layout first, and selectively use structured Textract APIs only for pages or regions where the schema requires stronger structure signals.

This keeps the requirement coverage intact without silently sending every page through high-cost OCR APIs.

## 2. Requirements Covered

Stage 3 covers these requirements from `requirements.md`:

- Execute OCR asynchronously for scanned or hybrid pages.
- Support text, forms, and tables extraction where required by document type or schema policy.
- Prefer raw OCR plus downstream model extraction for the cost-controlled default path.
- Treat advanced structured OCR APIs as selective or premium paths.
- Preserve page numbers, reading order, layout hints, table boundaries, and source spans.
- Process chunks in parallel while protecting OCR service quotas and tenant fairness.
- Maintain chunk-level state, retries, DLQs, replay, and poison-pill handling.
- Track OCR page count, OCR latency, OCR error rate, OCR throttling, stage cost, and queue age.

## 3. Inputs from Stage 2

Stage 3 starts from the parent Step Functions workflow using a Distributed Map over `chunk-manifest.json`.

Required inputs per chunk:

- `tenant_id`
- `job_id`
- `chunk_id`
- `schema_id`
- `page_manifest_uri`
- Ordered `pages`
- Per-page routes: `DIRECT_TEXT`, `FULL_OCR`, `HYBRID`, `LOW_QUALITY_SCAN`, or `UNSUPPORTED`
- Enhanced image URIs where Stage 2 created them
- `estimated_tokens_with_margin`
- `chunk_status`, including `READY` or `DEGRADED`
- Critical page hints and unsupported-page policy
- Tenant KMS key reference
- Stage 1/2 usage ledger pointer

Stage 3 must not infer page routing from aggregate counts. The per-page manifest is the contract.

## 4. AWS Components

| Component | Role |
|---|---|
| Step Functions Standard | Parent orchestration and child chunk workflows |
| Step Functions Distributed Map | Bounded fan-out over ordered chunk definitions |
| DynamoDB On-Demand | Chunk state, OCR idempotency records, callback state, retry counters, and usage ledger updates |
| S3 intermediate bucket | Stores temporary OCR input PDFs, Textract output, normalized OCR artifacts, and mapping files |
| Amazon Textract async APIs | Raw text OCR and selective structured Forms/Tables/Queries extraction |
| SNS Textract completion topic | Receives Textract async completion notifications |
| SQS callback queue | Buffers OCR completion messages and supports DLQ redrive |
| Callback Lambda | Verifies SNS message integrity and records completion state |
| Normalization worker Lambda / Fargate task | Retrieves Textract output and writes normalized `ocr-structure.json` |
| ECS/Fargate PDF worker | Builds selected-page OCR input PDFs and region-crop inputs when needed |
| Tenant KMS CMKs | Encrypts OCR input artifacts, Textract output, and normalized OCR artifacts |
| EventBridge | Emits stage events and audit events |
| Firehose audit stream | Delivers immutable audit events to the audit bucket |
| CloudWatch Logs/Metrics | Captures structured logs, EMF metrics, alarms, and queue health |
| X-Ray | Traces orchestration, callbacks, OCR workers, and normalization |
| DLQs | Isolate failed callback, OCR, normalization, and poison chunks |

## 5. Recommended Flow

### 5.1 Start Chunk OCR Work

The parent Step Functions workflow enters a Distributed Map over the ordered chunk list.

For each chunk, the child workflow:

1. Creates or loads a chunk state record from DynamoDB.
2. Validates that the chunk belongs to the tenant and job.
3. Loads the page manifest.
4. Splits pages into direct-text, OCR, hybrid, and unsupported groups.
5. Applies OCR admission control before starting any paid OCR job.
6. Writes `OCR_CHUNK_STARTED` audit and operational events.

Chunk state is separate from job state. A 1,000-page document can have many chunks in different states at the same time.

Recommended chunk states:

| State | Meaning |
|---|---|
| `OCR_QUEUED` | Chunk is ready for Stage 3 but not yet running |
| `OCR_RUNNING` | OCR/direct-text normalization is active |
| `OCR_CALLBACK_WAITING` | Async OCR job has started and the workflow is waiting for completion |
| `OCR_CALLBACK_RECEIVED` | OCR completion notification was received and validated |
| `OCR_NORMALIZING` | Raw OCR/direct text is being normalized into the Stage 3 artifact |
| `OCR_COMPLETED` | Chunk artifact is ready for Stage 4 |
| `OCR_DEGRADED` | Chunk completed but has tolerated non-critical unsupported or low-confidence pages |
| `OCR_MANUAL_INSPECTION_REQUIRED` | Critical page or OCR result cannot be trusted |
| `OCR_FAILED_RETRYABLE` | Temporary failure; retry allowed |
| `OCR_POISON` | Retry budget exceeded or deterministic failure detected |

### 5.2 Direct Text Path

Pages marked `DIRECT_TEXT` should bypass Textract.

The worker loads the Stage 2 profile/text extraction artifact, normalizes the text, and writes page-level text blocks with approximate layout hints. If Stage 2 did not materialize normalized text for direct-text pages, Stage 3 extracts it through the same PDF adapter layer used in Stage 2 and records the parser version.

Direct-text normalization includes:

- Remove invisible/control characters.
- Preserve paragraph and line breaks when reliable.
- Preserve page number and block ordering.
- Keep table/layout hints from Stage 2 when available.
- Assign synthetic source spans using page number, block index, and character offsets.
- Store confidence as `source_confidence`, not OCR confidence.

This path is the main cost control for digital PDFs. It avoids paying OCR fees for pages that already have reliable embedded text.

### 5.3 Text, Forms, and Tables Extraction Policy

Pages marked `FULL_OCR`, `LOW_QUALITY_SCAN`, or OCR-required `HYBRID` regions use asynchronous OCR.

Default v1 service:

| Need | Service/API | Policy |
|---|---|---|
| Text extraction | Embedded PDF text or Amazon Textract async `StartDocumentTextDetection` / `GetDocumentTextDetection` | Default path |
| Table extraction | Raw OCR/layout plus Stage 4 schema extraction by default; Textract async `StartDocumentAnalysis` with `TABLES` only for selected pages/regions | Supported in v1, but selective because of cost |
| Form/key-value extraction | Raw OCR/layout plus Stage 4 schema extraction by default; Textract async `StartDocumentAnalysis` with `FORMS` only for selected pages/regions | Supported in v1, but selective because of cost |
| Queries/signatures/expense-specific extraction | Textract `StartDocumentAnalysis` or `AnalyzeExpense` | Explicit schema/tenant policy only |

This means Stage 3 supports text, forms, and tables, but the default path remains cost-aware. Per `aws-pricing.md`, Textract `DetectDocumentText` is `$0.0015/page` for the first 1M pages/month, while Forms + Tables can reach `$0.065/page`. A 100-page document with every page sent to Forms + Tables would spend about `$6.50` on Textract alone, far above the `$0.10/100-page` target.

Structured OCR is selected by policy, not by developer guesswork. The policy can choose:

- Raw text only.
- Raw text plus table extraction on selected table pages.
- Raw text plus form extraction on selected form pages.
- Forms + Tables on selected critical pages.
- Schema-specific expense extraction when the tenant explicitly pays for that path.

Stage 3 uses asynchronous APIs for operational reasons:

- Long-running OCR does not hold Lambda execution open.
- Callbacks can be retried and validated.
- Step Functions can wait durably.
- Service throttling can be handled with backoff and queue shaping.

### 5.4 OCR Job Granularity

Do not start one Textract job per page unless the page-level artifact requires it. Per-page jobs create too much orchestration overhead and callback volume.

Recommended v1 granularity:

- Submit one OCR job per contiguous page range with the same source artifact and OCR policy.
- Prefer chunk-level OCR jobs when most pages in the chunk need OCR.
- Use smaller page-range jobs when a chunk has only a few OCR pages.
- Keep direct-text pages out of OCR jobs.
- Preserve the original page numbers in the Stage 3 artifact even if OCR runs on a temporary selected-page PDF or single-page image.

Textract async input mechanics are concrete:

- For multi-page OCR, create a new selected-page PDF that contains only the pages that should be OCRed for that OCR unit.
- For a single region or page crop, create a single-page image when that is cheaper or more accurate than adding it to a PDF.
- Do not rely on "image bundles" as a Textract async input format.
- Do not send the original 1,000-page PDF to Textract just to process a few OCR pages.

When the worker creates a temporary per-chunk or per-range OCR input artifact, it must record:

```json
{
  "ocr_input_uri": "s3://document-ai-intermediate/tenant=tenant_123/jobs/job_01/chunks/chunk_003/ocr-input.pdf",
  "source_pages": [51, 52, 53, 58],
  "ocr_input_page_to_source_page": {
    "1": 51,
    "2": 52,
    "3": 53,
    "4": 58
  }
}
```

Temporary OCR input artifacts are encrypted with the tenant KMS key and follow the same short retention policy as enhanced images unless needed for manual inspection.

Every Textract async call must pass `OutputConfig`:

```json
{
  "OutputConfig": {
    "S3Bucket": "document-ai-intermediate",
    "S3Prefix": "tenant=tenant_123/jobs/job_01/stage3/textract-output/chunk_003/ocr_unit_001/",
    "KMSKeyId": "arn:aws:kms:us-east-1:123456789012:key/tenant-123-cmk"
  }
}
```

This keeps Textract raw output under the tenant-scoped S3 prefix and encrypted with the tenant CMK. Without this, Textract may write service-managed output outside the tenant CMK isolation model, which would contradict the Stage 1 security decision.

### 5.5 Hybrid Pages and Region-Level OCR

`HYBRID` pages contain useful embedded text plus rendered content that may contain signatures, stamps, scanned tables, handwritten notes, or image-only fields.

V1 should prefer region-level OCR when Stage 2 provides trustworthy coordinates. This minimizes paid OCR input and reduces duplicate text that later stages must reconcile.

Initial v1 confidence threshold: Stage 3 trusts Stage 2 region coordinates when `region_confidence >= 0.80`. If Stage 2 provides separate signals, require `bbox_confidence >= 0.85` and `content_classification_confidence >= 0.75`. These are starting estimates and should be tuned from labeled production examples.

Recommended policy:

- Keep the reliable embedded text.
- Use region-level OCR for image-only or low-confidence regions when region coordinates are available and the number of regions is small.
- Use full-page OCR only when region coordinates are missing, unreliable, or when many small regions would cost more and be harder to merge than one page-level OCR call.
- For table/form-heavy pages, use structured Textract only when schema policy requires it and only for selected pages/regions.
- Merge embedded text and OCR blocks by page order and bounding box.
- Deduplicate overlapping text where the OCR repeats embedded text.

The merge must keep provenance. A block can come from `embedded_text`, `textract_raw_ocr`, or `merged_hybrid`.

Region-level OCR has a cost guardrail: if region cropping would create more billable OCR units than full-page OCR for the same page, use full-page OCR unless the region is critical and full-page OCR is known to underperform.

### 5.6 Structured OCR Policy

Some document types or tenants require forms, tables, queries, signatures, or expense-specific extraction before Stage 4. This is supported in v1, but it must be controlled by schema and tenant policy rather than ad hoc code paths.

Recommended policy object:

```json
{
  "ocr_policy_version": "2026-05-15.v1",
  "default_api": "TEXTRACT_DETECT_DOCUMENT_TEXT",
  "structured_ocr_enabled": true,
  "structured_ocr_apis": ["TABLES", "FORMS"],
  "structured_page_selectors": ["schema.table_pages", "schema.form_pages", "critical_pages_only"],
  "max_structured_pages_per_job": 10,
  "max_queued_or_active_structured_pages_per_tenant": 50,
  "max_region_crops_per_page": 3,
  "requires_customer_approval_above_usd": 0.10
}
```

Strict cost-controlled policy example:

```json
{
  "ocr_policy_version": "2026-05-15.v1",
  "default_api": "TEXTRACT_DETECT_DOCUMENT_TEXT",
  "structured_ocr_enabled": false,
  "structured_ocr_apis": [],
  "structured_page_selectors": [],
  "max_structured_pages_per_job": 0,
  "max_queued_or_active_structured_pages_per_tenant": 50,
  "max_region_crops_per_page": 0,
  "requires_customer_approval_above_usd": 0.10
}
```

Every structured OCR decision must be written to the usage ledger and audit trail with the reason. If estimated structured OCR cost would exceed the tenant or job ceiling, the chunk is paused for explicit approval or routed to manual inspection based on tenant policy.

Structured page selectors are declarative expressions evaluated against the page manifest at chunk start. V1 can use JSONPath or CEL-style expressions such as `schema.table_pages`, `schema.form_pages`, and `critical_pages_only`; schemas declare these selectors in the schema definition document, versioned alongside `ocr_policy_version`.

V1 uses a fixed base-tenant structured OCR admission limit of `50` queued-or-active structured OCR pages. If accepting a new job or starting a Stage 3 chunk would push the tenant above that limit, the platform should apply backpressure:

- At job creation or acceptance time, return `429 Too Many Requests` with `Retry-After` when the projected structured OCR page count is already known.
- Inside Stage 3, keep the chunk queued and emit queue-age/backpressure metrics if the job was already accepted.
- Count region crops as page-equivalent OCR units for this limit unless the provider billing model proves a cheaper unit.
- Defer tenant-tier-specific structured OCR limits to v2.

## 6. Fan-out, Backpressure, and Tenant Fairness

Stage 3 uses nested controls:

- Stage 1 admission control limits accepted jobs by tenant.
- Stage 2 Distributed Map creates bounded chunk fan-out.
- Stage 3 OCR workers enforce OCR-specific concurrency before paid calls.
- Tenant-level token buckets prevent one tenant from consuming all Textract capacity.
- Global concurrency limits protect AWS service quotas.

Recommended controls:

| Control | Purpose |
|---|---|
| Global OCR concurrency limit | Protect Textract async job quota and downstream callback volume |
| Tenant OCR concurrency limit | Preserve fairness during burst intake |
| Per-job chunk concurrency limit | Prevent one 1,000-page job from starving many small jobs |
| Structured OCR page/region limit | Prevent cost blowups from structured OCR APIs |
| Queue-age alarm | Detect backpressure before SLO misses become widespread |
| Circuit breaker | Stop starting new OCR jobs during provider throttling or error spikes |

When capacity is unavailable, Stage 3 should not fail the chunk immediately. It should remain queued, emit queue-age metrics, and let admission control slow new accepted work if backlog threatens the 5-minute P95 target.

Recommended v1 starting limits:

| Limit | Initial value | Why |
|---|---:|---|
| Global active OCR units | `100` | Conservative launch default until AWS service quotas and observed callback latency are validated |
| Base tenant active OCR units | `5` | Prevents one tenant from dominating shared OCR capacity |
| Per-job active OCR chunks | `2` | Keeps one 1,000-page job from starving many small jobs |
| Structured OCR active units per tenant | `1` | Structured OCR is expensive and should be tightly shaped |
| Queued-or-active structured OCR pages per tenant | `50` | Applies backpressure before one tenant builds a large structured OCR backlog |

V1 has only base-level tenants, so there is no standard/enterprise split in these limits. These numbers are not hard product limits. They are safe v1 defaults. Before launch, run a load test against approved Textract service quotas, callback throughput, DynamoDB write capacity, and Step Functions concurrency. Increase global and tenant limits only when queue-age, cost, and error-rate metrics remain healthy. Tier-based OCR concurrency is a v2 concern.

## 7. Stage 3 Latency Budget

Stage 3 is the most latency-sensitive stage because it contains async OCR. To defend the 5-minute end-to-end P95 target, Stage 3 gets an explicit latency budget measured from `OCR_QUEUED` to `OCR_COMPLETED`, `OCR_DEGRADED`, or `OCR_MANUAL_INSPECTION_REQUIRED`.

Recommended v1 budget:

| Percentile | Budget | Interpretation |
|---|---:|---|
| P50 | `<= 60s` | Normal digital/hybrid chunks and small OCR ranges |
| P95 | `<= 180s` | Large but expected OCR chunks |
| P99 | `<= 240s` | P95-tail scanned/heavily degraded chunks |

This leaves roughly 90 seconds for downstream AI extraction, merge, validation, storage, and notification inside the 5-minute P95 target. If Stage 3 queue age plus projected OCR time exceeds the budget, the platform should apply admission backpressure instead of accepting more work that cannot meet the SLA.

Latency controls:

- Keep direct-text pages out of OCR.
- Prefer selected-page PDFs and region crops instead of sending whole source PDFs.
- Bound global, tenant, and per-job OCR concurrency.
- Use the 3-minute poll fallback before the Stage 3 P95 budget is exhausted.
- Mark scanned-heavy jobs as P95-tail risk when Stage 2 predicts high OCR volume.

## 8. Idempotency and Callback Handling

OCR start and completion must be retry-safe.

### 8.1 Starting OCR

Each OCR request uses an idempotency record scoped by:

```text
tenant_id + job_id + chunk_id + ocr_unit_id + ocr_policy_version + input_artifact_hash
```

If a retry sees an existing OCR job with the same key, it reuses the existing Textract job ID instead of starting a duplicate paid job.

If the same key is reused with a different input artifact hash or OCR policy, reject it as a conflict and route the chunk to operator review.

### 8.2 Completion Callback

Textract completion notifications arrive through SNS/EventBridge/SQS into a callback handler. The handler validates:

- SNS message signature is valid and chains to AWS SNS signing certificates.
- SNS topic ARN is the expected Textract completion topic.
- Message timestamp is within the allowed freshness window.
- Textract job ID exists in the chunk state record.
- Tenant, job, chunk, and OCR unit match the expected record.
- The callback has not already been processed.
- The OCR job completed successfully.

Duplicate callbacks are acknowledged without reprocessing. Missing or unknown callbacks are sent to a DLQ for investigation.

The callback handler should write only a small state transition. Heavy result retrieval and normalization should run in a worker task so callback processing stays fast.

### 8.3 Callback Timeout and Poll Fallback

Stage 3 should not wait indefinitely for a callback.

Recommended v1 timing:

| Timer | Value | Action |
|---|---:|---|
| Expected OCR callback window | `10 seconds - 2 minutes` | Normal path |
| Chunk OCR max wait before poll | `3 minutes` | Call `GetDocumentTextDetection` or `GetDocumentAnalysis` once to check status |
| Large-scan extended wait ceiling | `5 minutes` | Allowed only when the job is already marked as P95-tail risk |
| Callback freshness window | `10 minutes` | Reject stale or replayed callback messages |

If the 3-minute timer fires and polling says the Textract job is complete, continue to normalization and mark the callback as `POLL_RECOVERED`. If polling says the job is still running, keep waiting only when the job is allowed to enter the large-scan tail path. Otherwise, mark the OCR unit as `OCR_TIMEOUT_RETRYABLE` and use the retry policy without starting a duplicate paid job.

If the final wait ceiling expires, route the OCR unit to the OCR DLQ with the Textract job ID, OCR unit ID, attempt count, input artifact hash, and last known Textract status.

## 9. Output Artifact Contract

Stage 3 writes one normalized OCR/structure artifact per chunk:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage3/chunks/<chunk_id>/ocr-structure.json
```

The artifact is encrypted with the tenant KMS key and referenced from the chunk state record.

Recommended schema:

```json
{
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "chunk_id": "chunk_003",
  "schema_id": "invoice_v1",
  "stage": "ocr_structure",
  "artifact_version": "2026-05-15.v1",
  "source_page_manifest_hash": "sha256:...",
  "source_chunk_manifest_hash": "sha256:...",
  "ocr_policy_version": "2026-05-15.v1",
  "pages": [
    {
      "page_number": 51,
      "route": "FULL_OCR",
      "source": "textract_raw_ocr",
      "ocr_engine": "amazon_textract",
      "ocr_api": "StartDocumentTextDetection",
      "confidence_summary": {
        "min": 0.74,
        "avg": 0.93,
        "low_confidence_block_count": 3
      },
      "blocks": [
        {
          "block_id": "p51-b0001",
          "type": "LINE",
          "text": "Invoice Number 12345",
          "confidence": 0.98,
          "page_number": 51,
          "reading_order": 1,
          "bbox": {
            "left": 0.12,
            "top": 0.08,
            "width": 0.22,
            "height": 0.03
          },
          "source_span": {
            "page_number": 51,
            "block_id": "p51-b0001",
            "char_start": 0,
            "char_end": 20
          }
        }
      ],
      "warnings": []
    }
  ],
  "chunk_warnings": [],
  "usage": {
    "direct_text_pages": 12,
    "raw_ocr_pages": 13,
    "structured_ocr_pages": 0,
    "structured_ocr_regions": 0,
    "estimated_stage_cost_usd": 0.0195
  }
}
```

The artifact should not be logged. Logs may reference only the artifact URI, hashes, counts, timings, and non-PII error reasons.

## 10. Low Confidence and Partial Success

Stage 3 should not fail a chunk only because one non-critical block has low confidence. It should produce a usable artifact and mark uncertainty for Stage 4 and Stage 5.

Recommended default handling:

| Condition | Action |
|---|---|
| Low confidence on non-critical page | Mark page warning and continue |
| Low confidence on critical page | Apply the critical-page escalation ladder below |
| OCR failed for non-critical page within tolerance | Keep page placeholder, mark degraded, continue |
| OCR failed for critical page | Apply the critical-page escalation ladder below; route to manual inspection if recovery fails |
| Unsupported page from Stage 2 | Preserve placeholder and apply unsupported-page policy |
| Textract service error | Retry with backoff; then DLQ/poison if retry budget exhausted |

Criticality comes from the schema and Stage 2 manifest. For example, signature pages, invoice totals, contract clauses, and required form pages may be critical even if they are only a small fraction of the chunk.

Textract returns confidence values on a `0-100` scale. The normalized Stage 3 artifact stores confidence on a `0.0-1.0` scale. Thresholds below use the normalized scale.

Default v1 confidence thresholds:

| Signal | Critical content threshold | Non-critical content threshold | Action |
|---|---:|---:|---|
| Required field block confidence | `< 0.90` | `< 0.75` | Mark low confidence; critical fields enter escalation |
| Page average OCR confidence | `< 0.85` | `< 0.70` | Mark page warning; critical pages enter escalation |
| Low-confidence block fraction | `> 5%` of blocks below `0.90` | `> 15%` of blocks below `0.75` | Mark page/chunk degraded |
| Table/form structural confidence | `< 0.85` | `< 0.70` | Prefer stronger structured extraction or manual inspection |

These defaults are schema-configurable in v1. For example, invoice totals, signatures, and required legal clauses can use stricter thresholds than non-critical boilerplate. Tenant-specific threshold overrides are deferred to v2 because v1 has only base-level tenants.

Critical-page escalation ladder:

| Tier | Action | Billing policy |
|---|---|---|
| Tier 1 | Re-run OCR with different settings, such as corrected rotation, alternate enhancement, higher DPI for that page/region, or full-page OCR after region OCR underperformed | Billable when caused by document quality; non-billable for platform retry/recovery |
| Tier 2 | Route the affected chunk to the stronger extraction model or stronger OCR/analysis policy available in v1 | Billable when caused by document quality; non-billable for platform retry/recovery |
| Tier 3 | Send the affected chunk to `MANUAL_INSPECTION_REQUIRED` with page, region, confidence, and artifact pointers | Manual inspection billing follows tenant contract; no extra automated OCR cost after this point |

Other chunks should continue while the affected chunk escalates, unless the failed critical page blocks document-level interpretation for the selected schema. In v1, stronger-model escalation is the default recovery behavior for all base tenants; making this tenant-configurable is deferred to v2. The parent job should move toward `PARTIAL_PROCESSING` or `MANUAL_INSPECTION_REQUIRED`, not fail opaque.

## 11. Cost Controls

Stage 3 updates the usage ledger before and after paid OCR work.

### 11.1 Unit Economics Policy

The `$0.10/100-page` target should be treated as a representative-mix target, not a guarantee for every possible 100-page document.

V1 cost model assumption:

| Document mix | Share of pages | Expected OCR behavior |
|---|---:|---|
| Digital-text pages | `60%` | Bypass OCR and use embedded text |
| Hybrid pages | `30%` | OCR only selected regions/pages, with raw OCR first |
| Fully scanned pages | `10%` | Raw OCR required for most or all pages |

Under this assumption, the average OCR page rate can stay low enough for the overall pipeline to approach the target. A scanned-heavy workload is different. If 100 out of 100 pages require raw Textract OCR, Stage 3 alone costs about `$0.15` before LLM, orchestration, storage, and observability. The architecture must therefore make scanned-heavy documents visible and policy-controlled.

Policy:

- Report cost against the representative mix and separately report scanned-heavy exception cost.
- Estimate scanned-page ratio in Stage 2 and carry it into Stage 3 admission control.
- If projected cost per 100 pages exceeds the tenant/job ceiling, pause for approval or route to manual inspection instead of silently processing.
- Track `cost_policy_result` as `WITHIN_TARGET`, `MIX_EXCEPTION`, `CUSTOMER_APPROVAL_REQUIRED`, or `REJECTED_BY_COST_POLICY`.
- Use scanned-heavy documents as a product/pricing exception, not as evidence that the platform can always meet `$0.10/100 pages`.

Cost ledger fields:

- `ocr_pages_submitted`
- `raw_ocr_pages`
- `structured_ocr_pages`
- `structured_ocr_regions`
- `ocr_api`
- `ocr_unit_price_usd`
- `estimated_ocr_cost_usd`
- `actual_ocr_cost_usd` when known
- `billable`
- `billing_reason`
- `replay_initiator`

Default raw OCR cost formula:

```text
raw_ocr_cost = raw_ocr_pages * 0.0015
```

For example, if a 25-page chunk has 13 raw OCR pages:

```text
13 * $0.0015 = $0.0195
```

This cost is acceptable only when OCR is limited to pages that need it. The unit-economics policy above is the reason Stage 2 routing and Stage 3 stop-loss controls are required.

Stop-loss behavior:

- Estimate Stage 3 cost before starting OCR.
- If projected job cost exceeds tenant policy, pause for approval, downgrade to raw text extraction where safe, or route to manual inspection.
- Never silently switch into structured OCR or a stronger model.
- Mark platform-initiated replays as non-billable according to DDR-038.

## 12. Reliability and Replay

Retry policy:

| Failure | Retry? | Notes |
|---|---|---|
| Textract throttling | Yes | Exponential backoff with jitter; preserve idempotency key |
| Textract transient error | Yes | Bounded retries, then DLQ |
| Invalid input artifact | No | Deterministic failure; route to manual inspection or poison |
| KMS access denied | No automatic retry until config fixed | Security/configuration issue |
| Callback not received by deadline | Yes, poll status once; then retry or DLQ | Avoid duplicate paid start |
| Normalization worker failure | Yes | Reuse completed OCR job result |

Replay scopes:

- Chunk replay from Stage 3 input manifests.
- OCR unit replay from the OCR input artifact.
- Normalization-only replay from completed Textract job results.

Replay must not duplicate billing unless the replay is customer-requested with changed input, changed schema, or changed OCR policy.

## 12.1 Textract Outage and Regional Fallback

V1 uses Textract in the platform's primary AWS region. That keeps data residency, tenant KMS usage, service quotas, and operational testing simpler for the initial system. It also means Stage 3 has a single managed OCR-provider dependency in v1.

Outage policy:

| Condition | V1 action |
|---|---|
| Short Textract throttling or transient errors | Retry with exponential backoff and jitter under the retry budget |
| Sustained Textract error rate or throttling | Open Stage 3 circuit breaker; stop starting new OCR jobs; keep chunks queued |
| Provider outage threatening SLO | Apply Stage 1/3 admission backpressure and return `429` for new work requiring OCR |
| Jobs already accepted before outage | Keep queued where possible; move to `OCR_FAILED_RETRYABLE` or `MANUAL_INSPECTION_REQUIRED` when customer policy requires a terminal update |
| Outage longer than operational threshold, for example 4 hours | Execute provider-outage runbook, notify affected tenants, and pause OCR-heavy intake |

V2 fallback options:

- Cross-region Textract failover for tenants whose data-residency policy allows it.
- A secondary OCR provider behind the Stage 3 normalized artifact contract.
- Self-hosted OCR fallback on ECS/EKS for critical tenants or outage mode.

Cross-region failover is not enabled by default in v1 because it complicates per-tenant CMK isolation, data residency, quota management, callback routing, and audit evidence. The v1 design instead makes the dependency explicit, adds circuit breakers and backpressure, and keeps the Stage 3 output contract portable enough to add a fallback provider later.

## 12.2 Poison-Pill Handling

`OCR_POISON` is a terminal chunk-level isolation state for work that should not keep retrying automatically.

Poison triggers:

| Trigger | Threshold / rule |
|---|---|
| Retryable Textract throttling or transient errors | `3` paid-start attempts or `3` normalization attempts, whichever applies |
| Callback timeout | `2` timeout cycles after the Textract job ID has been recorded |
| Deterministic input failure | Immediate poison for corrupt selected-page PDF, invalid single-page image, unsupported Textract input, or unreadable OCR input artifact |
| KMS/IAM denial | Immediate poison after config validation confirms access is denied |
| Callback integrity failure | Immediate security DLQ; poison only if tied to a known OCR unit |
| Cost stop-loss breach | Do not retry; route to approval/manual inspection instead of poison |

Poison workflow:

1. Stop automatic retries for the OCR unit.
2. Mark the chunk `OCR_POISON`.
3. Write a poison record in DynamoDB with tenant, job, chunk, OCR unit, attempt count, failure reason, cost incurred, artifact pointers, and replay eligibility.
4. Send a compact message to the OCR poison DLQ.
5. Copy relevant non-PII diagnostics to:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage3/poison/<chunk_id>/<ocr_unit_id>/diagnostics.json
```

6. Emit immutable audit event `OCR_POISONED` with IDs, hashes, counts, policy versions, and non-PII reason codes.
7. Surface the chunk in the operator replay/manual-inspection tool.

Operator options:

| Option | When used | Billing |
|---|---|---|
| Replay normalization only | Textract completed but artifact write/normalization failed | Non-billable for platform recovery |
| Replay OCR with same input | Provider/transient failure and retry budget was conservative | Non-billable if platform-initiated; billable if customer-requested |
| Replay OCR with modified settings | Bad quality input may recover with different enhancement/DPI | Billable when caused by document quality or customer request |
| Send to manual inspection | Automated recovery is unlikely or cost ceiling prevents more OCR | Manual inspection billing follows tenant contract |
| Fail affected chunk/job | Deterministic unsupported input or policy rejection | No further automated cost |

Cost bound: Stage 3 should never spend more than the configured OCR retry budget for a chunk without explicit approval. The default budget is one initial paid OCR attempt plus up to two automated recovery attempts, except deterministic failures and cost stop-loss breaches, which do not get automatic paid retries.

## 12.3 OCR Artifact Retention

Normalized OCR artifacts are intermediate artifacts, not the final customer record.

Retention policy:

- Keep `ocr-structure.json` until Stage 4/5 extraction output is accepted.
- Keep it during retry, replay, manual inspection, and customer-visible correction windows.
- Delete it after final extraction acceptance and replay window closure.
- Retain only non-PII hashes, counts, policy versions, timings, and cost records in the immutable audit and job state records.

This reduces storage cost and supports GDPR data minimization. If a customer later requests reprocessing after the OCR artifact has been deleted, the platform must replay from the retained source document if it is still within the tenant's document retention window, and the replay is billable when customer-requested.

## 13. Security and Compliance

Stage 3 inherits the Stage 1 and Stage 2 security posture:

- Tenant-scoped S3 prefixes.
- Tenant KMS CMKs for input, intermediate, and output artifacts.
- IAM conditions requiring tenant/job scoped object paths.
- No raw OCR text in logs.
- Emit-time PII scrubbing for all structured logs.
- CloudWatch Logs subscription detection for leakage.
- PII-free immutable audit events.
- Object Lock only for audit records, not document content.

The GDPR/SOC 2 separation remains important:

- OCR artifacts may contain PII and must remain erasable or crypto-shreddable through the tenant KMS key.
- Immutable audit records must contain only tenant/job/stage identifiers, hashes, counts, timestamps, policy versions, and non-PII status.

## 14. Observability

Stage 3 emits metrics at job, chunk, and service levels.

Recommended metrics:

| Metric | Purpose |
|---|---|
| `ocr_pages_total` | Number of pages submitted to OCR |
| `direct_text_pages_total` | Number of pages bypassing OCR |
| `structured_ocr_pages_total` | Detect expected and unexpected structured OCR API use |
| `ocr_stage_latency_ms` | Stage latency budget tracking |
| `ocr_queue_age_ms` | Backpressure visibility |
| `ocr_callback_latency_ms` | Provider and callback health |
| `ocr_throttle_count` | Service quota pressure |
| `ocr_error_count` | Reliability signal |
| `ocr_low_confidence_pages` | Quality signal |
| `ocr_cost_estimated_usd` | Cost control |
| `ocr_manual_inspection_count` | Operational load |

High-cardinality values such as tenant ID and job ID should stay in structured logs and traces, not as unconstrained CloudWatch metric dimensions.

Traces must propagate:

- `correlation_id`
- `tenant_id`
- `job_id`
- `chunk_id`
- `ocr_unit_id`
- Step Functions execution ARN
- Textract job ID

## 15. Closed Stage 3 Decisions

The previous open Stage 3 decisions are now closed for v1:

| Decision | V1 choice | V2 refinement |
|---|---|---|
| Structured OCR page/region limit | Fixed `50` queued-or-active structured OCR pages per base tenant before backpressure/`429` | Tenant-tier-specific limits |
| Region-coordinate confidence | Trust Stage 2 regions at `region_confidence >= 0.80`, with `bbox_confidence >= 0.85` and `content_classification_confidence >= 0.75` when separate signals exist | Tune thresholds from labeled examples |
| Stronger-model escalation | Default on for all base tenants in the critical-page escalation ladder | Make recovery policy tenant-configurable |
