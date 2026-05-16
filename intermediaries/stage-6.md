# Stage 6 - Post-Processing and Final Validation Design

## 1. Goal

Stage 6 turns the Stage 5 document-level extraction artifact into the final customer-facing result package.

This stage is deterministic by default. It should not re-run OCR, broadly re-call the extraction model, or change source evidence. Its job is to:

- Validate the merged document output against the tenant schema.
- Normalize field values into customer-safe canonical formats.
- Run deterministic business-rule validators.
- Recompute final field confidence and validation status.
- Apply tenant output policies such as redaction, field suppression, and result shaping.
- Decide whether the result can proceed to storage and webhook notification.
- Produce a final JSON artifact that downstream customer workflows can consume.
- Update job status, cost, elapsed time, audit, and observability.

Stage 6 is the final quality gate before Stage 7 storage, retrieval, and notification.

## 2. Requirements Covered

Stage 6 covers these requirements from `requirements.md`:

- Validate extracted data against tenant schemas.
- Normalize dates, currencies, addresses, names, identifiers, totals, and tax fields.
- Apply deterministic validators for high-value fields.
- Assign field-level confidence scores.
- Route low-confidence or unverifiable critical fields to manual review or stronger-model reprocessing.
- Produce structured JSON output suitable for API retrieval and downstream customer workflows.
- Maintain stage-level latency, idempotency, replay, DLQs, and audit.
- Keep logs and immutable audit records PII-free.
- Support redacted outputs where tenant policy requires it.

## 3. Inputs from Stage 5

Stage 6 starts only when Stage 5 has produced a deliverable artifact:

- `MERGE_COMPLETED`
- `MERGE_DEGRADED_APPROVED` after degraded-review approval
- `MERGE_MANUAL_REVIEW_APPROVED` after manual review resolves critical blockers

Stage 6 does not start for unresolved `MERGE_DEGRADED`, `MERGE_MANUAL_INSPECTION_REQUIRED`, or `MERGE_POISON`.

Required inputs:

- `tenant_id`
- `job_id`
- `schema_id`
- `schema_version`
- `merge_policy_version`
- `post_processing_policy_version`
- Stage 5 `document-extraction.json`
- Stage 5 artifact hash and `merge_attempt_id`
- Source chunk artifact hashes
- Field values, citations, confidence, coverage, conflicts, and review outcomes
- Tenant output policy: redaction, suppression, formatting, allowed fields, locale, timezone, currency defaults
- Tenant KMS key reference
- Current usage and cost ledger

Stage 6 validates against the current schema/post-processing policy versions referenced by the job. If the tenant changes schema after Stage 5, that is a new replay attempt and must not mutate the prior result.

## 4. AWS Components

| Component | Role |
|---|---|
| Step Functions Standard | Orchestrates final validation, normalization, packaging, and Stage 7 handoff |
| Lambda post-processing worker | Runs normal documents through schema validation, normalization, redaction, and final artifact creation |
| ECS/Fargate post-processing worker | Handles large documents, very large arrays/tables, or validation workloads likely to exceed Lambda limits |
| DynamoDB On-Demand | Stores validation state, idempotency, final field status, review gates, and usage ledger updates |
| S3 intermediate bucket | Reads Stage 5 artifact and writes Stage 6 final result package before Stage 7 retention placement |
| Schema registry in DynamoDB/S3 | Same tenant schema registry used by Stages 4 and 5; contains post-processing rules alongside extraction and merge rules |
| Rules package / validation library | Versioned deterministic validators for field formats, totals, tax, dates, signatures, and schema-specific constraints |
| Redaction policy engine | Applies tenant output redaction and field suppression rules |
| Degraded review SQS queue | Receives outputs that fail non-critical validation and require review before delivery |
| Manual inspection SQS queue | Receives unresolved critical validation failures |
| EventBridge | Emits validation, review, replay, audit, and Stage 7 handoff events |
| Stage 7 handoff SQS queue | Durable handoff message for final storage, retrieval indexing, and webhook notification |
| Tenant KMS CMKs | Encrypts Stage 6 artifacts and any review/correction artifacts |
| CloudWatch Logs/Metrics and X-Ray | Captures validation latency, failures, redaction counts, final quality metrics, and traces |
| DLQs | Isolate final validation failures, artifact write failures, and poison results |

