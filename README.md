# IT Incident SLA Breach Prediction

Predicting which IT support tickets will miss their resolution target — using only the information a service desk actually has at the moment a ticket is created.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/TODO_USERNAME/TODO_REPO/blob/main/sla_breach_prediction_colab.ipynb)

---

## The problem

A service desk normally learns about an SLA breach after it has already happened. By then the user has waited too long and the monthly compliance report is already damaged. The information exists to do better: every ticket arrives with a priority, a category, a reporting channel, and a timestamp. The question is whether that intake-time information carries enough signal to flag the tickets that are heading for trouble.

If it does, the desk can escalate or add a second responder while the outcome can still change.

---

## Why this project is not just another classifier

This dataset has a trap in it, and most published work on it falls in.

It contains fields that only exist because a ticket already finished — `resolved_at`, `closed_code`, the final `reassignment_count`, the terminal `incident_state`. Feed those to a model and it reports accuracy above 90%. That is not a prediction. It is a lookup of the answer.

Every column here was tested against one question:

> Could a service desk agent read this value at 09:00 on a ticket that opened at 09:00?

Anything that failed was dropped. The notebook then deliberately trains a second model **on the leaking fields alone**, so the inflated score sits directly next to the honest one. The distance between them is the point of the project.

| | ROC-AUC |
|---|---|
| Model given post-resolution fields | `TODO` |
| Model given intake-time fields only | `TODO` |

---

## Dataset

**UCI — Incident Management Process Enriched Event Log**

Anonymised audit records from a ServiceNow instance at an IT company, 2016.

| | |
|---|---|
| Rows | ~141,700 events |
| Incidents | ~24,900 |
| Attributes | 36 |
| Grain | one row per state change, **not** per ticket |
| Source | [archive.ics.uci.edu/dataset/498](https://archive.ics.uci.edu/dataset/498/incident+management+process+enriched+event+log) |

The grain matters. A ticket that bounced between four teams contributes many more rows than one resolved on first contact, so training on the raw file would weight tickets by how much trouble they caused.

---

## Method

**1. Collapse the event log to one incident per row.**
Features come from the *first* event, the closest available snapshot to intake. The label comes from the *last* event, the settled outcome. The first audit record is written moments after the ticket opens rather than at the instant it opens, which is an approximation, and the notebook says so rather than glossing it.

**2. Define the target two ways.**
The recorded label is ServiceNow's own `made_sla` verdict — the number that reaches the monthly report. The derived label is resolution time against a per-priority threshold. Recorded is primary; derived is a robustness check. Where they disagree is itself informative about how compliance gets recorded versus what actually happened.

**3. Audit for leakage.**
Every column classified keep / borderline / leak, with the reasoning written out.

**4. Engineer intake-time features.**
Hour, day of week, weekend and out-of-hours flags, plus queue pressure — how many other tickets arrived in the same hour. That last one is knowable at intake and often predicts delay better than anything on the ticket itself.

**5. Split chronologically, not randomly.**
These tickets are time-ordered. A random split lets the model learn from the future of its own test period, which is not how it would be deployed.

**6. Model and evaluate.**
Logistic regression baseline, then random forest and gradient boosting. Reported on PR-AUC against the base rate, because accuracy is meaningless on an imbalanced target.

**7. Choose a threshold from cost, not from habit.**
0.5 is a default, not a decision. A missed breach costs more than a false alarm, but false alarms are not free — enough of them and the flag gets ignored. The notebook assumes a ratio, states it, and lets it pick the threshold.

**8. Explain the model.**
Permutation importance on held-out data, followed by a direction check: if a feature matters but its pattern makes no operational sense, that is either a surviving leak or a data quirk worth writing about.

---

## Results

> **Not yet filled in.** Run the notebook, then replace every `TODO` below. Delete this blockquote when you do.

| Metric | Value |
|---|---|
| Incidents after cleaning | `TODO` |
| Breach rate (base rate) | `TODO` |
| Best model | `TODO` |
| PR-AUC | `TODO` |
| ROC-AUC | `TODO` |
| Cost-optimal threshold | `TODO` |
| Breaches caught at that threshold | `TODO` |
| Share of volume flagged | `TODO` |

**Findings**

1. `TODO — where breaches concentrate`
2. `TODO — the honest score against the base rate`
3. `TODO — the leaky score, and what the gap means`
4. `TODO — the strongest intake-time signal and its likely operational cause`

**If the honest model is only modestly above the base rate, that is the result.** Report it. Intake-time fields may simply not carry much information about what happens over the following days. Showing that clearly, with the evidence, is a stronger piece of work than quietly reintroducing a leaked feature to reach a nicer number.

---

## Recommended actions

| Risk band | Action | Owner |
|---|---|---|
| Critical | Escalate at intake, assign a second responder | Shift lead |
| High | Flag in the queue view, review at standup | Team lead |
| Medium | Monitor only | — |
| Low | No action | — |

---

## Repository

```
sla-breach-prediction/
├── sla_breach_prediction_colab.ipynb   ← start here
├── sla_risk_scores.csv                 ← scored tickets, generated by the notebook
├── requirements.txt
├── README.md
└── plots/
    ├── 01_breach_rates.png
    ├── 02_resolution_time.png
    ├── 03_model_curves.png
    ├── 04_threshold.png
    └── 05_importance.png
```

The dataset CSV is not committed. It is ~30 MB and belongs to UCI, so download it yourself.

---

## Running it

**Colab (no setup).** Click the badge at the top. The first cells install dependencies and fetch the data.

**Locally.**

```bash
git clone https://github.com/TODO_USERNAME/TODO_REPO
cd TODO_REPO
pip install -r requirements.txt

# Download the dataset and unzip it into the project root:
# https://archive.ics.uci.edu/dataset/498/incident+management+process+enriched+event+log

jupyter notebook sla_breach_prediction_colab.ipynb
```

---

## Limitations

- One company, one period in 2016. The model would need retraining anywhere else.
- The first audit event approximates intake but is not intake itself.
- The derived SLA thresholds are an assumption, not a contract.
- The cost ratio behind the threshold is an assumption too, and the notebook shows how sensitive the answer is to it.
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

`TODO — your name`
[LinkedIn](TODO) · [GitHub](TODO)

---

*Dataset: Amaral, C., Fantinato, M., & Peres, S. (2018). Incident management process enriched event log. UCI Machine Learning Repository. https://doi.org/10.24432/C57S4H — confirm the current licence terms on the dataset page before redistributing anything derived from it.*
