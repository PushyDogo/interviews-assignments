# Assumptions

Assumptions that directly justify time, cost, and operational metrics. Design decisions are tracked separately in `design-decisions-records-tracker.md`.

---

## Stage 1 — Ingestion

| # | Assumption | Justifies | Impact if Wrong |
|---|---|---|---|
| A1 | AWS region is `us-east-1`. | All cost estimates in Stage 1 and the break-even model in Stage 7. | Service availability (GuardDuty Malware Protection, S3 Transfer Acceleration) and pricing differ by region; cost ledger must be re-run. |
| A2 | The 5-minute P95 SLA clock starts at `accepted_at` — after upload is received, metadata-validated, malware-scanned, and quota-checked. Upload time is excluded. | The latency target and all stage-level latency budgets downstream. | If the clock starts at upload initiation, the architecture cannot meet the 5-minute target for 300 MB files over variable networks. |
| A3 | GuardDuty Malware Protection scan completes in seconds for typical files. Worst-case scan latency is budgeted at under 30 seconds. | The ingestion latency budget and the claim that malware gating does not meaningfully affect the 5-minute SLA. | If scan latency regularly exceeds 30 seconds, it consumes a significant share of the 5-minute budget and the gate design needs re-evaluation. |
| A4 | S3 Transfer Acceleration costs ~`$0.04/GB`; GuardDuty Malware Protection costs ~`$0.09/GB scanned + $0.215/1K objects`. Both are approximate and exclude free-tier effects. | The per-document ingestion cost estimate (~`$0.04` at the 300 MB ceiling) and the overall `$0.10` budget headroom left for OCR and LLM stages. | Pricing changes require re-running the cost ledger before claiming the `$0.10` target is met. |
| A5 | Lambda async retry is capped at 2 attempts (~1 min, ~2 min intervals). Total auto-retry window is ~3 minutes before a failed validation invocation reaches the EventBridge on-failure destination. | The operational metric for time-to-failure-detection and the manual replay SLO for validation failures. | If transient outages regularly exceed 3 minutes, DLQ depth will grow and require operator intervention before auto-recovery; retry cap or retry window must be increased. |

## Stage 2 — Pre-processing

| # | Assumption | Justifies | Impact if Wrong |
|---|---|---|---|
| A6 | Stage 2 latency target is under 30 seconds for typical digital or mixed PDFs. A 1,000-page low-quality scan is the P95-tail case and is expected to be slower. | The overall 5-minute end-to-end SLA budget allocation across stages. | If typical documents regularly exceed 30 seconds in pre-processing, the remaining latency budget for OCR and extraction is insufficient to meet the 5-minute P95 target. |
| A7 | Token estimation uses 1 token ≈ 4 characters for English-heavy text, plus a 20% safety margin. This is model-agnostic and conservative by design. | The 25K soft cap / 30K hard cap per chunk and the claim that chunks stay well within the 128K context window after adding prompt overhead. | If actual token counts consistently exceed estimates, chunks will overflow the model context window in Stage 5. Stage 3 recalculates exact tokens using the model-specific tokenizer as a safety check. |
| A8 | Enhanced page images are retained for 24 hours then deleted. This is sufficient for OCR retry within the same job lifecycle. | Intermediate storage cost and the claim that pre-processing does not materially increase per-document storage cost. | If OCR retry windows need to be longer than 24 hours (e.g., due to extended Textract outages), the retention policy must be extended. |

## Stage 3 — OCR and Structure Extraction

| # | Assumption | Justifies | Impact if Wrong |
|---|---|---|---|
| A9 | The platform will exceed 1M Textract pages/month, qualifying for volume pricing: text and table OCR at `$0.0010/page`, form (key-value) extraction at `$0.0040/page`. | All Stage 3 cost estimates, the cost formula, and the $0.10/100-page budget model. | If the platform stays below 1M pages/month, text/table OCR costs `$0.0015/page` — increase Stage 3 cost estimates by 50% for text/table pages. |
| A10 | Inbound document mix: ~60% digital-text pages (bypass OCR), ~30% hybrid (selective OCR), ~10% fully scanned (full OCR). Form-heavy and fully-scanned workloads are treated as cost exceptions, not the design centre. | The claim that the representative-mix cost target of `$0.10/100 pages` is achievable. | If the actual mix skews heavier toward scanned or form-heavy documents, per-document cost will exceed the target. The architecture exposes this via `cost_policy_result` tracking rather than hiding it. |
| A11 | Expected Textract async OCR callback window: 10 seconds–2 minutes for typical chunks. Poll fallback activates at 3 minutes. | The Stage 3 latency budget (P50 ≤60s / P95 ≤180s) and the claim that async OCR fits within the 5-minute end-to-end SLA. | If Textract callback latency regularly exceeds 2 minutes, the P95 budget is at risk. The 3-minute poll fallback is the safety valve. |
| A12 | V1 starting OCR concurrency defaults: 100 global active OCR units, 5 per tenant, 2 per job. These must be validated by load test against Textract service quotas before launch. | The burst protection and tenant fairness claims in Stage 3. | If defaults are too low, queue age will grow and the 5-minute SLA will miss during bursts. If too high, Textract quota limits will cause throttling. |

