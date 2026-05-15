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
