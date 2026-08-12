# Prior Signals, Future Decline
## A leakage-controlled ranking system for content review prioritization

**Author:** Agmo  
**Lane:** Refresh / Content Opportunity Scoring — future decline prediction  
**Dataset:** FlyRank ML Internship pseudonymized warehouse release  
**Release:** `flyrank_pseudonymized_warehouse_release_v20260703`  
**Decision horizon:** 30-day future decline  
**Feature window:** 90 days before each decision date

---

## Abstract

This capstone asks whether information available before a decision date can rank content pages that are likely to experience a meaningful 30-day future decline. Using the pseudonymized FlyRank warehouse, I aggregate a 90-day pre-decision feature window and define decline from the following 30-day impression change. I compare a transparent stale-and-visible baseline with Logistic Regression and Random Forest models using leakage checks, future validation, an unseen-client validation audit, and a sealed May 2026 test period. On the sealed May test, the final Random Forest achieved Precision@20 of **0.95**, Precision@50 of **0.98**, average precision of **0.852**, and ROC-AUC of **0.751**, compared with **0.95**, **0.96**, **0.675**, and **0.500** for the baseline. The final output is a ranked, human-review queue with risk scores, reason codes, and action labels; it is intended as observational decision-support rather than causal evidence about search performance or refresh impact.

---

## 1. Introduction / Problem statement

Content teams rarely have unlimited editorial capacity. The practical question is therefore not simply whether a page is declining; it is:

> **Which pages should be reviewed first when editorial capacity is limited?**

The unit of analysis is **one content page at one decision date**.

The system produces a **decline-risk score, rank, reason code, action label, and priority band**. A human reviewer can use the queue to prioritize investigation for refresh, performance review, CTR/snippet review, search-visibility review, or monitoring.

A false positive consumes limited editorial capacity on a page that did not meet the defined future-decline outcome. A false negative leaves a genuinely declining page lower in the queue. Because the operational decision is a ranked review queue, **Precision@20** is the primary top-of-queue metric, while average precision and ROC-AUC describe broader ranking discrimination.

This is not an attempt to predict Google's algorithm. It is a decision-support exercise: learn whether observable pre-decision signals can improve the ordering of pages for human review.

---

## 2. Data

### Release

The analysis uses the fixed FlyRank warehouse snapshot:

`flyrank_pseudonymized_warehouse_release_v20260703`

The release is the pseudonymized warehouse snapshot exported on 2026-07-03. Daily facts run through 2026-06-30 because the freshest three days were deliberately excluded from the release.

### Tables

The analysis uses:

- `fact_content_daily_performance` — daily client × content performance.
- `dim_content` — content-level metadata.
- `dim_clients` — client-level metadata and history coverage.

The daily fact table contains the time-series signals needed to construct pre-decision windows and future outcomes. The content and client dimensions provide context and grouping keys.

### Decision windows

Four decision dates were constructed:

| Role | Decision date |
|---|---|
| Training | 2026-02-28 |
| Training | 2026-03-31 |
| Validation | 2026-04-30 |
| Sealed test | 2026-05-31 |

For each decision date:

- **Feature window:** prior 90 days.
- **Target window:** following 30 days.
- **Minimum feature impressions:** 300.
- **Minimum feature observations:** 60.
- **Minimum target observations:** 21.
- **Decline threshold:** 30%.

The resulting label populations showed substantial movement over time. Decline rates were approximately **14.8%** at the February decision date, **34.7%** in March, **55.1%** in April, and **67.4%** in the sealed May test. This changing base rate is one reason the evaluation reports the base rate alongside ranking metrics.

### Features

The final model uses eight pre-decision features:

1. feature-period impressions
2. feature-period clicks
3. feature-period CTR
4. feature-period average position
5. number of feature observations/days
6. impression momentum
7. click momentum
8. content age in days

These features are constructed from information available by the decision date.

### Deliberately excluded

The model excludes outcome-derived and product-decision fields, including:

- `trend_direction`
- `trend_pct`
- future-window impression/click/position/session fields
- `health_score`
- `priority_score`
- `action_type`

Pseudonymous identifiers such as `client_hash_id` and `content_hash_id` are retained only for grouping, joins, and traceability; they are not model features.

---

## 3. Methodology

### Target definition

The target is a binary future outcome:

> A page is positive when its future 30-day performance meets the predefined **30% decline** threshold, subject to the minimum target-observation requirement.

The important distinction is:

**prior 90-day information → future 30-day outcome**

This prevents the model from seeing the answer it is being asked to predict.

### Baseline

The Week-4 baseline is a transparent rule designed around stale, visible pages. It provides an intentionally simple benchmark that can be understood without ML.

The baseline is useful because a learned model should earn its additional complexity. If the model cannot improve ranking quality over a transparent rule, there is little operational reason to deploy the model.

### Candidate models

Two candidate models were evaluated:

- Logistic Regression — simple, linear, and interpretable.
- Random Forest — able to represent nonlinear relationships and interactions among the pre-decision signals.

The Random Forest was selected as the final model based on ranking performance.

### Validation design

The project uses future-oriented decision dates because the target is explicitly future-looking.

The workflow contains:

1. training on earlier decision dates;
2. validation on a later decision date;
3. an unseen-client validation audit;
4. a sealed future May test.

The unseen-client check matters because pages belonging to the same client can share patterns. A random page-level split could therefore make the task easier than the real operational setting.

### Leakage checks