## 5. Recommended Flow

### 5.1 Start Final Validation

The parent workflow starts Stage 6 after Stage 5 reaches a deliverable state.

The post-processing worker:

1. Loads the Stage 5 artifact by URI and validates its hash.
2. Validates tenant, job, schema, merge attempt, and source artifact ownership.
3. Loads schema, output policy, and post-processing policy versions.
4. Validates final document shape against the tenant JSON Schema.
5. Normalizes fields into canonical and display values.
6. Runs deterministic business validators.
7. Applies final confidence policy.
8. Applies tenant redaction and field suppression rules.
9. Writes `final-result.json`, `customer-result.json`, and `validation-report.json`.
10. Emits Stage 6 audit and operational events.
11. Publishes a durable Stage 7 handoff message.

Recommended Stage 6 states:

| State | Meaning |
|---|---|
| `POST_PROCESSING_QUEUED` | Stage 5 result is deliverable and ready for final validation |
| `POST_PROCESSING_RUNNING` | Schema validation, normalization, or output shaping is active |
| `POST_PROCESSING_VALIDATING` | Deterministic validators are running |
| `POST_PROCESSING_COMPLETED` | Final result package is ready for Stage 7 |
| `POST_PROCESSING_DEGRADED_REVIEW_REQUIRED` | Non-critical final validation needs degraded-review approval |
| `POST_PROCESSING_MANUAL_INSPECTION_REQUIRED` | Critical final validation failed or cannot be verified |
| `POST_PROCESSING_REPROCESS_REQUESTED` | Stronger model or earlier-stage replay is required by policy |
| `POST_PROCESSING_FAILED_RETRYABLE` | Temporary failure; retry allowed |
| `POST_PROCESSING_POISON` | Deterministic validation/configuration failure or retry budget exhausted |

### 5.1.1 Stage 7 Handoff

Stage 6 hands off to Stage 7 using both the parent Step Functions transition and a durable EventBridge/SQS message.

Recommended v1 mechanism:

1. The parent Step Functions workflow transitions from the Stage 6 task to the Stage 7 task after `POST_PROCESSING_COMPLETED`.
2. Stage 6 emits `POST_PROCESSING_COMPLETED` on EventBridge with artifact hashes and non-PII status.
3. EventBridge routes the event to a Stage 7 SQS queue for durable storage, retrieval indexing, and webhook notification work.
4. Stage 7 idempotently consumes the message using `tenant_id + job_id + post_processing_attempt_id + final_result_hash`.

The Step Functions transition keeps the main job lifecycle simple. The EventBridge/SQS handoff gives Stage 7 a durable replayable message and decouples final storage/webhook retries from the Stage 6 worker.

### 5.1.2 Audit Events

Stage 6 emits PII-free audit events through the immutable audit path.

Minimum audit events:

| Event | When emitted |
|---|---|
| `POST_PROCESSING_STARTED` | Final validation starts from a Stage 5 artifact |
| `SCHEMA_VALIDATION_COMPLETED` | JSON Schema and structural validation finishes |
| `NORMALIZATION_COMPLETED` | Field normalization completes |
| `BUSINESS_VALIDATION_COMPLETED` | Deterministic validators complete |
| `OUTPUT_REDACTION_APPLIED` | Tenant redaction or suppression policy changes customer-facing output |
| `POST_PROCESSING_COMPLETED` | Final result is ready for Stage 7 |
| `POST_PROCESSING_DEGRADED_REVIEW_REQUIRED` | Non-critical final validation requires degraded review |
| `POST_PROCESSING_MANUAL_INSPECTION_REQUIRED` | Critical field or business rule requires manual inspection |
| `POST_PROCESSING_REPROCESS_REQUESTED` | Earlier-stage replay or stronger-model recovery is requested |
| `POST_PROCESSING_POISON` | Deterministic failure is isolated |

