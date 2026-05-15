# Stage 7 - Storage, Retrieval, and Notification Design

## 1. Goal

Stage 7 is the final stage of the document-processing pipeline. It promotes Stage 6 artifacts into durable output storage, updates the job to a terminal state, exposes retrieval through the API, delivers webhook notifications, and starts cleanup/lifecycle actions for intermediate artifacts.

This stage should not mutate extraction values, re-run validation, or send large results directly to customer endpoints. Its job is to:

- Consume the Stage 6 handoff message idempotently.
- Copy or promote `customer-result.json`, `final-result.json`, and `validation-report.json` to the correct durable storage locations.
- Apply tenant retention and lifecycle policies.
- Update durable job state and retrieval indexes.
- Generate customer-safe retrieval pointers.
- Send signed webhook notifications for terminal job states.
- Ensure webhook delivery is retryable, bounded, and observable.
- Emit immutable PII-free audit events.
- Trigger cleanup of short-lived intermediate artifacts.

Stage 7 is where the pipeline becomes externally visible as complete.

## 2. Requirements Covered

Stage 7 covers these requirements from `requirements.md`:

- Store original inputs, intermediate artifacts, final JSON outputs, and audit events separately.
- Persist final results to S3.
- Update durable job status on every major state transition.
- Provide an API to retrieve job status and final result.
- Support webhook notification on terminal job states without pushing large result payloads directly to customer endpoints.
- Apply default 365-day retention for customer data unless tenant policy requires otherwise.
- Support GDPR access and erasure workflows.
- Keep immutable audit records PII-free.
- Ensure webhook delivery is idempotent, retryable, DLQ-backed, and observable.

## 3. Inputs from Stage 6

Stage 7 starts after Stage 6 reaches `POST_PROCESSING_COMPLETED`.

Stage 6 hands off through both:

- The parent Step Functions next-task transition.
- A `POST_PROCESSING_COMPLETED` EventBridge event routed to the Stage 7 SQS queue.

Required handoff fields:

- `tenant_id`
- `job_id`
- `schema_id`
- `schema_version`
- `post_processing_attempt_id`
- `stage6_artifact_uris`
- `final_result_hash`
- `customer_result_hash`
- `validation_report_hash`
- `output_policy_hash`
- `tenant_kms_key_id`
- `webhook_id`, if configured
- `terminal_status_candidate`
- Current usage and cost ledger

Stage 7 idempotency key:

```text
tenant_id + job_id + post_processing_attempt_id + final_result_hash
```

If Stage 7 sees the same key again, it must return the existing durable output pointers and avoid duplicate webhook delivery.

## 4. AWS Components

| Component | Role |
|---|---|
| Step Functions Standard | Final workflow transition and terminal job status orchestration |
| Stage 7 SQS queue + DLQ | Durable handoff from Stage 6 and retry isolation |
| Storage worker Lambda | Promotes artifacts to output storage, updates indexes, and emits terminal events |
| Webhook dispatcher Lambda | Delivers signed webhook notifications with bounded retries |
| DynamoDB On-Demand | Job terminal state, retrieval index, output manifest, idempotency records, webhook delivery state |
| S3 output bucket | Durable customer output artifacts with tenant-scoped prefixes and lifecycle policies |
| S3 intermediate bucket | Source of Stage 6 artifacts and lifecycle-managed intermediate objects |
| S3 original input bucket | Stores original uploaded PDF under input retention policy |
| S3 audit bucket with Object Lock | Immutable PII-free audit stream |
| API Gateway + Lambda | Job status and result retrieval APIs |
| Secrets Manager | Stores webhook signing secrets and endpoint configuration |
| EventBridge | Terminal job events, audit events, cleanup events, and webhook dispatch events |
| Tenant KMS CMKs | Encrypts output artifacts and result manifests |
| CloudWatch Logs/Metrics and X-Ray | Storage, retrieval, webhook, lifecycle, and terminal-state observability |

## 5. Recommended Flow

### 5.1 Promote Final Artifacts

The Stage 7 storage worker consumes the handoff message and:

1. Loads the Stage 6 artifact metadata and validates hashes.
2. Checks or creates the Stage 7 idempotency record.
3. Validates tenant/job ownership and tenant KMS key.
4. Copies Stage 6 artifacts from the intermediate bucket to the output bucket.
5. Writes an `output-manifest.json`.
6. Updates the DynamoDB job record with durable output pointers.
7. Marks the job terminal or review-terminal.
8. Emits terminal and audit events.
9. Enqueues webhook dispatch if a `webhook_id` exists.
10. Emits cleanup events for short-lived intermediate artifacts.

