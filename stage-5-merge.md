# Stage 5 - Merge and Document-Level Synthesis Design

## 1. Goal

Stage 5 turns the Stage 4 chunk extraction artifacts into one document-level extraction result.

This stage should not re-run OCR, re-parse the PDF, or broadly re-extract every field from the full document. Its job is to:

- Load all completed, degraded, and manual-inspection chunk extraction artifacts.
- Merge chunk-level fields into a single schema-shaped document result.
- Deduplicate repeated values and repeated entities.
- Reconcile conflicts with deterministic rules first.
- Run a bounded document-level synthesis pass only for fields that require cross-chunk reasoning.
- Preserve citations back to Stage 4 fields and Stage 3 page/block/source spans.
- Decide whether the document is complete, partially successful, or requires manual inspection.
- Update the usage ledger with elapsed time and incremental merge/synthesis cost.

The merge result becomes the stable input for Stage 6 post-processing, final validation, storage, and webhook notification.

## 2. Requirements Covered

Stage 5 covers these requirements from `requirements.md`:

- Merge multi-chunk extraction results into a coherent document-level output.
- Support document-level synthesis for contracts and reports where cross-page reasoning is required.
- Keep the LLM context bounded even when the original document exceeds the model context window.
- Preserve source citations or spans for extracted fields.
- Treat failed critical pages differently from non-critical pages.
- Base partial success on field coverage and business importance, not only chunk success count.
- Maintain idempotency, replay, DLQs, and poison handling for merge work.
- Track merge latency, merge conflicts, field coverage, synthesis token usage, and cost.
- Keep audit records immutable and free of PII.

## 3. Inputs from Stage 4

Stage 5 starts after all required Stage 4 chunk workflows have reached a terminal-or-resumable state:

- `EXTRACTION_COMPLETED`
- `EXTRACTION_DEGRADED`
- `EXTRACTION_MANUAL_INSPECTION_REQUIRED`
- `EXTRACTION_POISON`

`EXTRACTION_MANUAL_INSPECTION_REQUIRED` is resumable, not final from the customer's perspective. Stage 5 can produce an interim merge result from the current artifact set, but the job remains in `MERGE_MANUAL_INSPECTION_REQUIRED` until manual review supplies an updated artifact or review decision.

Required inputs:

- `tenant_id`
- `job_id`
- `schema_id`
- `schema_version`
- `merge_policy_version`
- Ordered `chunk_id` list from Stage 2 `chunk-manifest.json`
- One `chunk-extraction.json` artifact per Stage 4 terminal-or-resumable chunk, including unresolved/manual-review markers where applicable
- Stage 2 `page_manifest_hash` and `chunk_manifest_hash`
- Stage 3 `ocr_policy_version`
- Stage 4 `prompt_version`, `model_route`, and extraction policy versions
- Chunk-level fields, confidences, citations, warnings, `not_found` markers, and `requires_document_synthesis` flags
- Unsupported page and low-confidence page summaries
- Tenant KMS key reference
- Current usage and cost ledger

Stage 5 should trust only validated Stage 4 artifacts. It should not read raw model responses except during operator replay or diagnostics.

### 3.1 Manual-Review Handoff and Re-Merge

Stage 5 should not block forever waiting for manual review before producing a useful internal result.

Recommended handoff behavior:

1. When all chunks are in terminal-or-resumable Stage 4 states, Stage 5 starts a merge attempt.
2. Chunks in `EXTRACTION_MANUAL_INSPECTION_REQUIRED` are included as unresolved resumable inputs.
3. Stage 5 writes an interim `document-extraction.json` with current valid fields, unresolved critical fields, and review reasons.
4. The job remains in `MERGE_MANUAL_INSPECTION_REQUIRED` and does not proceed to customer delivery.
5. The manual review tool writes an updated reviewed chunk artifact or reviewed correction artifact.
6. `manual_review.completed` is emitted on EventBridge with tenant/job/chunk IDs, reviewed artifact hash, review outcome, and non-PII reason codes.
7. Stage 5 starts a re-merge against the updated ordered chunk artifact set.
8. If coverage and conflict checks now pass, the job can move to `MERGE_COMPLETED`; otherwise it remains in review or degraded-review state.

This keeps the pipeline resumable without losing provenance. Re-merge creates a new merge attempt linked to the previous attempt; it does not mutate the prior artifact or audit records.