Events carry tenant/job IDs, post-processing attempt ID, artifact hashes, schema/policy versions, validation counts, timings, status, and non-PII reason codes. They must not include field values, raw document text, summaries, filenames, or unmasked PII.

### 5.2 Policy in the Tenant Schema Registry

Stage 6 uses the same tenant schema registry introduced in Stage 4 and extended in Stage 5. The post-processing policy is a section of the schema document alongside extraction and merge rules, versioned together unless explicitly pinned by `post_processing_policy_version`.

Each field entry declares type, normalization rule, criticality, validators, and redaction policy. For example:

```json
{
  "total_amount": {
    "type": "currency_amount",
    "normalization": "decimal_currency",
    "required": true,
    "criticality": "critical",
    "validators": ["invoice_total_reconciliation"]
  }
}
```

Schema, merge, and post-processing policies should be compatible. CI should fail schema releases where a required output field lacks a post-processing rule, a critical field lacks a validator when one is declared mandatory, or a redaction rule references an unknown field.

### 5.3 Normalization

Stage 6 stores both canonical values and raw values when useful. Canonical values are for customer workflows; raw values preserve what the source said.

All normalized fields carry both a `value` (canonical form) and a `raw_value` (what the source said). Canonical forms follow standard conventions: ISO-8601 dates, ISO-4217 currency codes with decimal strings, format-preserving identifiers, and ordered arrays with row IDs for tables. Ambiguous values must not be guessed — if locale or schema cannot disambiguate, keep `raw_value`, set `value` to `null`, and mark the field `NEEDS_REVIEW` if critical.

### 5.4 Deterministic Validators

Validators run after normalization and before redaction.

Core v1 validators:

| Validator | Applies to | Action |
|---|---|---|
| `json_schema_validation` | All documents | Fails if final output shape is invalid |
| `citation_presence_validation` | All non-null fields | Rejects uncited values |
| `citation_hash_validation` | All citations | Verifies citations against current Stage 5/4/3 artifact hashes |
| `critical_field_coverage_validation` | All schemas | Blocks delivery when critical fields are missing or invalid |
| `invoice_total_reconciliation` | Invoices | Verifies subtotal, tax, line items, discounts, and total; conflicts route to manual inspection |
| `currency_consistency_validation` | Invoices/forms/reports | Flags mixed currencies unless schema allows them |
| `date_order_validation` | Contracts/reports/forms | Validates effective/end dates, report periods, and signature dates |
| `signature_presence_validation` | Contracts/forms | Requires signature fields when schema marks them critical |
| `summary_citation_validation` | Contracts/reports with summaries | Requires summary and key points to cite source spans |
| `redaction_policy_validation` | Tenant policies with redaction | Ensures redacted fields are masked or suppressed in customer-facing output |

Validator outcomes:

| Outcome | Meaning |
|---|---|
| `PASS` | Field/document passed the validator |
| `WARN` | Non-critical issue; can proceed if policy allows |
| `DEGRADED_REVIEW_REQUIRED` | Non-critical output needs degraded review before delivery |
| `MANUAL_INSPECTION_REQUIRED` | Critical issue needs human review |
| `REPROCESS_REQUESTED` | Policy requires stronger model or earlier-stage replay |
| `FAIL_POISON` | Deterministic schema/configuration bug or impossible state |

Invoice totals remain conservative in v1. If the total, subtotal, tax, discount, or line-item aggregation conflicts, Stage 6 records arithmetic evidence and routes to manual inspection. It does not auto-select a final total.

### 5.5 Confidence Policy

Stage 6 should not invent confidence. It derives final confidence from upstream evidence and deterministic validation.

Recommended formula inputs:

- Stage 4 field confidence.
- Stage 5 merge confidence.
- OCR confidence on cited spans.
- Citation validity.
- Validator outcomes.
- Manual review approval, if present.

Final confidence is derived from upstream evidence: minimum of Stage 4 field confidence, Stage 5 merge confidence, and cited OCR confidence, adjusted down for non-critical warnings. Manual reviewer approval overrides to `1.0` with `confidence_source="manual_review"`. Invalid or missing citations cause the field to be rejected as `not_found` — Stage 6 does not invent confidence. The final artifact exposes `confidence`, `confidence_source`, and validator outcomes when output policy allows them.

