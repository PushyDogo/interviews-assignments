# Stage 4 - AI Extraction Design

## 1. Goal

Stage 4 turns each normalized Stage 3 chunk artifact into schema-constrained, citation-backed extracted data.

This stage is where SLMs or LLMs are used. It should not re-run OCR, re-chunk the document, or perform document-level merging. Its job is to:

- Build a bounded prompt from `ocr-structure.json`.
- Select the right extraction policy for the document type and schema.
- Call the managed or self-hosted model path.
- Produce valid chunk-level JSON.
- Attach source citations back to page numbers, blocks, bounding boxes, and source spans.
- Track token usage, model latency, quality, and cost.

Stage 5 will merge chunk outputs into one document-level result. Stage 4 should preserve enough provenance and uncertainty for Stage 5 to reconcile conflicts, cross-page fields, repeated entities, and partial success.

## 2. Requirements Covered

Stage 4 covers these requirements from `requirements.md`:

- Apply SLMs or LLMs for document classification, entity extraction, and schema-constrained JSON generation.
- Support invoices, contracts, forms, and corporate reports with document-type-specific extraction policies.
- Enforce schema-valid outputs.
- Include source citations or spans for extracted fields.
- Support chunk-level extraction plus document-level synthesis for contracts and reports.
- Keep prompts token-capped to stay within context-window, latency, and cost constraints.
- Track token usage, TTFT, model latency, model throttles, model error rate, and cost.
- Support managed inference first, while explaining an EKS self-hosted model option.
- Version prompts, schemas, model configurations, and evaluation datasets.

## 3. Inputs from Stage 3

Stage 4 starts when a Stage 3 chunk reaches `OCR_COMPLETED` or `OCR_DEGRADED`.

Required inputs:

- `tenant_id`
- `job_id`
- `chunk_id`
- `schema_id`
- `ocr_structure_uri`
- Stage 2 `page_manifest_hash` and `chunk_manifest_hash`
- Stage 3 `ocr_policy_version`
- Ordered pages, blocks, text, bounding boxes, reading order, source spans, and confidence summaries
- Chunk warnings, unsupported page placeholders, and low-confidence markers
- Tenant KMS key reference
- Current usage ledger

Stage 4 should not consume raw PDFs or raw Textract output directly. It consumes the normalized Stage 3 contract.

## 4. AWS Components

| Component | Role |
|---|---|
| Step Functions Standard / Distributed Map child workflow | Orchestrates chunk-level extraction after Stage 3 |
| DynamoDB On-Demand | Chunk extraction state, idempotency records, token/cost ledger, prompt/model versions |
| S3 intermediate bucket | Stores prompt inputs, model raw responses, validated chunk extraction artifacts, and diagnostics |
| Amazon Bedrock Converse API | Managed v1 model invocation path |
| Bedrock Guardrails | Required v1 control for PII filtering, prompt-injection denial, denied topics, and tenant-safe model interaction |
| Bedrock Prompt Management | Versioned prompt templates and prompt variants |
| Schema registry in DynamoDB/S3 | Versioned tenant schema definitions, extraction policies, and selector expressions |
| Bedrock Knowledge Bases over OpenSearch Serverless | Few-shot retrieval over curated/tenant-approved examples |
| Titan Embeddings V2 | Embeds query/example text for few-shot retrieval |
| SQS / EventBridge | Backpressure, retry, and model-invocation work dispatch |
| EKS + Karpenter + KEDA + vLLM/TGI | V2 or high-volume self-hosted inference option |
| Tenant KMS CMKs | Encrypts prompt inputs, model outputs, and extraction artifacts |
| CloudWatch Logs/Metrics and X-Ray | Structured logs, token/cost metrics, traces, and alarms |
| DLQs | Isolate failed model invocations, schema validation failures, and poison chunks |

## 5. Recommended Flow

### 5.1 Start Chunk Extraction

For each Stage 3 chunk artifact, the workflow:

1. Loads chunk state and validates tenant/job/chunk ownership.
2. Loads the schema definition document and extraction policy for `schema_id`.
3. Checks the chunk status, warnings, unsupported pages, and low-confidence markers.
4. Estimates exact model input tokens using the selected model tokenizer.
5. Applies model admission control and cost stop-loss checks.
6. Builds the prompt package.
7. Invokes the model.
8. Validates the model output against the schema.
9. Writes a chunk-level extraction artifact.

Recommended chunk states:

| State | Meaning |
|---|---|
| `EXTRACTION_QUEUED` | Chunk is ready for model extraction |
| `EXTRACTION_RUNNING` | Prompt assembly or model invocation is active |
| `MODEL_CALLBACK_WAITING` | Async or queued model call is outstanding, if applicable |
| `EXTRACTION_VALIDATING` | Model output is being parsed and schema-validated |
| `EXTRACTION_COMPLETED` | Schema-valid chunk artifact is ready for Stage 5 |
| `EXTRACTION_DEGRADED` | Chunk output is usable but contains low-confidence or missing non-critical fields |
| `EXTRACTION_MANUAL_INSPECTION_REQUIRED` | Critical field extraction cannot be trusted |
| `EXTRACTION_FAILED_RETRYABLE` | Temporary failure; retry allowed |
| `EXTRACTION_POISON` | Retry budget exceeded or deterministic failure |

### 5.2 Extraction Policy by Document Type

The schema controls how Stage 4 extracts fields. V1 should keep policy explicit and deterministic.

| Document type | Stage 4 behavior |
|---|---|
| Invoice | Extract invoice header, vendor/customer, line items when present, totals, tax, currency, dates, payment terms, and citations |
| Contract | Extract parties, effective dates, term/renewal, obligations, governing law, key clauses, signatures, and unresolved cross-page references |
| Form | Extract key-value fields, checkboxes, tables, signatures, required/optional field status, and citations |
| Corporate report | Extract requested metrics, tables, sections, dates, entities, risks, and source-backed summaries |

Stage 4 should avoid pretending that a chunk can resolve global document meaning. For contracts and reports, chunk outputs should include `requires_document_synthesis=true` when fields depend on other chunks.

### 5.3 Prompt Package

The model input should be a structured prompt package, not a raw concatenation of OCR text.

Prompt package sections:

- System instructions and schema rules.
- Schema definition and output contract.
- Document type and extraction policy.
- Chunk metadata: page range, warnings, OCR confidence summary, unsupported pages.
- Ordered text blocks with compact source IDs.
- Table/form hints from Stage 3.
- Few-shot examples only when retrieved examples match the schema and stay within budget.
- Explicit instruction to return only schema-valid JSON.

Source IDs should be short and stable:

```text
p51-b0001
p51-table-002-r03-c02
p52-region-001
```

The prompt must never include raw tenant secrets, auth material, original filenames when they may contain PII, or unrelated page content from other tenants/jobs.

Prompt and schema registry:

- Use Bedrock Prompt Management for versioned prompt templates, prompt variants, and rollout metadata.
- Store tenant schema definitions, JSON Schemas, extraction policies, selector expressions, and field-criticality rules in a tenant-scoped schema registry backed by DynamoDB metadata and S3 versioned schema documents.
- Record `prompt_version`, `prompt_management_arn`, `schema_version`, and `extraction_policy_version` in every chunk artifact.
- Rollback prompt/schema/model changes independently.

Few-shot retrieval is enabled in v1 because extraction quality matters and schemas can vary materially by customer workflow.

Few-shot retrieval uses Bedrock Knowledge Bases with mandatory metadata filters for `tenant_id`, `schema_id`, `document_type`, `prompt_major_version`, and `pii_safe=true` to prevent cross-tenant example leakage.

Few-shot retrieval rules:

- Retrieve examples only from the same `schema_id`, document type, and prompt major version.
- Prefer tenant-approved examples; otherwise use platform-curated examples that contain no tenant PII.
- Cap examples at `3` and `4K` input tokens per chunk.
- Store example IDs and retrieval policy version in the chunk artifact.
- Do not retrieve examples from another tenant's private data.
- Disable examples for a schema if canary evaluation shows token cost increases without field-level quality lift.

### 5.4 Token Budget

Stage 2 estimated tokens conservatively. Stage 4 must calculate prompt tokens locally before invocation. Bedrock does not expose a count-only token endpoint for Llama, so the platform packages the Meta Llama 3.1 tokenizer locally, for example through HuggingFace `tokenizers` using the `meta-llama/Llama-3.1-8B-Instruct` tokenizer or an equivalent approved BPE tokenizer artifact.

Recommended v1 token budgets per chunk:

| Budget | Limit |
|---|---:|
| Target input tokens | `<= 25K` |
| Hard input cap | `30K` |
| Target output tokens | `<= 1.5K` |
| Hard output cap | `2K` |
| Few-shot examples | `<= 3` examples and `<= 4K` input tokens |

If the chunk exceeds the hard input cap, Stage 4 should not call the model. It should request a manifest replay or sub-split the chunk using Stage 2/3 provenance while preserving page order and source IDs.

Token budget controls:

- Use compact block serialization.
- Drop duplicate headers/footers when Stage 2 marked them as repeated furniture.
- Include only fields relevant to the schema.
- Include table cells in compact row/column form.
- Keep few-shot examples short, schema-matched, and retrieved by document type.
- Pin tokenizer version and checksum in the worker image.
- Store `tokenizer_name`, `tokenizer_version`, and `tokenizer_hash` in the chunk artifact.
- Fail closed when local token count and provider usage metrics later diverge materially.

### 5.4.1 Context Window Overflow Policy

Llama 3.1 Instruct has a 128K token context window and a 2K max output limit on Bedrock. A 1,000-page document plus schema, instructions, examples, and citations can easily exceed that window. The architecture must never depend on fitting the whole document into one model call.

Stage 4 uses a hard chunk-level prompt cap of `30K` input tokens even though the model supports 128K. The larger model context is treated as headroom for recovery, dense tables, and future synthesis, not as permission to pass full documents.

The 30K hard cap uses only a fraction of the 128K window; the remaining space is reserved for system instructions (~8K), few-shot examples (~4K), output (~2K), and a safety margin for tokenizer variance, citations, and repair prompts.

Overflow handling:

1. Exact-token count the prompt with the selected model tokenizer.
2. Remove repeated furniture, duplicate OCR text, and non-schema fields.
3. Compress table blocks into compact row/column form.
4. Trim few-shot examples before trimming source content.
5. If still above cap, request a Stage 2/3 sub-split using page and source-span provenance.
6. If a field requires cross-chunk reasoning, emit `requires_document_synthesis=true` and defer resolution to Stage 5 rather than expanding the prompt.
7. Never silently truncate source text. Truncation without provenance would break citations and field coverage.

For document-level extraction in contracts and reports, Stage 5 should use hierarchical synthesis: merge chunk outputs and source-backed summaries, retrieve only the relevant chunk artifacts for each cross-page question, and cite back to Stage 4/3 source spans. This keeps document-level reasoning bounded even when the original document is far larger than 128K tokens.

### 5.5 Model Routing

V1 managed path:

| Route | Model | Use |
|---|---|---|
| Default extraction | Bedrock Llama 3.1 8B Instruct | `$0.22/1M` tokens — cost-controlled default; same family as self-hosted path |
| Stronger recovery | Bedrock Llama 3.1 70B Instruct | `$0.72/1M` tokens — critical-field recovery only |
| Self-hosted | EKS vLLM/TGI serving the same Llama 3.1 Instruct family | V2/high-volume option after break-even validation |

Llama 3.1 8B is the default because it shares the same model family as the self-hosted path, keeping the migration to EKS clean. Batch processing mode (half-price) is reserved for offline replays and evaluation, not the interactive SLA path.

Model routing policy:

- Use Llama 3.1 8B Instruct for normal chunk extraction.
- Use Llama 3.1 70B Instruct automatically for critical fields that fail validation or confidence thresholds in v1.
- Do not route to a self-hosted model in v1 unless it has passed evaluation against the managed path.
- Record `model_id`, `model_version`, `model_route`, `prompt_version`, and `schema_version` in every chunk artifact.

### 5.6 Structured Output and Validation

Stage 4 must require schema-valid JSON. The model response is not accepted until it passes validation.

V1 structured-output default:

- Use Bedrock Converse API with tool use.
- Declare one tool named `submit_extraction`.
- The tool input schema is generated from the tenant/schema JSON Schema.
- The model must call `submit_extraction` exactly once.
- The platform validates the tool input as JSON; free-form assistant text is ignored and treated as invalid output.

If Bedrock structured-output or constrained-decoding mode is available for the selected model in the account, it can replace or reinforce tool use. The contract remains the same: Stage 4 accepts only schema-valid JSON matching the declared extraction schema.

Validation layers:

| Layer | Purpose |
|---|---|
| JSON parse | Response must be valid JSON and not explanatory text |
| JSON Schema validation | Required fields, types, enums, arrays, object shape |
| Citation validation | Every extracted field must cite page/block/source span where possible |
| Confidence validation | Low-confidence critical fields route to stronger model or inspection |
| Cost/token validation | Actual tokens and model route must match ledger |

Hallucination control:

- Every extracted non-null value must have a non-empty `source_citations` array.
- Each citation must resolve to a valid Stage 3 page/block/source span from the same chunk or an allowed cross-chunk synthesis reference in Stage 5.
- If a non-null field has no valid citation, reject the value and mark the field as `not_found` with `rejection_reason="MISSING_CITATION"`.
- The model may output `null`/`not_found` for absent fields, but it may not invent values to satisfy required schema fields.
- Required critical fields that are `not_found` route to stronger-model recovery or manual inspection based on schema policy.

If validation fails:

1. Retry once with a compact repair prompt using the original model response and schema errors.
2. If critical fields still fail, route to stronger model.
3. If stronger model fails, mark chunk `EXTRACTION_MANUAL_INSPECTION_REQUIRED`.

Do not run unbounded repair loops. They create cost spikes and make outputs hard to audit.

## 6. Stage 4 Latency Budget

Stage 4 must leave enough time for Stage 5 merge, Stage 6 post-processing, storage, and notification after Stage 3 consumes up to 180 seconds at P95.

Recommended v1 budget, measured from `EXTRACTION_QUEUED` to `EXTRACTION_COMPLETED`, `EXTRACTION_DEGRADED`, or `EXTRACTION_MANUAL_INSPECTION_REQUIRED`:

| Percentile | Budget | Interpretation |
|---|---:|---|
| P50 | `<= 20s` | Normal chunk extraction without repair |
| P95 | `<= 60s` | Chunk extraction plus one repair or stronger-model recovery |
| P99 | `<= 90s` | Tail chunks with throttling, larger prompts, or recovery |

This is the most reasonable allocation given the 5-minute end-to-end target:

- Stage 1/2 should usually finish quickly after upload acceptance.
- Stage 3 gets the largest budget because async OCR is the slowest managed dependency.
- Stage 4 gets up to 60 seconds at P95 because model latency, schema validation, and one recovery attempt must fit.
- Stages 5-7 get the remaining time for merge, deterministic validation, storage, and webhook notification.

If queue age plus projected model latency exceeds the Stage 4 P95 budget, Stage 4 should apply model backpressure instead of starting more invocations.

These budgets are not meant to be added naively across stages. The pipeline fans out chunks in parallel: Stage 3 OCR, Stage 4 extraction, and later merge work overlap across chunks as upstream chunks complete. The end-to-end P95 budget is defended by critical-path latency, queue age, and the slowest required chunk, not by summing every stage's P95. Stage budget alarms still matter because a sustained Stage 4 queue can become the critical path for large documents.