## 4. AWS Components

| Component | Role |
|---|---|
| Step Functions Standard | Orchestrates document-level merge after chunk extraction completes |
| Lambda merge coordinator | Loads chunk states, checks terminal readiness, and starts merge work |
| ECS/Fargate merge worker | Runs heavier merge, dedupe, conflict resolution, and synthesis preparation for large documents |
| DynamoDB On-Demand | Stores job merge state, idempotency records, field coverage summaries, conflict records, and cost ledger updates |
| S3 intermediate bucket | Stores merge inputs, document merge artifact, synthesis prompt packages, raw synthesis responses, and diagnostics |
| Amazon Bedrock Converse API | Bounded document-level synthesis only when required |
| Bedrock Guardrails | Required for any synthesis model call |
| Bedrock Prompt Management | Versioned synthesis prompts and merge-assist prompt templates |
| Schema registry in DynamoDB/S3 | Same tenant schema registry introduced in Stage 4; merge policy lives alongside extraction rules and is versioned with the schema document |
| Degraded review SQS queue | Separate queue for v1 degraded outputs that need lighter manual evaluation before customer delivery |
| Manual inspection SQS queue | Separate queue for critical failures, invoice total conflicts, unsupported critical pages, and unresolved critical fields |
| SQS / EventBridge | Merge work dispatch, retry, audit, and downstream stage events |
| Tenant KMS CMKs | Encrypts merge inputs, synthesis prompts, model outputs, and document artifacts |
| CloudWatch Logs/Metrics and X-Ray | Merge latency, coverage, conflict, synthesis, cost, and trace observability |
| DLQs | Isolate failed merge jobs, synthesis failures, and poison merge records |

## 5. Recommended Flow

### 5.1 Start Document Merge

The parent workflow starts Stage 5 when the chunk extraction barrier is reached.

The merge coordinator:

1. Loads the ordered chunk list from the Stage 2 chunk manifest.
2. Loads Stage 4 terminal-or-resumable state for every chunk.
3. Verifies that every chunk belongs to the same tenant, job, schema, and manifest hash.
4. Loads the schema merge policy.
5. Builds a field coverage matrix.
6. Runs deterministic merge and conflict resolution.
7. Runs bounded synthesis only for fields marked as requiring document synthesis.
8. Writes `document-extraction.json`.
9. Emits merge completion, manual-inspection, or poison events.

Recommended merge states:

| State | Meaning |
|---|---|
| `MERGE_QUEUED` | All required chunk terminal-or-resumable states are available and merge can start |
| `MERGE_RUNNING` | Deterministic merge, dedupe, or coverage computation is active |
| `SYNTHESIS_REQUIRED` | One or more fields require bounded document-level synthesis |
| `SYNTHESIS_RUNNING` | Bedrock synthesis call is active |
| `MERGE_VALIDATING` | Document artifact is being schema, citation, and coverage validated |
| `MERGE_COMPLETED` | Document-level artifact is ready for Stage 6 |
| `MERGE_DEGRADED` | Internal state only in v1; artifact may be usable but must be manually evaluated before customer delivery |
| `MERGE_MANUAL_INSPECTION_REQUIRED` | Critical fields, critical pages, or unresolved conflicts require human review |
| `MERGE_DEGRADED_APPROVED` | Degraded-review workflow approved the result for Stage 6 |
| `MERGE_MANUAL_REVIEW_APPROVED` | Manual-review workflow resolved critical blockers and approved the result for Stage 6 |
| `MERGE_FAILED_RETRYABLE` | Temporary failure; retry allowed |
| `MERGE_POISON` | Retry budget exceeded or deterministic merge failure |

### 5.1.1 Stage 5 Audit Events

Stage 5 emits PII-free audit events through the same immutable audit path established in Stage 1. Events carry tenant/job IDs, merge attempt ID, artifact hashes, counts, timings, policy versions, status, and reason codes. They must not include field values, prompt text, OCR text, summaries, filenames, or raw citations containing customer text.

Minimum audit events:

| Event | When emitted |
|---|---|
| `MERGE_STARTED` | A merge attempt starts from the current ordered chunk artifact set |
| `MERGE_COMPLETED` | Document merge completes and can proceed to Stage 6 |
| `SYNTHESIS_REQUESTED` | Stage 5 starts a bounded synthesis call |
| `SYNTHESIS_COMPLETED` | Synthesis output is validated or rejected |
| `MERGE_DEGRADED` | Merge result requires degraded-review queue before delivery |
| `MERGE_MANUAL_INSPECTION_REQUIRED` | Critical fields, total conflicts, or unresolved critical pages require manual review |
| `MERGE_POISON` | Deterministic merge failure or retry budget exhaustion isolates the job |
| `REMERGE_STARTED` | Manual review or artifact replacement triggers a new merge attempt |
| `REMERGE_COMPLETED` | Re-merge completes after updated reviewed artifacts |
| `REMERGE_MANUAL_INSPECTION_REQUIRED` | Re-merge still requires manual review |

### 5.1.2 Review Approval Workflow

Stage 5 has two review approval paths before Stage 6 can start:

| Review path | Input state | Approval event | Output state |
|---|---|---|---|
| Degraded review | `MERGE_DEGRADED` | `degraded_review.approved` | `MERGE_DEGRADED_APPROVED` |
| Critical manual review | `MERGE_MANUAL_INSPECTION_REQUIRED` | `manual_review.completed` with `outcome=approved` or `outcome=corrected` | `MERGE_MANUAL_REVIEW_APPROVED` after re-merge passes coverage and conflict checks |

The review service is the owner of these transitions. It writes a reviewed correction or approval artifact, emits the approval event on EventBridge, and records a PII-free audit event with tenant/job IDs, review ID, reviewed artifact hash, reviewer role, outcome, and reason codes. Stage 5 then re-validates the current artifact set before marking the merge as approved for Stage 6.

Stage 6 should consume only `MERGE_COMPLETED`, `MERGE_DEGRADED_APPROVED`, or `MERGE_MANUAL_REVIEW_APPROVED`.

### 5.2 Merge Policy in the Schema Registry

Stage 5 uses the same tenant schema registry introduced in Stage 4. The merge policy is a section of the schema definition document alongside extraction rules, field criticality, selector expressions, and output schemas. It is versioned together with the schema document so extraction and merge behavior cannot drift independently.

Each schema must define how fields merge. This keeps merge behavior explicit and testable.

Each field entry in the merge policy declares its cardinality, criticality, merge strategy, and minimum coverage. For example:

```json
{
  "invoice_number": {
    "cardinality": "singleton",
    "criticality": "critical",
    "merge_strategy": "highest_confidence_with_conflict_check",
    "conflict_policy": "manual_inspection_on_distinct_high_confidence_values",
    "minimum_coverage": "required"
  }
}
```

V1 merge strategies:

| Strategy | Use |
|---|---|
| `first_valid_by_document_order` | Fields where first occurrence is authoritative |
| `last_valid_by_document_order` | Amendments, signatures, or final values where later pages override earlier pages |
| `highest_confidence_with_conflict_check` | Singleton fields such as invoice number, dates, totals, and parties |
| `append_preserve_order` | Clauses, report sections, comments, and other ordered repeated content |
| `append_dedupe_preserve_order` | Line items, repeated entities, tables, and form rows |
| `aggregate_numeric` | Count, sum, min, max, or date-range fields when schema explicitly allows aggregation |
| `document_synthesis_required` | Cross-chunk summaries, obligations, risk summaries, or clauses that cannot be resolved locally |
| `manual_inspection_required` | Critical fields where automation is not allowed to decide conflicts |

No field should rely on an implicit merge rule. If a schema lacks a merge rule for a field, Stage 5 should fail schema validation before producing a customer-facing result.

### 5.3 Deterministic Merge First

Most Stage 5 work should be deterministic.

For every field, the merge worker:

1. Collects all chunk candidates.
2. Drops values that Stage 4 marked invalid.
3. Keeps explicit `not_found` markers for coverage accounting.
4. Normalizes candidate values using field-specific normalization.
5. Deduplicates equivalent candidates.
6. Applies the schema merge strategy.
7. Computes merged confidence and citation coverage.
8. Records merge decisions and conflicts.

Normalization is field-type-specific: dates parse to ISO-8601, amounts normalize currency code and decimal separator, identifiers trim whitespace, text fields normalize whitespace only while preserving source-backed wording. Original values are always kept as `raw_value` alongside the normalized form.