### 5.6 Manual Review and Reprocessing Gate

Stage 6 can discover issues that Stage 5 did not catch, especially business-rule conflicts.

Recommended routing:

| Condition | Route |
|---|---|
| Critical field missing, invalid, uncited, or failed deterministic validator | Manual inspection queue |
| Invoice total conflict | Manual inspection queue |
| Non-critical validation warning and output still useful | Degraded review queue |
| Model extraction likely caused a critical error and policy allows recovery | Emit `POST_PROCESSING_REPROCESS_REQUESTED` for Stage 4/5 replay |
| Schema/policy configuration bug | `POST_PROCESSING_POISON` |

Stronger-model reprocessing is not performed directly inside Stage 6. Stage 6 emits a bounded replay request that targets the earliest stage capable of correcting the issue:

- Stage 4 stronger-model replay for extraction failures.
- Stage 5 re-merge/re-synthesis for merge or summary failures.
- Manual review for financial totals, critical conflicts, and policy-disallowed automation.

Each job allows at most one Stage 6-initiated reprocess request per target stage. For example, Stage 6 may request one Stage 4 replay and one Stage 5 replay for the same job, but if the same class of failure appears again after that replay, Stage 6 routes the job to manual inspection instead of requesting another automated loop.

Replay is idempotent and must preserve billing policy: platform recovery is non-billable; customer-requested schema/policy changes are billable.

### 5.7 Output Redaction and Field Suppression

Tenant output policy may require redacted customer-facing results while preserving the full internal final artifact for permitted retrieval and audit.

Redaction is wired in by default in v1. Fields without an explicit `redaction_policy` or `suppress_fields` rule flow through to `customer-result.json` unchanged; fields with a redaction or suppression rule are masked, suppressed, or value-hidden according to policy.

V1 supports:

- Mask field value except last N characters.
- Suppress a field entirely from customer-facing output.
- Include only field status/confidence without value.
- Redact summary text when it contains sensitive values.
- Keep citations but suppress quoted source text.

Redaction must happen after validation, not before. Validators need the real values to check totals, dates, and identifiers. The output package can include both:

- `final-result.json`: full result, encrypted with tenant KMS, access-controlled.
- `customer-result.json`: redacted or suppressed result according to tenant output policy. Redaction is enabled by default in v1.

If redaction fails or policy references an unknown field, Stage 6 must not deliver the unredacted result. Route to poison or security review depending on the cause.

### 5.8 Usage, Cost, and Elapsed Time Tracking

Stage 6 is mostly compute and storage, so cost should be small and predictable.

Recommended ledger fields:

- `post_processing_started_at`
- `post_processing_completed_at`
- `post_processing_elapsed_ms`
- `fields_validated`
- `fields_normalized`
- `validators_run`
- `validator_failures`
- `redacted_fields`
- `suppressed_fields`
- `manual_review_routes`
- `degraded_review_routes`
- `reprocess_requests`
- `estimated_post_processing_compute_cost_usd`
- `estimated_post_processing_storage_cost_usd`
- `billable`
- `billing_reason`
- `replay_initiator`

Stage 6 is compute-only — no model inference. Total cost should be well under `$0.005/100 pages`. If the hard stop-loss is breached, it indicates retry storms, oversized outputs, or configuration bugs — route to operator review rather than continuing.

## 6. Latency Budget

Stage 6 should be fast enough to leave time for Stage 7 storage and webhook delivery inside the 5-minute P95 target.

Recommended v1 budget, measured from `POST_PROCESSING_QUEUED` to terminal Stage 6 state:

| Percentile | Budget | Interpretation |
|---|---:|---|
| P50 | `<= 2s` | Normal validation and output shaping |
| P95 | `<= 10s` | Large document, many fields/tables, redaction enabled |
| P99 | `<= 20s` | Large arrays, replay decision, or Fargate worker path |

If queue age plus projected validation time threatens the P95 budget, Stage 6 should apply backpressure to Stage 5 handoff and raise an SLO alert.