## Stage 4 — AI Extraction

| # | Assumption | Justifies | Impact if Wrong |
|---|---|---|---|
| A13 | Llama 3.1 8B Instruct on Bedrock: `$0.22/1M` tokens (input and output). Llama 3.1 70B Instruct: `$0.72/1M` tokens (input and output). Both rates are for on-demand inference. | Stage 4 cost formula, the ≤$0.025/100-page target, and the stop-loss thresholds ($0.035 soft / $0.050 hard). | If pricing increases or batch discounts change, re-run the cost model. 70B at full price must remain reserved for critical recovery — routing broad extraction through it makes the $0.10 pipeline target impossible. |
| A14 | A typical 100-page document produces ~4 chunks of ~25 pages. At 18K input + 1.2K output tokens per chunk with 8B: 4 × $0.0042 = ~$0.017 Stage 4 cost per 100 pages. 70B recovery on all 4 chunks would cost ~$0.055 — above the Stage 4 stop-loss. | The claim that Stage 4 fits within the $0.025/100-page target under representative load and that 70B is cost-safe only for selective recovery. | If documents are denser (higher token counts per chunk) or chunks are smaller (more chunks), Stage 4 cost increases proportionally. |
| A15 | Token budget per chunk: ≤25K target input, 30K hard cap, ≤1.5K target output, 2K hard cap, ≤4K few-shot examples. These leave adequate headroom within the 128K context window for system instructions, citations, and repair prompts. | The Stage 2 chunking strategy, the overflow policy, and the claim that no chunk will hit the model context limit in normal operation. | If prompt overhead (schema, instructions, few-shot) regularly exceeds estimates, the effective content budget per chunk shrinks and more chunks will need sub-splitting. |
| A16 | V1 starting Bedrock concurrency defaults: 100 global active calls, 5 per tenant, 5 per job, 1 stronger-recovery call per tenant. Bedrock quota increase requests must be submitted before launch; new accounts typically start at 100–200 RPM per model. | The Stage 4 burst protection and tenant fairness claims. | If quota requests are not submitted before launch, burst traffic will hit Bedrock throttling limits before platform-level concurrency controls become the binding constraint. |

## Stage 5 — Merge and Document-Level Synthesis

| # | Assumption | Justifies | Impact if Wrong |
|---|---|---|---|
| A17 | Stage 5 cost targets: deterministic merge ≤`$0.001/100 pages`; synthesis target ≤`$0.010/100 pages`; hard stop-loss at `$0.025/100 pages`. | The claim that Stage 5 leaves meaningful budget headroom within the `$0.10/100-page` pipeline target. | If synthesis is triggered frequently (e.g., contract-heavy workloads), Stage 5 cost will be at the upper end. Document summaries must be skipped when the hard stop-loss would be breached. |
| A18 | Synthesis token budget per document: ≤20K target input, 30K hard cap, ≤1.5K target output, 2K hard cap, maximum 10 synthesis questions. | The Stage 5 latency budget (P95 ≤30s) and cost model. | If cross-chunk reasoning regularly requires more than 10 questions or denser synthesis packets, the cap must be raised and the latency and cost targets re-evaluated. |
| A19 | Document success thresholds: critical field coverage = 100%, required field coverage ≥ 95%, unresolved normal conflicts ≤ 3. These are configurable per schema in v1; tenant-specific overrides are v2. | The merge outcome rules and the partial-success classification accuracy claim. | If thresholds are too strict, manual-inspection rate will be high. If too loose, low-quality outputs reach customers. Production telemetry should drive threshold tuning. |