Deterministic merge must not invent values. If every candidate is `not_found`, the merged field remains `not_found`.

### 5.4 Conflict Resolution

Conflicts are expected in multi-page documents. The design should expose them rather than hide them.

Conflict classes:

| Conflict | Example | V1 action |
|---|---|---|
| Equivalent duplicate | `ACME Inc.` vs `Acme, Inc` | Normalize and merge citations |
| Low-confidence disagreement | Two values, one below threshold | Prefer valid higher-confidence value; keep warning |
| High-confidence singleton conflict | Two distinct invoice totals both above threshold | Manual inspection if critical; warning or synthesis if non-critical |
| Invoice total conflict | Header total, subtotal, tax, or line-item total disagree | Always manual verification in v1 |
| Page-order override | Earlier draft term differs from later signed term | Apply schema rule such as `last_valid_by_document_order` |
| Cross-chunk dependency | Contract obligation depends on definition in another section | Bounded synthesis |
| Unsupported critical source | Field depends on unsupported page | Manual inspection |

V1 defaults: candidate confidence floor is `0.90` for critical fields and `0.75` for non-critical; citation coverage is required for all non-null values. When a critical singleton field has multiple distinct high-confidence values and the schema does not define a deterministic override rule, Stage 5 must route the field to `MERGE_MANUAL_INSPECTION_REQUIRED`. A model should not be used to guess a critical value unless the schema explicitly allows synthesis and the output remains citation-gated.

### 5.5 Partial Success and Field Coverage

Stage 5 is where the platform decides whether the document can continue, continue degraded, or needs manual inspection.

Coverage should be calculated by field importance:

```text
critical_field_coverage = valid_critical_fields / expected_critical_fields
required_field_coverage = valid_required_fields / expected_required_fields
optional_field_coverage = valid_optional_fields / expected_optional_fields
critical_page_coverage = processed_critical_pages / expected_critical_pages
```

Default v1 document outcome rules:

| Condition | Outcome |
|---|---|
| All critical fields valid, required coverage `>= 95%`, unresolved normal conflicts `<= 3` | `MERGE_COMPLETED` |
| All critical fields valid, required coverage below `95%`, missing fields are non-critical | Internal `MERGE_DEGRADED`, then manual evaluation before customer delivery |
| Any critical field is `not_found`, invalid, uncited, or conflicted | `MERGE_MANUAL_INSPECTION_REQUIRED` |
| Any critical page is unsupported or failed after recovery | `MERGE_MANUAL_INSPECTION_REQUIRED` |
| Merge policy missing or incompatible with schema | `MERGE_POISON` |

These thresholds are schema-configurable in v1. Tenant-specific overrides are deferred to v2 because v1 has only base-level tenants.

Partial success is still useful, but v1 should not send degraded outputs directly to customers. A degraded document can keep valid extracted fields, explicit missing-field reasons, unresolved conflict records, and review routing information, but it must go through a separate degraded-review queue before webhook delivery. This queue is distinct from the manual inspection queue used for critical failures and invoice total conflicts.

### 5.6 Bounded Document-Level Synthesis

Some fields cannot be resolved chunk by chunk. Examples include contract summaries, obligations spread across sections, corporate-report risk summaries, or definitions referenced by later clauses.

Stage 5 uses synthesis only when one of these is true:

- A Stage 4 chunk sets `requires_document_synthesis=true`.
- The schema field rule uses `document_synthesis_required`.
- Deterministic merge finds a non-critical conflict that the schema permits synthesis to resolve.
- A contract/report schema declares `summary_required: true` and asks for a source-backed summary across multiple chunks.

Document-level summaries are doc-type and schema specific. They are emitted only when the schema declares `summary_required: true` and the projected synthesis cost stays within the Stage 5 cost budget. If summaries push the job above the hard Stage 5 stop-loss or materially threaten the overall `$0.10/100-page` representative target, the v1 system should skip automated summary generation, mark the summary field as `not_found` with `rejection_reason="COST_POLICY_DEFERRED"`, and call out self-hosted inference as the v2 path to bring summary economics under the cost barrier.

The `document_summary` output field carries a status (`generated`, `not_found`, `cost_deferred`, `manual_review_required`), source-backed summary text, key points, risks/open items, and citations. The specific schema for each document type must be validated with domain experts before production launch.

The synthesis pass must stay bounded:

- Do not send the full document.
- Do not send raw OCR for all pages.
- Build a synthesis packet from chunk-level extracted fields, source-backed summaries, candidate conflicts, and only the cited source spans relevant to the question.
- Use the same Bedrock Llama 3.1 8B default route as Stage 4, with Llama 3.1 70B only for critical recovery if the schema allows it.
- Use Bedrock Converse tool use with one tool named `submit_document_synthesis`.
- Require every non-null synthesized value to carry citations back to Stage 4 candidates and Stage 3 source spans.
- Reject uncited synthesized values as `not_found`.

Synthesis token budget: ≤20K target input, 30K hard cap, ≤1.5K target output, 2K hard cap, maximum 10 synthesis questions per document.

The cap of `10` synthesis questions is a v1 guardrail to bound model calls, token cost, review complexity, and tail latency. It should cover the minimal contract/report summary schema plus a small number of cross-chunk fields. If more than `10` synthesis questions are required, Stage 5 should process the highest-priority questions by schema criticality and route overflow questions to manual inspection or degraded review with `rejection_reason="SYNTHESIS_QUESTION_LIMIT_EXCEEDED"`. If the synthesis packet exceeds the hard cap, Stage 5 should split synthesis by field group. If a single field group still exceeds the cap, route that field to manual inspection rather than silently truncating citations.

### 5.7 Context Window Strategy

Stage 5 is the main defense against the 128K context-window problem at document level.

The design is hierarchical:

1. Stage 4 extracts each chunk with local citations.
2. Stage 5 merges structured chunk outputs deterministically.
3. Stage 5 asks the model only targeted cross-chunk questions.
4. Stage 5 provides only relevant chunk candidates and cited source spans to the synthesis prompt.
5. Stage 6 validates the final schema and business rules.

This means a 1,000-page document can produce a final document result without ever placing the full source text into one LLM prompt.

### 5.8 Usage, Cost, and Elapsed Time Tracking

Stage 5 must update the per-job usage ledger.

Recommended ledger fields:

- `merge_started_at`
- `merge_completed_at`
- `merge_elapsed_ms`
- `chunks_merged`
- `fields_seen`
- `fields_valid`
- `fields_not_found`
- `fields_conflicted`
- `critical_fields_valid`
- `critical_fields_expected`
- `synthesis_invocation_count`
- `synthesis_input_tokens`
- `synthesis_output_tokens`
- `estimated_merge_compute_cost_usd`
- `estimated_synthesis_cost_usd`
- `actual_synthesis_cost_usd`
- `billable`
- `billing_reason`
- `replay_initiator`

Approximate v1 cost policy:

| Cost item | Target |
|---|---:|
| Deterministic merge compute | `<= $0.001 / 100 pages` |
| Stage 5 synthesis target | `<= $0.010 / 100 pages` |
| Stage 5 soft stop-loss | `> $0.015 / 100 pages` |
| Stage 5 hard stop-loss | `> $0.025 / 100 pages` |

Most documents should not need synthesis, so Stage 5 should usually be a small compute cost. Synthesis-heavy contracts and reports can spend more, but they must remain inside the per-job cost ceiling and be visible in the ledger. Document-level summaries are optional in v1 and should be skipped when their projected model cost would push the job past the hard stop-loss.

Replay billing follows earlier policy:

| Replay mode | Tenant billable? | Notes |
|---|---|---|
| Platform retry after transient merge failure | No | Same inputs and same merge policy |
| Validation-only replay from existing merge artifact | No | No new model call |
| Platform synthesis recovery after model/provider failure | No | Quality/reliability recovery |
| Customer-requested replay with same schema/policy | Yes | Customer requested additional processing |
| Replay after customer changes schema/policy | Yes | New customer-selected work |
| Replay after customer-side cancellation | Yes for work already incurred | Consistent with Stage 1 acceptance billing |

## 6. Latency Budget

Stage 5 must be fast because the slowest chunk from Stages 3 and 4 already consumes most of the end-to-end budget.

Recommended v1 budget, measured from `MERGE_QUEUED` to terminal merge state:

| Percentile | Budget | Interpretation |
|---|---:|---|
| P50 | `<= 5s` | Deterministic merge with no synthesis |
| P95 | `<= 30s` | Merge plus bounded synthesis for a small number of fields |
| P99 | `<= 60s` | Large document, conflict-heavy merge, or one recovery attempt |