## 7. Output Artifact Contract

Stage 6 writes final result artifacts under the intermediate bucket first. Stage 7 is responsible for copying or promoting them to the durable output bucket with the final retention policy.

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage6/final-result.json
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage6/customer-result.json
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage6/validation-report.json
```

Recommended `final-result.json` shape:

```json
{
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "schema_id": "invoice_v1",
  "schema_version": "2026-05-15.v1",
  "merge_attempt_id": "merge_attempt_001",
  "post_processing_attempt_id": "postproc_attempt_001",
  "post_processing_policy_version": "2026-05-15.v1",
  "artifact_version": "2026-05-15.v1",
  "document_status": "POST_PROCESSING_COMPLETED",
  "source_document_extraction_hash": "sha256:...",
  "fields": {
    "invoice_date": {
      "raw_value": "05/15/2026",
      "value": "2026-05-15",
      "display_value": "May 15, 2026",
      "type": "date",
      "confidence": 0.96,
      "confidence_source": "model_and_validation",
      "validation_status": "VALID",
      "validators": [
        {
          "name": "date_format_validation",
          "status": "PASS"
        }
      ],
      "source_citations": []
    }
  },
  "document_summary": null,
  "validation_summary": {
    "fields_validated": 42,
    "critical_fields_valid": 8,
    "warnings": 1,
    "manual_review_required": false,
    "degraded_review_required": false
  },
  "output_policy": {
    "redacted_output_enabled": true,
    "validation_report_access": "operator_support_only",
    "suppressed_fields": []
  },
  "usage": {
    "post_processing_elapsed_ms": 1800,
    "estimated_stage_cost_usd": 0.0004
  }
}
```

`customer-result.json` has the same shape only for fields allowed by output policy. In v1, the redaction engine is enabled by default, so this artifact contains masked/suppressed values where a field policy requires it and an `output_redaction_applied=true` marker when any value changes. Fields without an explicit redaction or suppression rule flow through unchanged.

`customer-result.json` uses `value` for the customer-visible value even when masked. It should not include the unredacted `raw_value`. The full unmasked value remains only in `final-result.json` under stricter access controls.

`validation-report.json` is operator/support-only in v1. This is the safer default for SOC 2 and GDPR because the report can expose validator internals, field IDs, citation IDs, and operational reason codes that are not necessary for normal customer workflows. Customers receive a sanitized validation summary in `customer-result.json`; full report access can be added later through a governed support/export workflow. The report must not include raw field values unless tenant support policy explicitly allows it and the object is encrypted under the tenant KMS key.

## 8. Idempotency and Replay

Stage 6 idempotency key:

```text
tenant_id + job_id + schema_version + post_processing_policy_version + stage5_artifact_hash + output_policy_hash
```

If the same key already has a completed final artifact, Stage 6 returns the existing artifact pointer. If the Stage 5 artifact, schema, post-processing policy, or output policy changes, Stage 6 creates a new post-processing attempt linked to the prior attempt.

Replay scopes:

- Validation-only replay from Stage 5 artifact.
- Redaction-only replay when output policy changes.
- Full Stage 6 replay after schema/post-processing policy change.
- Reprocess request to Stage 4 or Stage 5 when deterministic validation identifies an upstream extraction or merge issue.

Replay must not duplicate billing unless it is customer-requested, follows a customer schema/output-policy change, or follows customer-side cancellation.

## 9. Reliability and Poison-Pill Handling

Retry policy:

| Failure | Retry? | Notes |
|---|---|---|
| S3/DynamoDB transient read/write failure | Yes | Bounded retries with jitter |
| Stage 5 artifact missing but state says completed | Yes, then DLQ | Possible consistency or artifact write issue |
| JSON Schema validation failure on output shape | No for deterministic issue | Manual review or poison based on cause |
| Normalization library transient failure | Yes | Retry same input, then DLQ |
| Unknown field policy reference | No | Schema/configuration bug |
| Redaction failure | No delivery | Route to poison or security review |
| KMS/IAM access denied | No automatic retry | Security/configuration issue |

`POST_PROCESSING_POISON` triggers:

| Trigger | Threshold / rule |
|---|---|
| Schema and post-processing policy are incompatible | Immediate poison |
| Redaction policy cannot be applied safely | Immediate poison/security review |
| Citation hashes cannot resolve against current artifacts | Manual inspection or poison depending on scope |
| Deterministic normalizer cannot parse required critical field and policy lacks review route | Poison |
| Retryable infrastructure failures | `3` attempts, then DLQ |

Poison diagnostics:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage6/poison/diagnostics.json
```

