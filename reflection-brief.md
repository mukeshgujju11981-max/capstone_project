# Capstone Reflection Brief

## System 1 - Validated, Routed Insurance Policy Extraction
- **Tests & Calibration:** 45 passed, 3 skipped. The `calibration_report.py` generated an overall Brier score of 0.291 (slices included auto premium_amount, home deductible, and umbrella exclusions).
- **Observability & Routing:** The routing mechanism dispatches clear policies to `AUTO_APPROVE` and routes unwell-formed or low-confidence records directly to `HUMAN_REVIEW` (as reflected in `routing_decisions.json`).

## System 2 - Resilient Mortgage Document Extraction
- **Tests & Validation:** 25 passed. The mathematical consistency validator verifies both calculated and stated values within tolerances.
- **Discrepancy Handling:** Fixture `income_sum_mismatch.txt` evaluated Marcus T. Hollingwworth's income sum ($9,642.17) against stated total ($10,892.17), flagging `{"validation": {"consistent": false}}` with `delta: -1250.0`.

## System 3 - Supply Chain Risk Investigation
- **Corroboration & Contested Data:** 34 passed. Corroborated lead time (12.0 days) and defect rate (180-190 ppm) across sources, while flagging on-time delivery rate as Contested (95% vs. 78%) with an escalation warning.
- **Graceful Degradation:** Under `--simulate-timeout`, the coordinator gracefully handled the failure, marking `logistics unavailable (timeout)` and classifying `late_shipment_count` under `Incomplete`.