Stage 5 should start as soon as the required chunk barrier is met. It should not wait for non-critical poison chunks if the schema says their fields are optional and coverage thresholds can still be met. It must wait for chunks that contain critical pages or expected critical fields.

## 7. Output Artifact Contract

Stage 5 writes one document-level artifact:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage5/document-extraction.json
```

Recommended schema:

```json
{
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "schema_id": "invoice_v1",
  "schema_version": "2026-05-15.v1",
  "merge_policy_version": "2026-05-15.v1",
  "merge_attempt_id": "merge_attempt_001",
  "stage": "merge",
  "artifact_version": "2026-05-15.v1",
  "source_chunk_manifest_hash": "sha256:...",
  "source_page_manifest_hash": "sha256:...",
  "source_chunk_artifacts": [
    {
      "chunk_id": "chunk_001",
      "artifact_uri": "s3://document-ai-intermediate/tenant=tenant_123/jobs/job_01/stage4/chunks/chunk_001/chunk-extraction.json",
      "artifact_hash": "sha256:...",
      "status": "EXTRACTION_COMPLETED"
    }
  ],
  "document_status": "MERGE_COMPLETED",
  "fields": {
    "invoice_number": {
      "value": "INV-12345",
      "confidence": 0.94,
      "merge_status": "VALID",
      "merge_strategy": "highest_confidence_with_conflict_check",
      "source_candidates": ["chunk_001:invoice_number"],
      "source_citations": [
        {
          "chunk_id": "chunk_001",
          "page_number": 1,
          "block_id": "p1-b0001",
          "char_start": 0,
          "char_end": 20,
          "bbox": {
            "left": 0.12,
            "top": 0.08,
            "width": 0.22,
            "height": 0.03
          }
        }
      ]
    }
  },
  "coverage": {
    "critical_field_coverage": 1.0,
    "required_field_coverage": 0.97,
    "optional_field_coverage": 0.72,
    "critical_page_coverage": 1.0
  },
  "conflicts": [],
  "manual_inspection": {
    "required": false,
    "reasons": []
  },
  "usage": {
    "merge_elapsed_ms": 4200,
    "synthesis_invocation_count": 0,
    "synthesis_input_tokens": 0,
    "synthesis_output_tokens": 0,
    "estimated_stage_cost_usd": 0.0009
  }
}
```

This artifact may contain PII and must be encrypted with the tenant KMS key. Logs and immutable audit events should reference artifact hashes, counts, status codes, timing, and policy versions only.

## 8. Idempotency and Replay

Stage 5 idempotency key:

```text
tenant_id + job_id + schema_version + merge_policy_version + ordered_chunk_artifact_hashes
```

If the same key already has a completed merge artifact, Stage 5 returns the existing artifact. If the same job tries to merge with different chunk artifact hashes, schema version, or merge policy version, Stage 5 treats it as a new merge attempt and records why the previous result is no longer current.

Citation stability rule: Stage 5 validates citations against the current Stage 4 chunk artifacts referenced by hash. If manual review or Stage 4 replay replaces a chunk artifact and block IDs or source spans change, Stage 5 must re-merge and re-validate citations from the updated artifact set. It must not carry forward stale citations from a prior merge attempt.

Replay scopes:

- Deterministic merge replay from Stage 4 artifacts.
- Synthesis-only replay for selected fields.
- Validation-only replay from `document-extraction.json`.
- Full Stage 5 replay after schema or merge-policy change.

Replay must be bounded and auditable. It must not mutate prior immutable audit records; it writes a new merge attempt record linked to the previous attempt.

## 9. Reliability and Poison-Pill Handling

Retry policy:

| Failure | Retry? | Notes |
|---|---|---|
| S3/DynamoDB transient read/write failure | Yes | Bounded retries with jitter |
| Missing chunk artifact but chunk state says completed | Yes, then DLQ | Possible consistency or write failure |
| Merge policy missing field rule | No | Deterministic schema/configuration error |
| Citation resolver failure | No if deterministic, otherwise retry | Do not accept fields with unresolved citations |
| Synthesis model throttling/transient error | Yes | Same bounded Bedrock policy as Stage 4 |
| Synthesis invalid JSON/tool response | Yes | One repair attempt, then manual inspection if critical |
| KMS/IAM access denied | No automatic retry | Security/configuration issue |

`MERGE_POISON` triggers:

| Trigger | Threshold / rule |
|---|---|
| Missing or incompatible merge policy | Immediate poison |
| Deterministic merge code cannot parse a valid Stage 4 artifact | Immediate poison after artifact validation |
| Citation graph cannot resolve required source spans | Immediate manual inspection or poison depending on scope |
| Synthesis repair failure | One repair attempt, then manual inspection for critical fields |
| Retryable infrastructure failures | `3` attempts, then DLQ |

Poison diagnostics:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage5/poison/diagnostics.json
```

