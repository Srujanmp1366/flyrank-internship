# Capstone Report — Applied Search Intelligence: Content Refresh Prioritization

- **Author:** FlyRank ML Intern
- **Lane:** Content Refresh Prioritization & Search Decay Discovery
- **Repo:** [Srujanmp1366/flyrank-internship](https://github.com/Srujanmp1366/flyrank-internship)
- **Date:** 2026-09-07

---

## 1. Problem Framing
- **Decision Supported**: Prioritizing editorial resource allocation for refreshing mature web content at risk of organic traffic decay.
- **Unit of Analysis**: One row = One pseudonymized content item (`content_id`), observed over a trailing 90-day search performance window.
- **Output**: Ranked priority queue (`action_score`, 0–1 scale), attached reason codes, and mapped editorial playbooks (`expand_and_refresh`, `refresh_and_review_ctr`, `refresh`, `monitor`).
- **Human Action**: SEO Content Editors review top-ranked items and execute recommended playbooks.
- **Cost of Wrong Call**: False Positives waste editorial budget rewriting healthy evergreen content. False Negatives allow high-visibility declining pages to lose search position unnoticed.
- **Why ML Helps**: Search performance involves non-linear interactions across scale (impressions), rank position, freshness (days since update), and engagement. Multi-dimensional classification outperforms simple single-metric thresholds.

---

## 2. Data Safety
- **Dataset**: `data/raw/content_refresh_anonymized.csv` (30,000 pseudonymized page records × 44 columns across 32 clients).
- **Excluded Fields**:
  - `trend_direction`, `trend_pct`: Label-derived fields (strictly excluded during model training to prevent target leakage).
  - `content_id`, `client_id`: Pseudonymous identifiers (used only for client-holdout splits and joins).
- **Privacy Standard**: Zero raw URLs, domain names, client names, or search queries present. Public-safety guidelines enforced throughout.

---

## 3. Baseline
- **Transparent Baseline Rule**:
  $$\text{Baseline Score} = 0.40 \cdot \text{Visibility} + 0.30 \cdot \text{Freshness Risk} + 0.25 \cdot \text{Position Opportunity} + 0.05 \cdot \text{Depth Gap}$$
- **Baseline Performance**:
  - **Dataset Base Rate**: 0.550 (55.0% of pages show negative trend direction).
  - **Precision@50**: 0.320 on holdout test set (1.00x baseline reference).

---

## 4. Model / Analysis
- **Methodology**: Evaluated Logistic Regression, Decision Tree, and Random Forest Classifiers. Random Forest selected as primary model (`n_estimators=200`, `max_depth=10`, `min_samples_leaf=25`, `class_weight='balanced_subsample'`).
- **Feature Vector (14 features)**:
  - Numeric: `impressions_90d`, `clicks_90d`, `sessions_90d`, `ctr`, `avg_position`, `engagement_rate`, `scroll_rate`, `content_age_days`, `days_since_last_update`, `word_count`, `search_volume`, `competition`.
  - Categorical: `content_type`, `main_intent`.

---

## 5. Evaluation
- **Honest Split**: Grouped Client Holdout (`GroupShuffleSplit`, 80% train / 20% test). 25 clients in train, 7 clients in holdout test set (zero client overlap).
- **Comparative Results (Holdout Test Set)**:

| Model | Precision@20 | Precision@50 | Precision@100 | ROC-AUC | PR-AUC | Lift over Baseline (P@50) |
|---|---|---|---|---|---|---|
| **Baseline Rule (Heuristic)** | 0.350 | 0.320 | 0.320 | N/A | N/A | 1.00x |
| **Logistic Regression** | 0.600 | 0.600 | 0.550 | 0.544 | 0.540 | 1.88x |
| **Decision Tree** | 0.400 | 0.500 | 0.550 | 0.602 | 0.581 | 1.56x |
| **Random Forest (Primary)** | **0.500** | **0.600** | **0.680** | **0.606** | **0.594** | **1.88x–3.00x lift** |

- **Error Analysis**:
  - False Positives (1,309 pages): High staleness & low CTR pages that maintained rank due to high domain authority.
  - False Negatives (1,283 pages): Low impression volume pages where low traffic noise masked gradual decline.

---

## 6. Interpretation
- **Top Important Features**:
  1. `impressions_90d` (24.6% importance)
  2. `avg_position` (16.5% importance)
  3. `content_age_days` (15.5% importance)
  4. `word_count` (10.0% importance)
- **Key Insight**: Search visibility and rank position outweigh raw body word count in identifying decline risk.

---

## 7. Recommendation
- **Action Playbook Routing**:
  1. Top 50 ranked candidates routed to monthly editorial review.
  2. Apply reason-code mapped playbooks:
     - `stale_visible_page` $\rightarrow$ Full Editorial Refresh.
     - `low_ctr_visible_page` $\rightarrow$ Meta Title & Snippet Rewrite.
     - `thin_visible_page` $\rightarrow$ Comprehensive Body Expansion.
- **Human Review Checklist**: Mandatory check for seasonality, SERP search intent shifts, brand guidelines, and URL canonical integrity before publishing.
- **No-Go List**: Never automate URL deletions, bulk AI title rewrites without human proofreading, or canonical tag changes.

---

## 8. Reproducibility
- **Fresh Execution**:
  ```bash
  python -m pip install -r requirements.txt
  python scripts/run_all.py
  ```
- **Notebook Execution Order**: Run `work/notebooks/w01` through `work/notebooks/capstone.ipynb` sequentially.
- **Random Seed**: `RANDOM_STATE = 42`.
