# Reflection Brief — Evaluation and Observability Capstone

**Name:** Mukesh Gujju
**Date:** September 28, 2026

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux-6.6.97+-x86_64-with-glibc2.36 |
| Python version | Python 3.13.0 |
| Date run | 2026-09-27 / 2026-09-28 |
| Ran any system live? (which) | No (used local recording replay, `--mode auto`, and `--offline` modes) |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped |
| Routing output file | `capstone-submission/01-policy-pipeline/routing_decisions.json` |
| auto_approve / human_review / spot_check counts | 2 / 1 / 0 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> From `pipeline-run.txt`:
> ```text
> Processing data/policies/policy_auto_003.txt...
>   Extraction completed: ESCALATED (missing required deductible)
>   Reviewer agreement: DISAGREEMENT (field: deductible)
>   Routing decision: HUMAN_REVIEW
> ```
> The system made exactly **1** API call. When a field is missing from the source document, re-prompting or retrying cannot recover it. Retrying burns tokens, adds latency, and forces the model toward hallucinating a plausible replacement value instead of safely flagging the omission.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> From `routing_decisions.json`:
> ```json
> {
>   "policy_id": "policy_auto_003",
>   "status": "HUMAN_REVIEW",
>   "confidence": 0.52,
>   "reviewer_disagreement": true,
>   "review_confidence": 0.45
> }
> ```
> Both **reviewer disagreement** (`reviewer_disagreement: true`) and **low model confidence** (0.52 < 0.85 threshold) routed the file. If the system had trusted single-pass self-reported confidence without an independent reviewer, an overconfident hallucination on an incomplete document could have been silently auto-approved.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> From `calibration-report.txt`:
> ```text
> umbrella   exclusions      n=2 conf=0.93 acc=0.00 brier=0.865
> OVERALL brier=0.291
> ```
> The aggregate Brier score of **0.291** obscures severe local miscalibration. On `umbrella` policy `exclusions`, the model had 0.93 average confidence but achieved 0.00 accuracy ($brier=0.865$). Slicing exposes field-specific risks that an overall score smooths away.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed |
| Document run | `fixtures/documents/income_missing_bonus.txt` |
| Classified type | Paystub / Income Document (borrower: Daniel R. Whitfield) |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> From `discrepancy-run.txt`:
> ```json
> {
>   "consistent": false,
>   "discrepancies": [
>     {
>       "field": "total_monthly_income",
>       "calculated": 9642.17,
>       "stated": 10892.17,
>       "delta": -1250.0
>     }
>   ]
> }
> ```
> Tool calling guarantees **syntactic and schema validity** (correct types, keys, and valid JSON structure), but cannot enforce **semantic or mathematical truth**. Tool calling cannot catch when stated total income diverges from the sum of its items. Conversely, a post-extraction validator cannot catch a malformed JSON syntax error if parsing fails before validation starts.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> From `extract-run.txt` (Daniel R. Whitfield):
> ```json
> "income": {
>   "base_monthly": 5673.08,
>   "bonus_monthly": null,
>   "bonus_ytd": null,
>   "commission_monthly": null,
>   "overtime_monthly": null,
>   "other_monthly": null,
>   "stated_monthly_total": null
> }
> ```
> Emitting `null` signals that the value was genuinely absent from the raw document, avoiding fabrication. This is explicitly enabled in the Pydantic schema by defining the fields as optional unions with a `None` default (e.g., `bonus_monthly: float | None = None`).

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> From `fixtures/documents/appraisal_informal_sqft.txt` and `tests/test_us03_prompts.py`:
> Source: `"approx 2,400 sq ft"` → Extracted: `"gross_living_area": 2400`.
> Normalizing at extraction time grounds the numerical transformation while the full linguistic context is present in the LLM's active window, preventing downstream brittle regex parsing on varied colloquial phrasing.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed |
| Briefing file | `capstone-submission/03-supply-chain/briefing.md` |
| Section the conflict landed in | `## Contested` |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> From `investigation-run.txt` / `briefing.md`:
> ```text
> ### on_time_delivery_rate  _[2 sources, conflicting]_  ⚠️ ESCALATE
> - escalation: high-impact metric is contested across sources
> - Reported values by source:
>     - 95.0 percent — supplier_audit (as of 2026-04-10)
>     - 78.0 percent — logistics (as of 2026-04-05)
> ```
> Preserving both sources shows a direct contradiction between internal logistics telemetry (78.0%) and the vendor's self-audit (95.0%). A forced average (86.5%) would conceal a potential audit discrepancy and obscure supply risk.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> From `timeout-run.txt`:
> ```text
> > Sources unavailable: logistics unavailable (timeout)
> ...
> ## Incomplete
> ### late_shipment_count  _[missing source: timeout reading logistics]_
> - missing source: timeout reading logistics
> ```
> "Unreachable" records an infrastructure failure under `Incomplete` with the exact cause (`timeout reading logistics`), whereas "nothing to report" means the source was queried successfully but returned no metrics. The run completes because each source query is isolated with error boundaries and timeouts.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> From `briefing.md`:
> - `average_lead_time_days: 12.0 days — logistics (as of 2026-04-05)`
> - `average_lead_time_days: 12.0 days — supplier_audit (as of 2026-04-10)`
> Dates provide a timeline. Metrics often shift naturally over time; attaching provenance dates ensures the agent interprets differing temporal observations as chronological updates rather than factual disagreements.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> In **System 2 (`discrepancy-run.txt`)**: For Marcus T. Hollingsworth, the model extracted clean JSON with a stated total of `$10,892.17`. A naive system would have accepted it; the deterministic mathematical consistency validator summed the components ($9,642.17) and flagged the `-$1,250.00` discrepancy.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> In **System 1 (`calibration-report.txt`)**: On the `umbrella × exclusions` slice, the model reported **0.93** confidence but had **0.00** accuracy ($brier=0.865$). Relying solely on model confidence would have auto-approved broken extractions. The independent reviewer caught the mismatch and escalated it to human review.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> In an accounts payable pipeline parsing incoming vendor invoices:
> 1. **Pattern:** **Validated retry with escalation**. The system checks line items against invoice totals ($\sum \text{items} + \text{tax} = \text{total}$). If calculations fail, it retries with the exact difference; if required terms are missing, it immediately escalates to human review.
> 2. **Instrumentation:** Monitor **discrepancy rate sliced by vendor**, **first-attempt vs retry-success rates**, and **Brier calibration scores** on extracted totals to detect shifts in model accuracy.