Diagnostics store non-PII IDs, hashes, schema/merge/synthesis versions, failure codes, attempt history, field names, and artifact pointers. They must not store raw field values, raw prompts, or raw OCR text.

## 10. Security and Compliance

Stage 5 inherits the same security model as earlier stages:

- Tenant KMS CMKs encrypt merge inputs, synthesis prompts, model responses, and output artifacts.
- IAM scopes merge workers to tenant/job prefixes.
- No raw field values, OCR text, synthesis prompts, or PII in logs.
- Immutable audit records contain only IDs, hashes, counts, policy versions, timings, status, and non-PII reason codes.
- GDPR erasure deletes or crypto-shreds document artifacts and tenant keys; immutable audit remains because it stores no PII.
- Bedrock Guardrails are required for synthesis calls.
- Prompt injection controls still apply because synthesized fields use source text from untrusted documents.

Stage 5 must not allow model output to decide:

- Webhook destination.
- Billing status.
- S3 object keys.
- IAM permissions.
- Whether a critical conflict can be ignored.

Those decisions remain deterministic application logic.

## 11. Observability

Recommended metrics:

| Metric | Purpose |
|---|---|
| `merge_stage_latency_ms` | Stage latency budget tracking |
| `merge_queue_age_ms` | Backpressure and SLA risk |
| `chunks_merged_count` | Document size and fan-in pressure |
| `field_candidate_count` | Merge complexity |
| `field_conflict_count` | Quality and schema signal |
| `critical_field_conflict_count` | Manual-inspection predictor |
| `field_coverage_ratio` | Partial success monitoring |
| `critical_field_coverage_ratio` | Customer-impact quality signal |
| `synthesis_invocation_count` | Model usage in merge |
| `synthesis_input_tokens_total` | Cost and context-budget control |
| `synthesis_output_tokens_total` | Cost and output-size control |
| `synthesis_validation_failure_count` | Prompt/schema quality signal |
| `manual_inspection_required_count` | Operational load |
| `merge_cost_estimated_usd` | Cost control |

High-cardinality tenant/job/chunk IDs should stay in structured logs and X-Ray traces, not as unconstrained metric dimensions.

### 11.1 Alerts

Alert on: merge latency SLO burn, queue age spikes, DLQ depth growth, critical field conflict rate increases, critical field coverage drops, synthesis cost spikes, synthesis validation failures, manual-inspection rate increases, KMS access failures, and PII detected in logs.

Useful traces:

- `Stage5.MergeCoordinator`
- `Stage5.LoadChunkArtifacts`
- `Stage5.ComputeCoverage`
- `Stage5.ResolveConflicts`
- `Stage5.DocumentSynthesis`
- `Stage5.ValidateCitations`
- `Stage5.WriteDocumentArtifact`

## 12. MLOps and Evaluation

Stage 5 has model behavior only when synthesis runs, but merge quality still needs evaluation.

Required versioning:

- Merge policy version.
- Schema version.
- Synthesis prompt version.
- Synthesis model ID and route.
- Citation resolver version.
- Field normalization library version.

Evaluation requirements:

- Document-level precision, recall, and F1.
- Field-level and critical-field F1 after merge.
- Citation validity rate after merge.
- Conflict resolution accuracy.
- Partial-success classification accuracy.
- Manual-inspection precision and recall.
- Synthesis quality for contracts and reports.
- Cost and latency by document type and page count.

Canary guardrails:

- No material decrease in critical-field F1.
- No material increase in unresolved critical conflicts.
- No material decrease in citation validity.
- No unexpected increase in synthesis token usage.
- No increase in manual-inspection rate unless it catches known bad outputs.
- P95 merge latency remains within `30s`.