Diagnostics store non-PII IDs, hashes, schema/policy versions, validator names, failure codes, attempt history, and artifact pointers. They must not store raw values, raw document text, or unmasked PII.

## 10. Security and Compliance

Stage 6 handles customer-facing output, so data protection needs to be strict:

- Tenant KMS CMKs encrypt final, customer, validation, review, and poison artifacts.
- IAM scopes post-processing workers to tenant/job prefixes.
- No raw field values, source text, summaries, filenames, or unmasked PII in logs.
- Immutable audit records contain only IDs, hashes, counts, policy versions, timings, status, and non-PII reason codes.
- Redaction is applied before customer-facing output is eligible for delivery.
- Full unredacted results remain access-controlled and are never sent directly to webhooks.
- GDPR deletion must delete or crypto-shred final and customer output artifacts alongside inputs/intermediates.

Stage 6 must not allow tenant schemas or model-derived values to control:

- S3 object keys.
- IAM permissions.
- Webhook destinations.
- Billing flags.
- Retention policy.
- Whether critical validation failures can be ignored.

Those decisions remain deterministic platform policy.

## 11. Observability

Recommended metrics:

| Metric | Purpose |
|---|---|
| `post_processing_stage_latency_ms` | Stage latency budget tracking |
| `post_processing_queue_age_ms` | Backpressure and SLO risk |
| `schema_validation_failure_count` | Schema quality and deployment signal |
| `normalization_failure_count` | Parser/rules quality signal |
| `business_validator_failure_count` | Extraction quality and customer workflow signal |
| `critical_validation_failure_count` | Manual review predictor |
| `degraded_review_route_count` | Quality gate workload |
| `manual_inspection_route_count` | Critical review workload |
| `reprocess_requested_count` | Upstream extraction/merge quality signal |
| `redacted_field_count` | Tenant output policy usage |
| `suppressed_field_count` | Tenant output policy usage |
| `post_processing_cost_estimated_usd` | Cost control |

High-cardinality tenant/job IDs stay in structured logs and X-Ray traces, not unconstrained metric dimensions.

### 11.1 Alerts

Alert on: post-processing latency SLO burn, critical validation failure rate spikes, invoice total conflict rate increases, manual-inspection routing spikes, reprocess request rate increases, DLQ depth growth, redaction failure events, KMS access failures, and PII detected in logs.

Useful traces:

- `Stage6.LoadStage5Artifact`
- `Stage6.SchemaValidation`
- `Stage6.NormalizeFields`
- `Stage6.BusinessValidators`
- `Stage6.ApplyRedaction`
- `Stage6.WriteFinalArtifacts`
- `Stage6.EmitStage7Handoff`

## 12. MLOps and Evaluation

Stage 6 has no model inference by default, but it materially affects quality metrics. Treat validators and normalizers as versioned model-adjacent components.

Required versioning:

- Schema version.
- Post-processing policy version.
- Output policy version.
- Normalization library version.
- Validator library version.
- Redaction policy version.

Evaluation requirements:

- Field-level precision/recall/F1 after final validation.
- Critical-field validity rate after post-processing.
- Validator true-positive and false-positive rates.
- Normalization accuracy by field type and locale.
- Invoice arithmetic validation accuracy.
- Redaction correctness and leakage tests.
- Manual-review routing precision and recall.
- Latency and cost by document type and page count.

Canary guardrails:

- No material decrease in critical-field F1.
- No unexpected increase in manual-inspection or degraded-review routing.
- No increase in redaction leakage.
- No increase in schema validation failures.
- P95 post-processing latency remains within `10s`.
- Stage 6 cost remains under hard stop-loss.

