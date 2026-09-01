# NEXORA 2026 — Gateway Visit Prioritisation
## Part 1 Solution & Data Science / Machine Learning Documentation

## 1. Challenge Overview

The LPDG Innovation Hub Challenge is an engineering and problem-solving challenge based on a gateway network that carries meter readings.

The operational problem is: every week, only 15 gateways can be selected for engineer site visits. The objective is to identify and rank the gateways that should receive those visits.

The challenge provides historical gateway telemetry, gateway information, historical field visits, weekly meter-read results and an engineer review dataset. It also provides a working `baseline_3sigma.py` implementation.

LPDG deliberately does not define exactly what "needs a visit" means. Participants are expected to make and defend that decision.

## 2. Business Problem

The gateway is an important point in the network because it relays information from many meters.

The operational team has a limited engineering capacity:

```text
Maximum site visits per week = 15
```

This is a hard limit.

The operational costs are asymmetric:

| Situation | Cost |
|---|---:|
| Engineer visits but nothing is wrong | €380 |
| Broken gateway is left unattended for one week | €600 |
| Broken gateway remains unattended another week | Additional €600 |

Therefore, this is not simply a classification problem. The real operational question is:

> Which 15 gateways should engineers visit this week?

## 3. Challenge Inputs

Main supplied inputs include:

```text
data/
├── telemetry/
│   └── month=YYYY-MM/*.parquet
├── telemetry_sample_2025-08.csv
├── gateway_master.csv
├── field_visits.csv
├── meter_read_success.csv
└── engineer_review_2026-02.xlsx
```

The challenge also provides:

```text
baseline_3sigma.py
validate_submission.py
Data Dictionary
```

The dataset must not be published to GitHub or elsewhere.

## 4. Input 1 — Gateway Telemetry

The main telemetry dataset contains hourly observations for gateways. LPDG describes it as one row per gateway per hour with 57 columns. The telemetry covers August 2025 through March 2026 and is divided into monthly Parquet partitions.

Representative telemetry fields include:

```text
gateway_id
ts_utc
rx_nr_pkts
rx_crc_bad
tx_success
tx_busy
number_of_messages
avg_idletime
avg_load1
avg_memfree
avg_uptime
reboot_cnt
reboot_duration_sec
...
```

Telemetry answers:

> What has this gateway been doing?

## 5. Input 2 — Gateway Master

`gateway_master.csv` provides information about the gateways themselves, including the gateway list and location-related information.

Conceptually:

```text
Telemetry
= Gateway behaviour

Gateway master
= Gateway information
```

## 6. Input 3 — Meter Read Success

`meter_read_success.csv` contains weekly gateway-level meter-reading information.

Important fields include:

```text
gateway_id
week_start
meters_expected
meters_read
```

This dataset is particularly important for the ML approach because it provides information related to the outcome being modelled.

## 7. Input 4 — Historical Field Visits

`field_visits.csv` contains historic work orders. Important fields include:

```text
visit_id
gateway_id
requested_on
visited_on
reason_reported
outcome
parts_replaced
technician_hours
```

This provides historical operational information about gateway visits.

## 8. Input 5 — Engineer Review

`engineer_review_2026-02.xlsx` contains engineer reviews of selected gateways. It includes fields such as:

```text
gateway_id
standort
Kategorie
reviewed_on
reviewer
Bemerkung
```

`Kategorie` contains values such as `Normal` and `Schlecht`.

## 9. LPDG Baseline Model

LPDG provides:

```text
baseline_3sigma.py
```

This is a working baseline. Participants are not required to build an ML model unless they choose the Machine Learning discipline.

Conceptually:

```text
Historical gateway history
        ↓
Statistical baseline
        ↓
3-sigma anomaly detection
        ↓
Gateway abnormality score
        ↓
Ranking
        ↓
Top 15
```

The baseline is the reference approach against which an ML solution can be evaluated.

## 10. Baseline vs Our Model

### LPDG Baseline

```text
Historical telemetry
        ↓
Statistical baseline
        ↓
3-sigma anomaly detection
        ↓
Gateway abnormality score
        ↓
Ranking
        ↓
Top 15
```

### Our Current Model

```text
Historical telemetry
        +
Gateway information
        +
Meter-read information
        ↓
Feature engineering
        ↓
LightGBM
        ↓
Gateway risk score
        ↓
Ranking
        ↓
Top 15
```

The key difference is that the baseline uses predefined statistical anomaly logic, while our model learns a relationship from historical data using LightGBM.

## 11. Our Part 1 Architecture

```text
                RAW DATA
                    │
       ┌────────────┼─────────────┐
       │            │             │
       ▼            ▼             ▼
   Telemetry   Meter Success   Gateway Master
       │            │             │
       └────────────┼─────────────┘
                    ▼
            Feature Engineering
                    │
                    ▼
               LightGBM
                    │
                    ▼
              Risk Score
                    │
                    ▼
              Gateway Ranking
                    │
                    ▼
              Select Top 15
                    │
                    ▼
             predictions.csv
```

## 12. Feature Engineering

Raw hourly telemetry is transformed into gateway-level features before modelling.

Conceptually:

```text
Hourly telemetry
       ↓
Group by gateway
       ↓
Historical window
       ↓
Calculate statistics
       ↓
Gateway-level feature vector
```

Current important feature families include offline behaviour, disconnection behaviour, reboot behaviour, RSSI/signal behaviour, recent activity, baseline activity and data availability.

The exact feature definitions should always be taken from the current `features.py` implementation.

## 13. Time-Aware Feature Construction

For a prediction week beginning on Monday, the model must use information available before that Monday.

```text
                 Prediction Monday
                       │
                       ▼
───────────────────────│────────────────────
       ALLOWED         │      NOT ALLOWED
       HISTORY         │      FUTURE DATA
───────────────────────│────────────────────
       telemetry       │      future telemetry
       before Monday   │      future outcomes
                       │
```

This prevents future information from entering the features and is essential for a realistic forecasting evaluation.

## 14. Target Construction

Our current ML approach derives a gateway-level problem label from meter-read data:

```text
meter_read_success
        ↓
problem_gateway_ids()
        ↓
problem = 0 / 1
```

Conceptually:

```text
Healthy / acceptable outcome → 0
Problem outcome              → 1
```

This is our modelling decision, not an LPDG-prescribed definition. LPDG intentionally leaves the meaning of "needs a visit" to the participant. The target definition therefore needs to be explicitly justified in the final DS/ML submission.

## 15. LightGBM Model

Our current model uses a single LightGBM model rather than an unnecessary ensemble or deep-learning pipeline.

```text
Features
   ↓
LightGBM
   ↓
Predicted risk
```

The purpose is not to maximize model complexity. The purpose is to produce a useful operational ranking that can be explained and validated.

## 16. Training Strategy

The training process uses a time-based split rather than randomly splitting individual rows.

```text
Earlier weeks
────────────────────
        ↓
     Training

Later weeks
────────────────────
        ↓
     Holdout
```

This simulates the real operational situation: train using earlier information and predict a future period.

## 17. Prediction Process

```text
New prediction week
        ↓
Historical data before cutoff
        ↓
Feature generation
        ↓
LightGBM
        ↓
Risk score for each gateway
        ↓
Sort descending
        ↓
Select first 15
```

The operational capacity is 15 gateways per week.

## 18. Ranking

The model produces a risk score for each gateway. Gateways are sorted from highest to lowest score.

Example:

```text
Gateway     Score
-----------------
G001        0.92
G019        0.87
G104        0.84
G087        0.79
...
```

The highest-risk 15 gateways become the recommended visit list.

The score scale itself is not important; the ranking order is what matters operationally.

## 19. Final Output — predictions.csv

The required output columns are:

```text
week_start
rank
gateway_id
score
reason
```

The required submission covers 8 weeks with 15 gateways per week:

```text
8 weeks × 15 gateways = 120 rows
```

Prediction weeks:

```text
2026-02-02
2026-02-09
2026-02-16
2026-02-23
2026-03-02
2026-03-09
2026-03-16
2026-03-23
```

Example structure:

```text
week_start | rank | gateway_id | score | reason
2026-02-02 | 1    | GW001      | 0.92  | Elevated recent instability and repeated gateway interruptions.
2026-02-02 | 2    | GW019      | 0.87  | Recent gateway behaviour indicates increased operational risk.
...
2026-02-02 | 15   | GW210      | 0.42  | Recent abnormal behaviour increases visit priority.
```

The actual reasons must come from the implementation and should remain understandable to an operations manager. LPDG limits each reason to 300 characters.

## 20. Part 1 Validation

Run:

```bash
python validate_submission.py predictions.csv
```

Structural validation should confirm:

```text
120 rows
8 weeks
15 rows/week
rank 1–15
no duplicate gateway/week
required columns
valid score
valid reason
```

Structural validation and model quality are different checks.

## 21. Baseline Comparison

For the Machine Learning discipline, LPDG states that the ML approach should beat `baseline_3sigma.py` on total cost rather than accuracy.

The comparison should therefore include:

```text
                     Baseline      LightGBM
                     --------      --------
False visit cost       €X             €Y
Missed gateway cost    €X             €Y
Total cost             €X             €Y
```

The primary operational metric is:

```text
TOTAL OPERATIONAL COST
```

Accuracy, precision, recall and F1 can be useful diagnostic metrics, but they are not the primary business objective.

## 22. Business Cost Model

The operational costs are asymmetric:

```text
Visit healthy gateway
        ↓
      -€380

Miss broken gateway
        ↓
      -€600/week
```

Therefore the ML problem should ultimately be treated as a decision-ranking problem rather than only a binary classification problem.

## 23. Backtesting

Our implementation contains a backtesting process to compare the ML approach against the 3-sigma baseline.

```text
Historical labelled weeks
          ↓
Train LightGBM
          ↓
Predict later weeks
          ↓
Select Top 15
          ↓
Calculate operational cost
          ↓
Compare with baseline
```

The purpose is to simulate future predictions using information that would have been available at the time.

## 24. Unseen-Gateway Evaluation

An unseen-gateway evaluation has already been performed.

```text
Training gateways: 280
Unseen gateways:   19
```