Durable output paths:

```text
s3://document-ai-output/tenant=<tenant_id>/jobs/<job_id>/attempt=<post_processing_attempt_id>/customer-result.json
s3://document-ai-output/tenant=<tenant_id>/jobs/<job_id>/attempt=<post_processing_attempt_id>/final-result.json
s3://document-ai-output/tenant=<tenant_id>/jobs/<job_id>/attempt=<post_processing_attempt_id>/validation-report.json
s3://document-ai-output/tenant=<tenant_id>/jobs/<job_id>/attempt=<post_processing_attempt_id>/output-manifest.json
```

`customer-result.json` is the default customer-facing artifact. `final-result.json` and `validation-report.json` are access-controlled and are not included in webhook payloads.

### 5.2 Output Manifest

Stage 7 writes one output manifest per completed post-processing attempt.

Recommended shape:

```json
{
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "post_processing_attempt_id": "postproc_attempt_001",
  "schema_id": "invoice_v1",
  "schema_version": "2026-05-15.v1",
  "terminal_status": "COMPLETED",
  "artifacts": {
    "customer_result": {
      "uri": "s3://document-ai-output/tenant=tenant_123/jobs/job_01/attempt=postproc_attempt_001/customer-result.json",
      "hash": "sha256:...",
      "content_type": "application/json",
      "customer_accessible": true
    },
    "final_result": {
      "uri": "s3://document-ai-output/tenant=tenant_123/jobs/job_01/attempt=postproc_attempt_001/final-result.json",
      "hash": "sha256:...",
      "content_type": "application/json",
      "customer_accessible": false
    },
    "validation_report": {
      "uri": "s3://document-ai-output/tenant=tenant_123/jobs/job_01/attempt=postproc_attempt_001/validation-report.json",
      "hash": "sha256:...",
      "content_type": "application/json",
      "customer_accessible": false
    }
  },
  "retention": {
    "policy": "default_365_days",
    "expires_at": "2027-05-15T00:00:00Z",
    "legal_hold": false
  },
  "webhook": {
    "configured": true,
    "delivery_status": "PENDING"
  }
}
```

The manifest may contain artifact URIs and hashes but must not contain extracted values or raw document text.

### 5.3 Terminal Job States

Stage 7 owns the customer-visible terminal status.

Recommended terminal states:

| State | Meaning |
|---|---|
| `COMPLETED` | Final customer result is available |
| `COMPLETED_WITH_REVIEW` | Manual/degraded review approved the result and final customer result is available |
| `MANUAL_INSPECTION_REQUIRED` | Pipeline cannot produce a deliverable result without human action |
| `FAILED` | Non-recoverable platform or configuration failure |
| `CANCELLED` | Customer cancellation completed |
| `DELETION_REQUESTED` | GDPR/tenant deletion workflow is active |
| `DELETED` | Customer content and indexes have been deleted or crypto-shredded |

Stage 7 must not mark a job `COMPLETED` until `customer-result.json` has been durably written, indexed, and retrieval-ready. Webhook delivery can happen after terminal state is durable; webhook failure does not roll back the terminal job state.

### 5.4 Retrieval API

The API should return metadata and retrieval pointers, not embed large JSON results by default.

Recommended endpoints:

```text
GET /v1/jobs/{job_id}
GET /v1/jobs/{job_id}/result
GET /v1/jobs/{job_id}/artifacts/customer-result
```

V1 retrieval behavior:

| Endpoint | Response |
|---|---|
| `GET /v1/jobs/{job_id}` | Job status, timestamps, schema ID/version, page count, terminal state, cost summary, and result availability |
| `GET /v1/jobs/{job_id}/result` | Metadata plus a short-lived signed URL for `customer-result.json`; no inline result body in v1 |
| `GET /v1/jobs/{job_id}/artifacts/customer-result` | Short-lived signed URL for `customer-result.json` |

Do not expose `final-result.json` or `validation-report.json` through customer APIs in v1. Operator/support access goes through separate audited tooling.

Always returning a signed URL keeps the API response shape consistent across small and large results and avoids accidental large-payload behavior in API Gateway/Lambda.

Retrieval authorization:

