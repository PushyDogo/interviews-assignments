# Risk Mitigation Plan

## 1. Summary

This plan identifies the major risks in the platform and explains how the architecture prevents, detects, contains, and recovers from them.

## 2. Case Study Interpretation and Scope

The case study explicitly calls out service quotas, cost overruns, and model hallucinations. It also implies risks around Lambda limits, 128K context windows, burst traffic, provider outages, GPU orchestration, and compliance.

This artifact covers:

- Risk register for the full seven-stage pipeline.
- Mitigations for technical, operational, ML, cost, security, and compliance risks.
- Early warning signals.
- Runbooks and escalation paths.
- Residual risk that should remain visible in the interview discussion.


## 4. Risk Control Architecture

![Security Architecture](./images/Risk.png)

The platform mitigates risk through layered controls:

| Control | Purpose |
|---|---|
| Admission control | Prevent accepting work that cannot be processed fairly or within current capacity. |
| Durable queues | Buffer bursts and provider slowdowns without losing jobs. |
| Bounded concurrency | Prevent service quota exhaustion and tenant starvation. |
| Idempotency | Make retries safe and prevent duplicate billing, OCR starts, model calls, and webhooks. |
| Cost stop-loss | Stop or require approval before expensive OCR/model paths exceed policy. |
| Validation gates | Block hallucinated, uncited, low-confidence, or conflicting outputs. |
| DLQs and poison queues | Isolate repeated failures so they do not block healthy traffic. |
| Manual review | Provide a safe path for critical ambiguity and non-automatable decisions. |
| Observability | Detect drift in latency, quality, cost, and provider health early. |

## 5. Latency Risk Mitigation

Latency risk comes from two sources: processing time and waiting time. Processing time is the actual work. Waiting time is queue age caused by bursts, throttles, provider limits, or tenant contention.

Mitigations:

- Use S3 Transfer Acceleration to reduce upload bottlenecks even though upload time is outside SLA.
- Fan out chunks with Step Functions Distributed Map.
- Track slowest required chunk as the true critical path.
- Avoid OCR for direct-text pages.
- Avoid image enhancement for clean digital PDFs.
- Use async OCR callbacks and poll fallback.
- Apply admission backpressure when queue age threatens SLA.
- Keep Stage 5/6/7 deterministic and fast.

Early warning signals:

- Stage 2 processing time above typical budget.
- Stage 3 callback latency > expected window.
- Stage 4 model queue age increasing.
- Bedrock or Textract throttles.
- Per-tenant active job imbalance.
- Manual-review backlog for critical fields.

Residual risk:

OCR heavy, low-quality, 1,000-page documents can be outside representative P95.

## 6. Cost Risk Mitigation

Cost risk is dominated by OCR and model tokens.

Mitigations:

- Classify pages before OCR.
- Use embedded text when valid.
- Use region-level OCR for hybrid pages when coordinates are reliable.
- Default structured OCR off unless schema selectors require it.
- Estimate cost before starting paid OCR/model work.
- Track actual usage after each stage.
- Apply soft and hard stop-loss thresholds.
- Skip optional document summaries when synthesis would exceed policy.
- Treat platform recovery as non-billable but still visible in ledger.

Residual risk:

The 0.10 target holds for the representative mix at current pricing (Textract 0.0010/page text/table at volume, Llama 3.1 8B $0.22/1M tokens). Scanned-heavy documents or structured OCR usage can push per-document cost well above target

## 7. Service Quota and Burst Risk

The 50,000-document burst cannot be allowed to become 50,000 simultaneous OCR or model calls.

Mitigations:

- Admission control at API and tenant level.
- Separate created jobs from accepted jobs.
- Quota consumed only after accepted upload.
- Per-stage concurrency controls.
- Tenant token buckets.
- Distributed Map max concurrency.
- SQS buffering for callbacks and handoffs.
- Quota increase requests as launch readiness items.