V1 starting defaults: 100 global active Bedrock calls, 5 per tenant, 5 per job, 1 stronger-recovery call per tenant. These are conservative launch values. Bedrock quota increase requests for Llama 3.1 8B and 70B must be submitted and load-tested before launch; a new account typically starts at 100–200 RPM per model.

## 7. Output Artifact Contract

Stage 4 writes one validated extraction artifact per chunk:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage4/chunks/<chunk_id>/chunk-extraction.json
```

Recommended schema:

```json
{
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "chunk_id": "chunk_003",
  "schema_id": "invoice_v1",
  "schema_version": "2026-05-15.v1",
  "prompt_version": "2026-05-15.v1",
  "model_route": "managed_default",
  "model_id": "meta.llama3-1-8b-instruct-v1:0",
  "stage": "ai_extraction",
  "artifact_version": "2026-05-15.v1",
  "source_ocr_structure_hash": "sha256:...",
  "fields": {
    "invoice_number": {
      "value": "INV-12345",
      "confidence": 0.94,
      "source_citations": [
        {
          "page_number": 51,
          "block_id": "p51-b0001",
          "char_start": 0,
          "char_end": 20,
          "bbox": {
            "left": 0.12,
            "top": 0.08,
            "width": 0.22,
            "height": 0.03
          }
        }
      ],
      "validation_status": "VALID"
    }
  },
  "chunk_warnings": [],
  "requires_document_synthesis": false,
  "usage": {
    "input_tokens": 18000,
    "output_tokens": 1200,
    "few_shot_example_count": 2,
    "retrieval_tokens": 900,
    "estimated_stage_cost_usd": 0.0069
  }
}
```

The artifact may contain PII and must be encrypted with the tenant KMS key. Logs should reference only hashes, counts, IDs, error codes, and token/cost metrics.

## 8. Cost Controls

Stage 4 is one of the largest variable-cost drivers after OCR.

Default managed cost formula (Llama 3.1 8B at `$0.22/1M` tokens input and output):

```text
chunk_cost = (input_tokens + output_tokens) / 1_000_000 * $0.22
```

For a typical chunk with 18K input + 1.2K output tokens: `19,200 / 1,000,000 * $0.22 = ~$0.0042`.
Recovery with Llama 3.1 70B at `$0.72/1M`: same chunk costs ~`$0.0138` — 3.3× more expensive, confirming it must be used selectively.

Recommended v1 stop-loss:

| Limit | Value | Action |
|---|---:|---|
| Target Stage 4 model cost per 100 pages | `<= $0.025` | Normal processing |
| Soft stop-loss per 100 pages | `> $0.035` | Continue only for already-started chunks; apply backpressure to new model work |
| Hard stop-loss per 100 pages | `> $0.050` | Pause for operator/customer approval or route to manual inspection |
| Per-job model cost ceiling | `page_count / 100 * $0.050` | Scales with document size |

This is conservative because the full pipeline target is `$0.10/100 pages`, and Stage 3 OCR can already consume most of that budget for scanned-heavy documents.

Cost controls:

- Calculate exact input tokens before model invocation.
- Maintain per-job and per-tenant token budgets.
- Stop or require approval when projected cost exceeds policy.
- Keep few-shot retrieval within its token budget.
- Keep schema prompts compact and versioned.
- Use stronger models only for critical recovery, not broad default extraction.
- Do not bill tenants for stronger-model recovery in v1; treat it as platform quality recovery. In v2, make stronger-model recovery customer opt-in and billable by tier/policy.
- Mark platform-initiated replays as non-billable in the usage ledger.

Cost ledger fields:

- `input_tokens`
- `output_tokens`
- `retrieval_tokens`
- `model_id`
- `model_route`
- `prompt_version`
- `schema_version`
- `estimated_model_cost_usd`
- `actual_model_cost_usd`
- `billable`
- `billing_reason`
- `replay_initiator`

## 9. Reliability, Idempotency, and Replay

Each model invocation uses an idempotency record scoped by:

```text
tenant_id + job_id + chunk_id + extraction_unit_id + schema_version + prompt_version + model_route + input_artifact_hash
```

If a retry sees the same key and a completed model response, it reuses the response and resumes validation rather than calling the model again. If the same key points to different input content or a different prompt/schema/model route, reject it as a conflict.

Retry policy:

| Failure | Retry? | Notes |
|---|---|---|
| Bedrock throttling | Yes | Exponential backoff with jitter and model admission control |
| Bedrock transient error | Yes | Bounded retries, then DLQ |
| Invalid JSON | Yes | One repair prompt, then stronger model if critical |
| Schema validation failure | Yes | One repair prompt; critical fields go to stronger model |
| Token cap exceeded | No model call | Request sub-split or manifest replay |
| KMS access denied | No automatic retry | Security/configuration issue |
| Deterministic prompt build failure | No | Route to poison/manual inspection depending on reason |

Replay scopes:

- Prompt rebuild from `ocr-structure.json`.
- Validation-only replay from stored raw model response.
- Model replay from the same prompt package.
- Stronger-model replay for critical recovery.

Replay must not duplicate billing unless it is customer-requested, changes the schema/prompt/model policy, or follows customer-side cancellation.

## 9.1 Managed-Model Outage and Throttling Policy

V1 uses Bedrock Llama 3.1 in the platform's primary US geo inference profile. That keeps operations simpler, but it creates a managed-model dependency that must have an explicit failure policy.

| Condition | V1 action |
|---|---|
| Bedrock throttling | Retry with exponential backoff and jitter under the retry budget; keep idempotency key stable |
| Sustained throttling or quota saturation | Open Stage 4 circuit breaker; stop starting new model calls; keep chunks queued |
| Model regional/geo outage | Apply admission backpressure and return `429 Retry-After` for new work requiring extraction |
| Model removed/deprecated | Block new deployments using the removed model; route through evaluated alternate model policy |
| Accepted jobs during outage | Queue where possible; otherwise mark chunks `EXTRACTION_FAILED_RETRYABLE` or `EXTRACTION_MANUAL_INSPECTION_REQUIRED` based on customer policy |
| Outage longer than operational threshold, for example 4 hours | Execute managed-model outage runbook, notify affected tenants, and pause extraction-heavy intake |

V1 fallback order:

1. Retry same model within bounded retry budget.
2. Use evaluated stronger Llama 3.1 70B recovery model for critical chunks if available.
3. Route affected chunks to manual inspection if model recovery is unavailable or cost/latency policy prevents it.

V2 fallback options include cross-region/alternate geo routing where compliance allows, self-hosted Llama on EKS, or a secondary managed provider behind the same Stage 4 artifact contract.

## 10. Poison-Pill Handling

`EXTRACTION_POISON` isolates chunks that should not keep consuming model tokens.

Poison triggers:

| Trigger | Threshold / rule |
|---|---|
| Model throttling/transient errors | `3` model invocation attempts |
| Invalid JSON repair failure | One repair attempt, then stronger model for critical fields; poison only after stronger path fails or deterministic issue is found |
| Schema validation deterministic failure | Immediate poison when schema/prompt mismatch is confirmed |
| Token cap exceeded after sub-split attempt | Poison or manual inspection depending on criticality |
| KMS/IAM denial | Immediate poison after config validation confirms access is denied |
| Cost stop-loss breach | Do not retry; route to approval/manual inspection instead of poison |

Poison artifacts:

```text
s3://document-ai-intermediate/tenant=<tenant_id>/jobs/<job_id>/stage4/poison/<chunk_id>/diagnostics.json
```

Diagnostics store non-PII IDs, hashes, schema/prompt/model versions, token counts, failure codes, attempt history, and artifact pointers. Raw prompt text and model output should not be logged.

## 11. Security and Compliance

Stage 4 inherits the earlier tenant isolation controls:

- Tenant KMS CMKs encrypt prompt packages, raw model responses, and validated extraction artifacts.
- IAM policies restrict workers to tenant/job/chunk-scoped prefixes.
- No raw OCR text, prompt text, or extracted PII in logs.
- Immutable audit records contain only IDs, hashes, counts, timings, policy versions, and non-PII status.
- Prompt and schema registries are tenant-scoped where tenant-specific schemas exist.
- Model requests use the geo endpoint by default when data residency matters.

Bedrock Guardrails are enabled in v1, not optional. The default guardrail policy includes:

- PII filtering and masking policy for model inputs/outputs where supported.
- Prompt-injection/jailbreak denial policy.
- Denied-topic policy for tenant-configured forbidden topics.
- No model invocation may bypass the configured guardrail unless an operator-approved break-glass workflow records the reason in immutable audit.

Prompt injection controls:

- Treat OCR text as untrusted data, not instructions.
- Put extraction instructions in system/schema sections, separate from document text.
- Instruct the model to ignore commands found inside the document.
- Validate all outputs deterministically after model response.
- Never allow model output to control S3 keys, IAM actions, webhook destinations, or billing decisions.

## 12. Observability

Recommended metrics:

| Metric | Purpose |
|---|---|
| `extraction_stage_latency_ms` | Stage latency budget tracking |
| `model_ttft_ms` | Managed/self-hosted model responsiveness |
| `model_latency_ms` | End-to-end invocation latency |
| `input_tokens_total` | Cost and context-budget control |
| `output_tokens_total` | Cost and response-size control |
| `few_shot_retrieval_count` | Retrieval usage and quality/cost correlation |
| `model_throttle_count` | Service quota pressure |
| `model_error_count` | Reliability signal |
| `schema_validation_failure_count` | Prompt/schema quality signal |
| `field_confidence_distribution` | Extraction quality monitoring |
| `manual_inspection_required_count` | Operational load |
| `model_cost_estimated_usd` | Cost control |

High-cardinality tenant/job/chunk IDs should remain in structured logs and traces, not unconstrained metric dimensions.

### 12.1 Alerts

Alert on: model throttle rate spikes, Stage 4 queue age breaching the latency budget, DLQ depth growth, schema validation failure rate spikes, cost-per-chunk exceeding stop-loss thresholds, circuit breaker open events, manual-inspection rate increases, TTFT degradation, and PII detected in logs.

## 13. MLOps and Evaluation

Stage 4 must be evaluated like model behavior, not only like application code.

Required versioning:

- Prompt version.
- Schema version.
- Model ID and model route.
- Extraction policy version.
- Few-shot retrieval policy version.
- Evaluation dataset version.

Evaluation requirements:

- Track precision, recall, and F1 at document, field, and critical-field level.
- Track quality by document type, schema, page quality, OCR route, and model version.
- Use 10% canary rollouts over a 1-hour soak for prompt, schema, model, and extraction-policy changes.

Canary guardrails:

- No material increase in schema validation failures.
- No material decrease in critical-field F1.
- No unexpected token or cost spike.
- No increase in manual-inspection rate.
- No increase in P95 extraction latency.

## 14. Self-Hosted Inference Option

The case study requires explaining a self-hosted LLM option. V1 should be managed-first, but the architecture should preserve the option.

Self-hosted path:

- EKS for inference services.
- Karpenter for GPU node provisioning.
- KEDA scales inference deployments from queue depth, token backlog, or custom metrics.
- vLLM or TGI serves the Llama 3.1 Instruct model family.
- Prometheus/Grafana tracks GPU utilization, GPU memory, queue depth, batch size, tokens/sec, TTFT, model latency, error rate, pod restarts, node provisioning time, and cost per GPU hour.

Use self-hosted inference only after:

- Model quality is comparable for the target schemas.
- Utilization is high enough to beat Bedrock cost per token.
- Tenant isolation, data residency, and security controls are validated.
- Rollback to managed Bedrock is tested.

