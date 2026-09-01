# IT Incident SLA Breach Prediction

Predicting which IT support tickets will miss their resolution target — using only the information a service desk actually has at the moment a ticket is created.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/anggitoardiansyah/IT-Incident-SLA-Prediction/blob/main/sla_breach_prediction_colab.ipynb)

---

## The problem

A service desk normally learns about an SLA breach after it has happened. By then the user has waited too long and the monthly compliance report is already damaged. The information exists to do better: every ticket arrives with a priority, a category, a reporting channel, and a timestamp. The question is whether that intake-time information carries enough signal to flag the tickets heading for trouble.

If it does, the desk can escalate or add a second responder while the outcome can still change.

---

## The headline result

This dataset has a trap in it, and most published work on it falls in.

It contains fields that only exist because a ticket already finished — `resolved_at`, `closed_code`, the final `reassignment_count`, the terminal `incident_state`. Feed those to a model and it scores beautifully. That is not a prediction. It is a lookup of the answer.

Every column was tested against one question:

> Could a service desk agent read this value at 09:00 on a ticket that opened at 09:00?

Anything that failed was dropped. The notebook then deliberately trains a second model **on the leaking fields alone**, so the inflated score sits directly next to the honest one:

| Model | ROC-AUC |
|---|---|
| Given post-resolution fields | **0.931** |
| Given intake-time fields only | **0.793** |

That 0.138 gap is the difference between prediction and hindsight. The honest number is the one this project reports everywhere else.

---

## Dataset

**UCI — Incident Management Process Enriched Event Log**

Anonymised audit records from a ServiceNow instance at an IT company, covering tickets opened between February 2016 and February 2017.