Quota risk table:

| Service | Risk | Mitigation |
|---|---|---|
| Textract | Async OCR job limits and throttles | OCR admission control, active OCR unit caps, circuit breaker. |
| Bedrock | RPM/token quotas | Model concurrency caps, quota increase requests, backpressure. |
| Step Functions | Map concurrency and transition pressure | Distributed Map with configured concurrency and Express children where appropriate. |
| DynamoDB | Hot partitions and write throttles | High-cardinality job PK, sharded tenant/quota counters, on-demand mode. |
| S3 | Request hot prefixes | Tenant/job-scoped keys with high-cardinality distribution. |
| Lambda | Concurrency and timeouts | Reserved concurrency where needed; Fargate for heavy PDFs. |
| Webhooks | Customer endpoint failures | Async dispatcher, retries, DLQ, result available independently. |

## 8. Model Hallucination and Quality Risk

The model can produce plausible but wrong data. The platform therefore treats model output as a proposal, not truth.

Mitigations:

- Tool-use structured output; free-form text is invalid.
- JSON Schema validation.
- Required source citations for every non-null value.
- Citation resolver validates page/block/span against current Stage 3 artifacts.
- Confidence thresholds route critical failures to stronger model or manual inspection.
- Invoice totals and high-value fields use deterministic validators.
- Merge refuses to guess critical conflicts.
- Stage 6 rejects invalid citations and unresolved critical fields.


## 9. Security and Compliance Risk

Security risk is highest at boundaries: upload, logs, model prompts, audit, webhooks, and operator access.

Mitigations:

- API key plus signed JWT.
- WAF and TLS 1.2+.
- S3 SecureTransport deny.
- Per-tenant KMS Keys.
- GuardDuty malware gate before acceptance.
- Emit-time log scrubbing.
- CloudWatch log leakage detection.
- PII-free immutable audit.
- Redacted customer outputs.
- Pointer-only signed webhooks.
- Operator access through separate audited tooling.

## 10. Provider Outage Risk

v1 relies on managed Textract and Bedrock in the primary region

Textract outage response:

1. Open Stage 3 circuit breaker.
2. Stop starting new OCR jobs.
3. Keep accepted chunks queued when policy allows.
4. Apply Stage 1 admission backpressure for OCR-heavy work.
5. Move jobs to retryable or manual states if customer policy requires terminal updates.
6. Preserve replay eligibility.

Bedrock outage response:

1. Open Stage 4 model circuit breaker.
2. Stop new model invocations.
3. Apply model queue backpressure.
4. Use retry budget with jitter.
5. Route critical work to evaluated alternate model only if available and policy-approved.
6. Otherwise route to manual inspection or wait in queued state.


## 10. Self-Hosted Inference Risk

Risks:

- GPU utilization too low to beat Bedrock.
- vLLM throughput lower than expected with long prompts.
- Spot interruptions increase tail latency.
- Karpenter node provisioning is slower than queue growth.
- Model quality differs from managed route.
- GPU cluster adds security and patching surface.

Mitigations:

- Do not launch self-hosted path until benchmarked with production-like prompts.
- Compare managed and self-hosted quality with the same eval set.
- Keep Bedrock fallback.
- Use KEDA for model-server scaling and Karpenter for node provisioning.
- Use reserved baseline plus spot burst plus on-demand overflow.
- Track cost per 1M tokens in Grafana.
- Require self-hosted route to pass latency, cost, citation, schema, and F1 gates before traffic cutover.


## 11. Manual Inspection Risk

Manual review is necessary, but it can become a bottleneck.

Mitigations:

- Separate degraded-review queue from critical manual-inspection queue.
- Keep interim artifacts and reason codes.
- Prioritize critical fields and high-value customers/jobs.
- Track backlog age and reviewer throughput.
- Use reviewed artifacts to improve eval datasets and schema policies.
- Do not block non-critical optional fields if schema coverage thresholds are met.