- Verify API key and signed JWT.
- Check tenant/job ownership in DynamoDB.
- Check job terminal state and artifact availability.
- Issue short-lived signed URLs, default `15 minutes`.
- Use tenant KMS and S3 IAM policies to prevent cross-tenant reads.
- Record access events in immutable PII-free audit.

### 5.5 Webhook Notification

V1 uses pre-registered webhook endpoints from Stage 1. Jobs reference a `webhook_id`; they cannot supply arbitrary callback URLs.

Webhook notifications are emitted for all customer-visible terminal states, including success and non-success outcomes. This includes `COMPLETED`, `COMPLETED_WITH_REVIEW`, `MANUAL_INSPECTION_REQUIRED`, `FAILED`, `CANCELLED`, `DELETION_REQUESTED`, and `DELETED`. For non-success states, the payload includes status and non-PII reason codes plus links to the job status endpoint; it does not include result artifact pointers unless a customer result is actually available.

Webhook payloads are pointer-only:

```json
{
  "event_type": "job.completed",
  "event_id": "evt_01HW...",
  "tenant_id": "tenant_123",
  "job_id": "job_01HW...",
  "status": "COMPLETED",
  "schema_id": "invoice_v1",
  "schema_version": "2026-05-15.v1",
  "result": {
    "available": true,
    "retrieval_url": "https://api.example.com/v1/jobs/job_01HW/result",
    "artifact_type": "customer-result.json",
    "hash": "sha256:..."
  },
  "usage": {
    "page_count": 100,
    "estimated_total_cost_usd": 0.071
  },
  "created_at": "2026-05-15T12:00:00Z"
}
```

Webhook payloads must not include extracted field values, raw text, PII, presigned S3 URLs, or validation-report details.

Webhook security:

- Sign payloads with HMAC-SHA256 using a per-webhook secret from Secrets Manager.
- Include `X-DocumentAI-Event-Id`, `X-DocumentAI-Timestamp`, and `X-DocumentAI-Signature`.
- Reject endpoints that are not pre-registered and verified.
- Enforce HTTPS.
- Rotate webhook secrets with overlap.
- Store delivery attempts without payload PII.

Webhook delivery:

| Rule | V1 value |
|---|---:|
| Initial dispatch target | `<= 5s` after terminal state is durable |
| Max delivery attempts | `8` |
| Retry policy | Exponential backoff with jitter |
| Max retry window | `24h` |
| Timeout per attempt | `5s` connect/read budget |
| Success status codes | `2xx` |
| Permanent failure | `410 Gone`, disabled endpoint, or tenant-deleted webhook |

Webhook failure does not change the completed job result. It changes only webhook delivery state.

### 5.6 Retention and Lifecycle

Default v1 retention:

| Data class | Default retention | Notes |
|---|---:|---|
| Original input PDF | `365 days` | Move to cold storage after `7 days`; tenant policy can shorten/extend where contractually allowed |
| Customer output artifacts | `365 days` | Includes `customer-result.json` |
| Internal final artifacts | `365 days` | Includes `final-result.json` and support-only validation report |
| Intermediate OCR/model artifacts | Short-lived after final acceptance | Earlier-stage policies control exact windows |
| Rejected uploads | `7 days` warm + `21 days` cold | From Stage 1 |
| Immutable audit records | Audit retention policy in Object Lock | PII-free; not deleted for GDPR erasure |

Stage 7 emits cleanup events after durable output promotion:

- Delete or expire enhanced images after their retry window.
- Delete or expire normalized OCR artifacts after final output is accepted and replay/manual-review windows close.
- Preserve manifests and non-PII hashes needed for replay/audit.
- Preserve original input and final outputs per tenant retention.

Retention is policy-driven and must not be controlled by model output or tenant-supplied document content.

### 5.7 GDPR Access and Erasure

Stage 7 maintains the mapping needed for GDPR workflows:

- Tenant ID.
- Job ID.
- Original input object key.
- Output manifest key.
- Customer/final/validation artifact keys.
- Intermediate artifact prefixes.
- Retrieval index records.
- Webhook delivery records.

Erasure workflow:

1. Mark job `DELETION_REQUESTED`.
2. Stop new processing/replay/webhook dispatch for the job.
3. Delete customer content objects from input, intermediate, and output buckets.
4. Delete retrieval index records and artifact manifests that contain customer content.
5. Delete webhook delivery payload records if they contain tenant/job-specific customer references beyond audit-safe IDs.
6. Crypto-shred tenant data when allowed by retention/legal-hold policy.
7. Mark job `DELETED`.
8. Keep immutable audit records because they contain only PII-free operational facts.