| | |
|---|---|
| Event rows | 141,712 |
| Incidents | 24,918 |
| Attributes | 36 |
| Grain | one row per state change, **not** per ticket |
| Source | [archive.ics.uci.edu/dataset/498](https://archive.ics.uci.edu/dataset/498/incident+management+process+enriched+event+log) |

The grain matters. Incidents average 5.7 events each and the worst one has 58. Training on the raw file would weight tickets by how much trouble they caused, purely because trouble generates rows.

---

## Method

**1. Collapse the event log to one incident per row.**
Features come from the *first* event, the closest available snapshot to intake. The label comes from the *last* event, the settled outcome. 24,918 incidents drop to **23,362** after removing tickets that were never resolved, had impossible timestamps, or carried no SLA outcome.

**2. Define the target two ways.**
The recorded label is ServiceNow's own `made_sla` verdict. The derived label is resolution time against a per-priority threshold. The two agree on **81.2%** of tickets — close enough to trust the recorded label, far enough apart to be worth noting that "breached" is partly an administrative judgment, not purely a stopwatch reading.

**3. Audit for leakage.** Every column classified keep / borderline / leak, with the reasoning written out in the notebook.

**4. Engineer intake-time features.** Hour, day of week, weekend and out-of-hours flags, plus queue pressure — how many other tickets arrived in the same hour (median 32, max 97).

**5. Split chronologically.** Train on earlier tickets, test on later ones. A random split would let the model learn from the future of its own test period.

**6. Model and evaluate on PR-AUC**, because the target is imbalanced and accuracy would reward predicting "no breach" every time.

**7. Choose a threshold from cost**, not from the 0.5 default.

**8. Explain the model** with permutation importance on held-out data, followed by a direction check on the strongest feature.

---

## Results

Overall breach rate across the cleaned data is **38.2%** (8,930 of 23,362 tickets).

| Model | ROC-AUC | PR-AUC |
|---|---|---|
| Logistic regression | 0.744 | 0.464 |
| Random forest | 0.789 | 0.550 |
| **Gradient boosting** | **0.793** | **0.558** |
| *Test-set base rate* | — | *0.236* |

Gradient boosting reaches 2.4x the base rate on PR-AUC. Modest, but it is real signal available at minute zero.

### Risk bands hold up

Scoring the held-out period and bucketing by predicted probability:

| Band | Tickets | Actual breach rate |
|---|---|---|
| Low | 2,766 | 10.1% |
| Medium | 1,368 | 36.5% |
| High | 375 | 50.1% |
| Critical | 164 | **82.3%** |

Against a 23.6% base rate, the Critical band carries a **3.5x lift**. Those 164 tickets are 3.5% of volume and four in five of them really did breach. This is the output a shift lead could actually use.

### The strongest predictor is not what you would guess

| Feature | Permutation importance |
|---|---|
| `assignment_group` | 0.182 |
| `priority` | 0.060 |
| `urgency` | 0.020 |
| `tickets_same_hour` | 0.020 |
| `u_symptom` | 0.019 |

Which team receives a ticket predicts breach roughly **three times better than how urgent the ticket is**. The direction check confirms it is not an artefact — breach rates by team range from 11.4% (Group 64) to 88.2% (Group 9), while nearly half of all tickets funnel into a single group that breaches 27.5% of the time.

That reframes the problem. This looks less like a ticket-difficulty issue and more like a team capacity and routing issue, which is an operational finding the model was not asked for but produced anyway.

---

## Two results that did not come out cleanly

Reported because they change how the numbers should be read.

**The cost-optimal threshold is not operationally usable.** With an assumed 10:1 penalty for a missed breach versus a false alarm, the search lands on a threshold of 0.08. That catches 97.5% of breaches at 29.1% precision — but it flags **79.1% of all tickets**. "Escalate four out of five tickets" is not triage, it is the status quo with extra steps. Either the 10:1 assumption is too aggressive for a desk with finite agents, or the model is not discriminative enough to support it. The risk-band table above is the more honest operational recommendation.

**Train and test are not like-for-like.** The chronological split puts the boundary at 11 May 2016, but the data is heavily front-loaded: 80% of tickets fall in roughly ten weeks, while the remaining 20% stretch across nine months. Breach rate drops from 41.9% in train to 23.6% in test. Some of the performance gap is that distribution shift rather than genuine difficulty. The chronological split is still the right choice, because it is what deployment looks like, but the test score should be read as a lower bound under changing conditions rather than a clean estimate.

---

## Recommended actions

| Risk band | Action | Owner |
|---|---|---|
| Critical (164 tickets, 82% breach) | Escalate at intake, assign a second responder | Shift lead |
| High (375 tickets, 50% breach) | Flag in the queue view, review at standup | Team lead |
| Medium / Low | Monitor only | — |

Separately, and independently of the model: the spread in breach rate across assignment groups is wide enough to justify a capacity review of the worst-performing teams.

---

## Repository

```
IT-Incident-SLA-Prediction/
├── sla_breach_prediction_colab.ipynb   ← start here
├── sla_risk_scores.csv                 ← 4,673 scored tickets
├── requirements.txt
├── README.md
└── plots/
    ├── 01_breach_rates.png
    ├── 02_resolution_time.png
    ├── 03_model_curves.png
    ├── 04_threshold.png
    └── 05_importance.png
```

The dataset CSV is not committed. It belongs to UCI and the notebook downloads it automatically.

---

## Running it

**Colab (no setup).** Click the badge at the top. The first cells install dependencies and fetch the data.

**Locally.**

```bash
git clone https://github.com/anggitoardiansyah/IT-Incident-SLA-Prediction
cd IT-Incident-SLA-Prediction
pip install -r requirements.txt
jupyter notebook sla_breach_prediction_colab.ipynb
```

---

## Limitations

- One company, one twelve-month window. The model would need retraining anywhere else.
- The first audit event approximates intake but is written minutes after it, so a field like `assignment_group` may already be populated.
- The derived SLA thresholds and the 10:1 cost ratio are both assumptions, and the notebook shows how sensitive the threshold is to the latter.
- Uneven data density across the period makes the chronological test set an imperfect comparison.
- Acting on a flag changes the outcome it predicted, so a deployed version degrades unless retrained on post-intervention data.

---

## Tools

| | |
|---|---|
| `pandas`, `numpy` | event log collapse, feature engineering |
| `scikit-learn` | pipelines, logistic regression, random forest, gradient boosting, permutation importance |
| `matplotlib`, `seaborn` | figures |

---

## Author

**Anggito Ardiansyah**
[GitHub](https://github.com/anggitoardiansyah)

---

*Dataset: Amaral, C., Fantinato, M., & Peres, S. (2018). Incident management process enriched event log. UCI Machine Learning Repository. https://doi.org/10.24432/C57S4H*