The final feature set was explicitly checked against a forbidden-feature list. No final feature contains `future`, `target`, or `decline` semantics, and no product decision fields or pseudonymous IDs are supplied to the model.

The feature window ends at the decision date and does not overlap the future target window.

---

## 4. Results

### Base rate

The sealed May test has a decline base rate of **67.44%**. A high Precision@K therefore has to be interpreted alongside this base rate.

### Sealed May test

| Method | Precision@20 | Precision@50 | Average Precision | ROC-AUC | Lift@20 |
|---|---:|---:|---:|---:|---:|
| Week-4 baseline | 0.95 | 0.96 | 0.675 | 0.500 | 1.409 |
| Final Random Forest | **0.95** | **0.98** | **0.852** | **0.751** | 1.409 |

The most important result is not that the model wins every top-K metric. Both methods achieve **0.95 Precision@20** on the sealed test.

The stronger difference appears in the broader ranking:

- Precision@50 increases from **0.96 to 0.98**.
- Average precision increases from **0.675 to 0.852**.
- ROC-AUC increases from **0.500 to 0.751**.

This suggests that the Random Forest provides more useful ordering information beyond the very first 20 candidates.

### Unseen-client validation

The unseen-client validation result for the Random Forest was:

| Metric | Result |
|---|---:|
| Precision@20 | 0.95 |
| Precision@50 | 0.88 |
| Average Precision | 0.833 |
| ROC-AUC | 0.842 |
| Lift@20 | 1.792 |

This check provides additional evidence that the model can discriminate on clients withheld from the fitting process, although it remains one validation period rather than a guarantee of performance on future releases.

---

## 5. Interpretation

The final feature set represents four broad signal families:

- **Exposure:** impressions.
- **Clicks/engagement capture:** clicks and CTR.
- **Search position:** average position.
- **Momentum and lifecycle:** impression momentum, click momentum, observation count, and content age.

The model therefore has access to both current state and recent movement. This is important because a page with high exposure but rapidly deteriorating momentum is operationally different from a low-volume page with little evidence of movement.

The recommendation engine does not interpret these signals causally. A reason code means that a pre-decision condition was present when the page received a high risk score; it does not mean that condition caused the future decline.

---

## 6. Limitations & honest framing

This is observational data. The model identifies pages that are directionally associated with a higher probability of the defined future-decline outcome; it does not establish why a page declines.

The target is a defined 30-day performance outcome rather than direct business impact. A correct prediction therefore means that the page matched the operational decline definition, not that the page necessarily lost business value.

The model cannot prove that refreshing, rewriting, consolidating, or pruning a page will cause recovery. The recommendations are decision-support for prioritizing human review.

The sealed evaluation represents one future test period. Performance can change across months, clients, content types, search environments, and seasonal conditions.

The data is an unbalanced panel, meaning clients have different amounts of historical coverage. Tracking availability also varies across clients and data sources.

The observed decline rates changed substantially across decision dates. This is an important warning against treating a single metric as a timeless benchmark.

Finally, the project does not predict Google's algorithm and does not establish causal refresh impact.

---

## 7. Ranked recommendations

The production-oriented output is a ranked queue. Each row contains:

**risk score → rank → reason code → action → priority band**

The reason codes are generated only from pre-decision features.

| Reason code | Operational interpretation | Suggested action |
|---|---|---|
| `TRAFFIC_AND_CLICK_DECLINE` | Impression and click momentum are both weak | Priority refresh review |
| `TRAFFIC_MOMENTUM_WEAK` | Recent impression momentum is weak | Performance review |
| `CLICK_MOMENTUM_WEAK` | Recent click momentum is weak | CTR/snippet review |
| `LOWER_SEARCH_POSITION` | Average position is relatively weak | Search visibility review |
| `CONTENT_STALENESS` | Content age is high | Content refresh review |
| `MULTI_SIGNAL_REVIEW` | No single reason dominates | General decline-risk review |

### How an editor should use the queue

1. Start with the highest-ranked pages.
2. Inspect the reason code and supporting pre-decision signals.
3. Check for obvious explanations such as seasonality, consolidation, tracking changes, or low-volume noise.
4. Decide whether the appropriate action is refresh, performance investigation, CTR/snippet review, search-visibility review, or monitoring.
5. Record the editorial decision separately from the model output.

The model should **prioritize human attention**, not automatically prescribe an irreversible content action.

---

## 8. Reproducibility

The project is implemented in the repository's `work/notebooks/` workflow, with the capstone notebook providing the end-to-end model, validation, leakage audit, recommendation engine, and artifact generation.

Key reproducibility choices:

- Random seed: **42**
- Feature window: **90 days**
- Target window: **30 days**
- Decline threshold: **30%**
- Minimum feature impressions: **300**
- Minimum feature observations: **60**
- Minimum target observations: **21**
- Training decision dates: **2026-02-28**, **2026-03-31**
- Validation decision date: **2026-04-30**
- Sealed test decision date: **2026-05-31**

The capstone notebook should be run from a fresh clone with the repository's documented Python dependencies. It generates the metrics and paper artifacts used in this report.

---

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

Data source: [FlyRank](https://flyrank.ai).

---

## Closing takeaway

The practical result is a **ranking system for review prioritization**, not an automated SEO decision-maker.

The strongest evidence is that the final Random Forest substantially improves broader ranking discrimination over the transparent baseline on the sealed May test, while the two methods are tied at Precision@20. The appropriate conclusion is therefore measured: the learned model adds useful ranking information, but human review remains necessary and the analysis does not establish causal reasons for decline or the effect of any subsequent content change.