If legal hold or contract retention prevents deletion, mark the job as deletion-restricted with a PII-free reason code and expose that status through the tenant support path.

## 6. Latency Budget

Stage 7 must leave room for storage and notification inside the 5-minute P95 target. Webhook retries can continue after the terminal result is available, but the first delivery attempt should be fast.

Recommended v1 budget:

| Percentile | Budget | Scope |
|---|---:|---|
| P50 | `<= 2s` | Promote artifacts, update job, enqueue webhook |
| P95 | `<= 10s` | Larger artifacts, S3/DynamoDB retries, first webhook attempt queued |
| P99 | `<= 20s` | Tail storage/indexing retries |

The customer-visible result is available when durable output storage and retrieval index updates complete. Webhook delivery state is tracked separately.

## 7. Cost Controls

Stage 7 is storage, API, and webhook-request heavy. Its per-document cost should be small.

Recommended ledger fields:

- `storage_started_at`
- `storage_completed_at`
- `stage7_elapsed_ms`
- `output_artifact_bytes`
- `output_artifact_count`
- `retrieval_index_writes`
- `webhook_configured`
- `webhook_attempts`
- `webhook_delivery_status`
- `estimated_storage_cost_usd`
- `estimated_request_cost_usd`
- `estimated_webhook_cost_usd`
- `billable`
- `billing_reason`

Approximate v1 cost policy:

| Cost item | Target |
|---|---:|
| Output artifact storage/write | `<= $0.001 / 100 pages` |
| Retrieval/index updates | `<= $0.0005 / job` |
| Webhook dispatch | `<= $0.0005 / job` |
| Stage 7 soft stop-loss | `> $0.003 / job` |
| Stage 7 hard stop-loss | `> $0.005 / job` |

Storage cost continues over retention lifetime. Cost dashboards should show both processing cost and retained-storage cost.

## 8. Idempotency and Replay

Stage 7 idempotency key:

```text
tenant_id + job_id + post_processing_attempt_id + final_result_hash
```

Idempotent operations:

- Artifact promotion: skip copy if destination object exists with expected hash and KMS key.
- Output manifest write: conditional write by manifest key and hash.
- DynamoDB terminal state update: conditional transition from non-terminal to terminal, or same terminal state with same manifest hash.
- Webhook event creation: one event ID per terminal state and post-processing attempt.
- Webhook delivery: one delivery state machine per event ID and endpoint.

Replay scopes:

- Storage promotion replay.
- Retrieval index replay.
- Webhook delivery replay.
- Full Stage 7 replay from Stage 6 handoff message.

Webhook replay must not create duplicate logical events. It can create additional delivery attempts for the same `event_id`.

## 9. Reliability and Poison-Pill Handling

Retry policy:

| Failure | Retry? | Notes |
|---|---|---|
| S3 transient copy/write failure | Yes | Bounded retries with jitter |
| DynamoDB conditional conflict | Usually no | Return existing state if same idempotency key; investigate if mismatched hash |
| KMS/IAM access denied | No automatic retry | Security/configuration issue |
| Retrieval index write transient failure | Yes | Retry, then DLQ |
| Webhook 5xx/timeout | Yes | Retry up to delivery budget |
| Webhook 4xx | Depends | 408/429 retry; most other 4xx fail delivery |
| Webhook endpoint disabled/deleted | No | Mark delivery permanently failed |

Poison triggers:

| Trigger | Threshold / rule |
|---|---|
| Destination output hash mismatch | Immediate poison/security review |
| Output manifest conflicts with different artifact hashes | Immediate poison |
| KMS/IAM denies output promotion | Immediate poison/config review |
| Stage 7 queue message exceeds retry budget | DLQ and operator review |
| Webhook retries exhausted | Webhook DLQ; job remains terminal |

Webhook poison does not poison the job result. It only marks notification failure and creates an operator/support task.

## 10. Security and Compliance

Stage 7 is the primary customer retrieval boundary:

- Output artifacts are encrypted with tenant KMS CMKs.
- Output bucket enforces `aws:SecureTransport`.
- S3 object keys are platform-generated and tenant/job-scoped.
- Bucket policy denies cross-tenant access except through platform roles.
- Retrieval APIs verify API key, signed JWT, tenant/job ownership, and result availability.
- Signed URLs are short-lived and point only to customer-accessible artifacts.
- `final-result.json` and `validation-report.json` are not customer-retrievable in v1.
- Webhooks are signed and pointer-only.
- Webhook payloads exclude extracted values, raw text, PII, validation internals, and presigned S3 URLs.
- Immutable audit records remain PII-free.
- GDPR erasure deletes customer content while preserving PII-free audit facts.

Stage 7 must not allow tenant schemas, model output, or customer document content to control:

- S3 object keys.
- Retention policy.
- Webhook destination.
- Signed URL TTL.
- Billing state.
- IAM permissions.

## 11. Observability

Recommended metrics:

| Metric | Purpose |
|---|---|
| `stage7_latency_ms` | Storage/retrieval handoff latency |
| `output_artifact_bytes` | Storage cost and retrieval size tracking |
| `output_promotion_failure_count` | Storage reliability |
| `retrieval_index_write_failure_count` | API availability risk |
| `job_terminal_state_count` | Success/failure/manual-review terminal mix |
| `webhook_delivery_attempt_count` | Delivery volume |
| `webhook_success_count` | Delivery health |
| `webhook_failure_count` | Endpoint or platform delivery issue |
| `webhook_retry_exhausted_count` | Support load |
| `retrieval_api_latency_ms` | Customer API performance |
| `signed_url_issued_count` | Retrieval usage |
| `gdpr_deletion_requested_count` | Privacy operations load |
| `gdpr_deletion_completed_count` | Privacy workflow health |
| `stage7_cost_estimated_usd` | Storage/request/webhook cost control |

Useful traces:

- `Stage7.ConsumeHandoff`
- `Stage7.PromoteArtifacts`
- `Stage7.WriteOutputManifest`
- `Stage7.UpdateJobTerminalState`
- `Stage7.EnqueueWebhook`
- `Stage7.DispatchWebhook`
- `Stage7.RetrieveResult`
- `Stage7.DeleteJobArtifacts`

High-cardinality tenant/job IDs should stay in structured logs and traces, not unconstrained metric dimensions.

## 12. MLOps and Operations

Stage 7 has no model behavior, but it is critical for customer experience and compliance.

Required versioning:

- Output manifest schema version.
- Retrieval API version.
- Webhook payload version.
- Retention policy version.
- Deletion workflow version.

Operational runbooks:

- S3 output promotion failures.
- KMS/IAM access denied.
- Retrieval API elevated 4xx/5xx.
- Webhook endpoint outage.
- Webhook retry exhaustion.
- Stage 7 DLQ growth.
- GDPR deletion stuck or legal-hold blocked.
- Output hash mismatch or manifest conflict.

Canary guardrails:

- No increase in terminal-state write failures.
- No webhook signing failures.
- No retrieval authorization bypass.
- No signed URL TTL policy drift.
- P95 Stage 7 latency remains within `10s`.
- No customer-facing access to support-only artifacts.

## 13. Closed Stage 7 Decisions

| Decision | V1 choice | V2 refinement |
|---|---|---|
| Handoff | Step Functions transition plus EventBridge/SQS handoff from Stage 6 | Separate storage and notification workflows if scale requires |
| Output storage | Promote Stage 6 JSON artifacts to tenant-scoped S3 output bucket | Multi-region output replication if needed |
| Customer retrieval | Always return API metadata plus short-lived signed URL for `customer-result.json`; no inline result body in v1 | Customer-selectable delivery modes |
| Webhook payload | Pointer-only signed event, no extracted values or presigned S3 URL | Optional small inline summaries if explicitly allowed |
| Webhook endpoint model | Pre-registered `webhook_id` only | Per-job callbacks after allowlisting/domain verification |
| Webhook retry | 8 attempts over 24h with jitter | Tenant-specific delivery policies |
| Terminal webhooks | Emit for success and non-success terminal states | Tenant-specific event subscriptions |
| Retention | Default 365 days for input/final output; move original inputs to cold storage after 7 days; short-lived intermediates; PII-free immutable audit | Tenant-specific retention policies |
| Customer artifacts | `customer-result.json` only in customer APIs | Governed access/export for validation report |
| Stage 7 latency budget | P50 `<= 2s`, P95 `<= 10s`, P99 `<= 20s` | Tune from output size and webhook volume |