Observed result:

```text
LightGBM:  €9,120
Baseline:  €9,120
```

This should be reported honestly as a small robustness evaluation. It does not prove that the approaches are equivalent because the evaluation contains a small number of unseen gateways and actual problem cases.

## 25. Feature Importance

Current LightGBM feature importance highlighted features including:

```text
hours_of_baseline_data
hours_of_recent_data
rssi_bad_recent_rate
offline_duration...
disconnection_zscore
```

The first two deserve investigation because high importance may indicate either useful data-availability information or a data-quality proxy.

They should not be removed automatically. An ablation test should be used to understand their effect.

## 26. Target Investigation

The current diagnostic showed an unusual pattern in the number of labelled problem gateways during early weeks:

```text
2025-08-04    79
2025-08-11    73
2025-08-18    69
2025-08-25     1
2025-09-01     4
```

This does not automatically mean the labels are wrong. The underlying data should be investigated to understand the reason for the change before finalizing the target definition.

## 27. Data Leakage Audit

For every prediction week, all model inputs must be available before the prediction cutoff.

```text
                 Prediction Monday
                        │
                        ▼
              ──────────┼──────────
                 Allowed │ Forbidden
                         │
        historical data  │ future outcomes
        historical       │ future telemetry
        telemetry        │ future visits
                         │ future labels
```

For example, when predicting Monday 2026-02-02, information from later dates cannot be used to construct the prediction features.

## 28. Part 1 vs Part 2

### Part 1

Part 1 is primarily an operational submission check:

```text
Make the system run
        ↓
Generate valid predictions.csv
        ↓
15 gateways × 8 weeks
```

LPDG treats Part 1 as Pass/Fail.

### Part 2 — Data Science + Machine Learning

The selected focus is Data Science and Machine Learning.

The important questions are:

```text
Why this target?
Why these features?
Why LightGBM?
Why this validation?
Does it beat the baseline?
Where does it fail?
What happens to unseen gateways?
What happens to future weeks?
What happens when network behaviour changes?
```

## 29. Expected Final DS + ML Story

```text
LPDG Problem
     ↓
Only 15 gateway visits available per week
     ↓
Define what "needs a visit" means
     ↓
Construct a defensible target
     ↓
Use historical information available
before the prediction week
     ↓
Engineer gateway-level features
     ↓
Train LightGBM
     ↓
Predict gateway risk
     ↓
Rank gateways
     ↓
Select Top 15
     ↓
Calculate €380 / €600 operational cost
     ↓
Compare against 3-sigma baseline
     ↓
Test future weeks
     ↓
Test unseen gateways
     ↓
Test network changes
     ↓
Understand failure cases
```

## 30. What LPDG Should See

The project should not be presented simply as:

> I built a LightGBM model to predict gateway failures.

A stronger description is:

> I built a gateway visit-prioritisation system that uses historical gateway behaviour to rank the 15 gateways most deserving of limited field-engineering capacity, and I evaluate the ML ranking using the operational costs defined by the challenge.

This connects the ML work directly to the LPDG operational problem.

## 31. Current Final Architecture

```text
                         LPDG DATA
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
       ▼                     ▼                     ▼
   Telemetry          Meter Read Success    Gateway Master
       │                     │                     │
       │                     ▼                     │
       │                ML Target                 │
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             │
                             ▼
                   Feature Engineering
                             │
                             ▼
                         LightGBM
                             │
                             ▼
                       Risk Score
                             │
                             ▼
                       Gateway Ranking
                             │
                             ▼
                         Top 15
                             │
                             ▼
                     predictions.csv
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
        Format Validation             Model Evaluation
              │                             │
              ▼                             ▼
           PASS/FAIL              Baseline Comparison
                                            │
                           ┌────────────────┼───────────────┐
                           ▼                ▼               ▼
                       Cost Test      Future Weeks    Unseen Gateways
                           │                │               │
                           └────────────────┼───────────────┘
                                            ▼
                                    Final DS + ML
                                      Assessment
```

## 32. Remaining Audit Checklist

Before calling the solution final, verify:

```text
[ ] predictions.csv validator
[ ] 120-row confirmation
[ ] 15 rows/week
[ ] duplicate gateway/rank check
[ ] target definition
[ ] target distribution investigation
[ ] feature leakage audit
[ ] feature ablation
[ ] baseline vs LightGBM cost
[ ] future-week evaluation
[ ] unseen-gateway evaluation
[ ] network-change robustness
[ ] DECISIONS.md
[ ] AI-USAGE.md
```

## 33. Key Summary

The complete solution is:

```text
LPDG gateway data
        ↓
Historical telemetry + operational data
        ↓
Feature engineering
        ↓
LightGBM risk estimation
        ↓
Gateway ranking
        ↓
Top 15 gateways per week
        ↓
predictions.csv
        ↓
Operational-cost evaluation
        ↓
3-sigma baseline comparison
        ↓
Future / unseen-gateway robustness tests
```

The central objective is not model complexity. It is to demonstrate a technically correct, operationally meaningful and explainable method for deciding which 15 gateways deserve limited engineer capacity each week.
