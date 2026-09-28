# Perturbation Log

## 1. Policy Pipeline
- **Perturbation:** Blanked out the eductible field in the input policy document so the required information was genuinely missing.
- **Hypothesis:** The pipeline should halt retry loops immediately without hallucinating a replacement value and escalate directly to human review.
- **Actual Result:** The run logged `ESCALATED (missing required deductible)` in `pipeline-run.txt`, emitted single API attempt behavior, and produced a `HUMAN_REVIEW` status in `routing_decisions.json`.

## 2. Mortgage Extraction
- **Perturbation:** Supplied input with mismatched stated total income versus line item sum (`fixtures/documents/income_sum_mismatch.txt` for Marcus T. Hollingsworth, stated: $10,892.17 vs calculated sum: $9,642.17).
- **Hypothesis:j* The mathematical consistency validator will sum the base ($5,416.67), bonus ($1,250.00), commission ($2,140.00), overtime ($385.50), and other ($450.00), detect the mismatch against stated monthly total, and flag a discrepancy.
- **Actual Result:** As captured in `discrepancy-run.txt`, the validator reported `{'validation': {'consistent': false}}` with a discrepancy delta of -1250.0 on field `total_monthly_income` (calculated: 9642.17, stated: 10892.17).

## 3. Supplier Chain Investigation
- **Perturbation:** Triggered synthesis with `--offline --simulate-timeout` runtime flags.
- **Hypothesis:** The coordinator will catch the logistics timeout gracefully, annotate the missing source as incomplete, and finish synthesis without crashing.
- **Actual Result:** As verified in `timeout-run.txt`, the run completed with exit code 0, reported `Sources unavailable: logistics unavailable (timeout)`, and logged `late_shipment_count [missing source: timeout reading logistics]` under `Incomplete`.
