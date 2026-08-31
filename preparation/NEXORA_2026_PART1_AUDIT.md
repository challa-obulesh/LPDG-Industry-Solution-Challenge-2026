# NEXORA 2026 - Part 1 Audit Status

## Challenge Understanding

The core NEXORA 2026 problem is to identify and rank the 15 gateways that should receive a field visit each week, given an operational capacity of 15 visits per week.

A gateway is a communication device that relays data from connected meters to the LPDG system. Gateway failures can lead to missing meter readings and operational/business impact.

## Part 1 Objective

Build a working prediction pipeline that produces the required `predictions.csv` for the challenge evaluation period.

Required output columns:

- `week_start`
- `rank`
- `gateway_id`
- `score`
- `reason`

Expected output:

- 8 target weeks
- 15 gateways per week
- 120 rows total
- ranks 1 through 15 for every week
- no duplicate gateway within a week
- no duplicate rank within a week

## Current Part 1 Approach

The current implementation uses a single LightGBM model rather than an ensemble or deep-learning system.

```text
Historical telemetry
        |
        v
Feature engineering
        |
        v
LightGBM
        |
        v
Risk score
        |
        v
Rank gateways
        |
        v
Select top 15
        |
        v
predictions.csv
```

The current design is intentionally kept relatively small and is intended to support the Data Science / Machine Learning specialization.

## Preliminary Audit

| Check | Current Status |
|---|---|
| Selects 15 gateways per week | Implemented using `head(15)` |
| Operational alignment | Uses the challenge visit/missed-problem cost framing |
| Challenge datasets | Telemetry, meter-read information, gateway master are used |
| Time-aware features | Feature window is restricted to telemetry before the prediction Monday |
| Output target | 8 weeks x 15 gateways intended |
| Required columns | All five required columns are generated |
| Baseline comparison | Backtesting compares the LightGBM approach with the provided 3-sigma baseline |
| Model complexity | Single LightGBM model; no unnecessary ensemble/deep-learning stack |
| Submission validator | Final execution still required |
| Duplicate/rank checks | Final execution still required |
| Reproducibility | CLI paths and requirements are provided |
| Documentation | `DECISIONS.md` and `AI-USAGE.md` still need finalization |
| Repository process | Final commits and submission preparation still required |

## Important Audit Findings

### 1. Target construction needs careful review

The current target is derived from meter-read success information. Initial diagnostics showed a large change in the number of flagged gateways during the first few labelled weeks. This needs to be understood before changing the target or threshold.

### 2. Feature importance needs investigation

The current model gives high importance to telemetry-coverage features such as:

- `hours_of_baseline_data`
- `hours_of_recent_data`

These should not be removed automatically. An ablation test should determine whether they provide a legitimate operational signal or act mainly as a data-quality proxy.

### 3. Unseen-gateway evaluation

An unseen-gateway evaluation has already been performed. The current result showed the same total cost for the model and baseline on that small evaluation set. This should be reported as a robustness test rather than interpreted as proof that the model has failed or succeeded generally.

## Current Verdict

The Part 1 technical approach is fundamentally sound based on the current implementation review.

The LightGBM model should **not** be removed simply because the provided baseline is simpler. The important question is whether the ML approach can be justified through target definition, leakage prevention, time-aware validation, business-cost evaluation, and robustness testing.

## Next Audit Steps

Do not change or retrain the model yet.

1. Run `validate_submission.py` on the current `predictions.csv`.
2. Confirm exactly 120 rows and 8 weeks.
3. Confirm 15 rows per week.
4. Confirm ranks 1-15 for every week.
5. Confirm no duplicate gateway/week combinations.
6. Confirm no duplicate ranks within a week.
7. Audit target construction.
8. Audit all features for future-data leakage.
9. Compare LightGBM and the 3-sigma baseline on time-based holdouts.
10. Run feature ablation where needed.
11. Only then decide what additional work belongs in Part 2.

## Part 2 Direction

The current intended specialization is **Data Science + Machine Learning**.

Part 2 should demonstrate depth rather than breadth. The focus should be on:

- defining what "needs a visit" means using the available evidence
- meaningful feature engineering
- avoiding future leakage
- time-based validation
- unseen-gateway robustness
- comparison against the provided baseline
- business-cost evaluation rather than accuracy alone
- clear explanation of assumptions, limitations, and decisions

## Working Principle

> First make Part 1 reliably valid. Then use Part 2 to demonstrate why the Data Science / Machine Learning approach is justified and how it improves or informs the gateway-visit decision.

No challenge dataset is included in this documentation.
