# Capstone Report — Applied Search Intelligence: Google Search Ranking & Discoverability

- **Author:** Sumit Kumar
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** [https://github.com/Sumit2003-dev/flyrank-ml-internship](https://github.com/Sumit2003-dev/flyrank-ml-internship)
- **Deployed Research Paper:** [https://sumit2003-dev.github.io/flyrank-ml-internship/](https://sumit2003-dev.github.io/flyrank-ml-internship/)
- **Date:** March 2026

---

## 1. Problem Framing

### Decision Context & Unit of Analysis
In enterprise search marketing, organizations publish and maintain tens of thousands of content URLs across diverse brand domains. As search algorithms evolve and competitors publish fresh material, existing content inevitably decays in search ranking and organic visibility. However, editorial bandwidth is strictly constrained: content teams cannot manually review and refresh 30,000 pages every quarter.

This study supports a specific operational decision: **Which content URLs should editorial teams prioritize for review and refresh in their bi-weekly production sprints?**

- **Unit of Analysis:** One row = one unique content page (URL), uniquely tracked by pseudonymous identifier `content_id`.
- **System Output:** A continuously ranked queue with probability scores and actionable human-readable reason codes (`refresh_and_review_ctr`, `refresh_and_review_engagement`, `expand_and_refresh`, `refresh`, `monitor`).
- **Human Action:** An editorial strategist opens the ranked queue, inspects top candidates along with their diagnostic reason codes, and assigns appropriate work orders (metadata optimization, on-page restructuring, depth expansion, or fact updating).
- **Asymmetry of Errors:**
  - *False Positive (Page flagged for refresh, but actually stable):* Low cost. An editor spends 5 minutes reviewing a performing page and marks it as healthy.
  - *False Negative (Declining page missed by the system):* High, compounding cost. A high-impression page that quietly slips from Page 1 to Page 3 loses 80%+ of its organic click traffic. Once ranking equity is lost, recovery often takes months of backlink re-acquisition and crawl re-indexing.

### Why Machine Learning Over Heuristics?
Traditional SEO relies on coarse heuristic filters (e.g. `days_since_update >= 180 AND impressions >= 500`). While this identifies a broad pool of stale content, it creates thousands of ties and fails to differentiate pages slipping in SERP rank from evergreen pillar assets that remain stable. Machine learning models evaluate non-linear interactions across multidimensional signals (impressions, rank trajectories, CTR, engagement depth, and content characteristics), delivering continuous prioritization and superior top-K precision.

---

## 2. Data Safety & Data Contract

### Dataset Scope & Protection
All analyses were conducted using the pseudonymized FlyRank search intelligence dataset (`data/raw/content_refresh_anonymized.csv`), comprising **30,000 unique content items across 32 enterprise clients**.
- To prevent private client identification, all URLs, client domain names, article titles, and search queries are completely absent or irreversibly hashed.
- Identifiers (`content_id`, `client_id`) are pseudonymous strings used strictly for database joins and validation fold grouping, never as feature variables.

### Feature Classification & Leakage Prevention Policy
Every field in the dataset was classified into a formal data contract:
1. **Features (24 signals):** Observational inputs knowable prior to prediction time:
   - Volume aggregates (90d): `impressions_90d`, `clicks_90d`, `pageviews_90d`, `sessions_90d`, `users_90d`, `engaged_sessions_90d`, `ai_sessions_90d`, `scroll_events_90d`.
   - Breadth & timing: `days_with_impressions`, `days_with_sessions`, `content_age_days`, `days_since_last_update`.
   - Search efficiency rates: `ctr`, `avg_position`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`.
   - Keyword & content depth: `search_volume`, `competition`, `cpc`, `word_count`, `char_count`, `content_type`, `main_intent`.
2. **Label / Proxy Target (2 fields):** `is_declining_label` (binary target: `1` if `trend_direction == 'down'`, else `0`), `trend_pct`.
3. **Context / Grouping (4 fields):** `content_id` (identifier), `client_id` (grouped holdout splitting), `provider_used`, `model_used` (LLM metadata excluded to prevent tech-stack confounding).
4. **Deliberately Excluded (14 fields):**
   - Direct target leakage: `trend_direction` and `trend_pct` (the mathematical source of the label).
   - Temporal evaluation window overlap: `impressions_last_30d`, `clicks_last_30d`, `sessions_last_30d`, `impressions_prev_30d`, `clicks_prev_30d`, `sessions_prev_30d` (these span the outcome observation period).
   - Coarse bins: `competition_level`, `age_tier`, `freshness_tier`, `word_count_tier`, `char_count_tier`, `impression_tier`, `position_tier`, `age_tier_order` (continuous variables preserved full nuance).

### Systematic Missingness & Sentinel Zeroes
- Rate metrics are expressed as percentages: `ctr = 0.76` signifies **0.76%**, not 76%.
- `avg_position = 0` is a sentinel representing unranked / no position data (1,205 rows), flagged explicitly with `has_position_data`.
- Keyword metadata exhibits structural missingness by `content_type` (e.g. `feedly article` has 100% missing search volume and 28.5% missing word count). Rather than imputing raw zeros, we engineered explicit indicator flags (`has_search_volume`, `has_word_count`).

---

## 3. Baseline Model

### Formulation & Reason Codes
To ensure honest benchmarking, we established a transparent, hand-crafted heuristic rule representing common enterprise SEO best practice:
$$\text{Baseline Score} = \text{stale} \times \text{visible} \times \left( 40 \times \text{striking\_distance} + 30 \times \text{low\_ctr} + 30 \times \min(1.0, \frac{\text{impressions}}{10000}) \right)$$
where:
- $\text{stale} = \mathbb{I}(\text{days\_since\_last\_update} \ge 90)$
- $\text{visible} = \mathbb{I}(\text{impressions\_90d} \ge 300)$
- $\text{striking\_distance} = \mathbb{I}(4.0 \le \text{avg\_position} \le 30.0)$
- $\text{low\_ctr} = \mathbb{I}(\text{ctr} < 2.0\%)$

### Baseline Benchmark Performance
Across the dataset, the baseline rule identifies 14,834 qualifying pages. However, because it relies on coarse threshold cutoffs:
- **Precision@50 on Client Holdout:** **0.240** (12 true declining pages in top 50).
- **ROC AUC:** **0.627**.
- **Limitation:** The heuristic rule produces massive tied score clusters at top boundaries and struggles to separate high-traffic evergreen articles that have naturally leveled off from articles experiencing active decay.

---

## 4. Model & Analysis

### Methodology & Architecture Selection
We evaluated three supervised learning architectures:
1. **Logistic Regression (L2 Regularized):** Establishes the linear baseline of signal predictiveness.
2. **Decision Tree (`max_depth=5`):** Discovers non-linear hierarchical rules and interaction thresholds.
3. **Random Forest Ensemble (100 estimators, `max_depth=10`, `min_samples_leaf=10`):** Combines variance reduction through bagging with randomized feature subspace sampling, capturing high-order non-linear interactions across traffic volume, position dynamics, and content age.

### Target Definition
The target variable is binary search momentum decay:
$$\text{Target } y_i = \begin{cases} 1 & \text{if } \text{trend\_direction}_i = \text{"down"} \\ 0 & \text{otherwise} \end{cases}$$
The portfolio base rate is **54.2% positive** (16,262 declining pages out of 30,000).

---

## 5. Evaluation & Validation Audit

### Grouped Client-Holdout Validation Design
To eliminate cross-client memorization and simulate real-world enterprise deployment on a new brand domain, we utilized a **5-fold `GroupKFold` split grouped by `client_id`**.
- Models are trained on 80% of client portfolios and evaluated exclusively on the held-out 20% of clients.
- In `work/notebooks/w06_validation_audit.ipynb`, we proved that a naive random 80/20 split inflated Precision@50 and AUC due to domain memorization. The client-grouped holdout provides the only honest estimate of generalization.

### Benchmark Results (Same Split, Same Metric)

| Model / Architecture | Split Strategy | ROC AUC | Precision@20 | Precision@50 | Lift over Baseline (P@50) | Holdout Base Rate |
|---|---|---:|---:|---:|---:|---:|
| **Baseline Heuristic Rule** | Client Holdout | 0.627 | 0.250 | **0.240** | — | 0.542 |
| **Logistic Regression** | Client Holdout | 0.700 | 0.750 | **0.740** | +0.500 | 0.542 |
| **Decision Tree (depth=5)** | Client Holdout | 0.742 | 0.700 | **0.700** | +0.460 | 0.542 |
| **Random Forest Ensemble** | Client Holdout | **0.747–0.763** | **0.750** | **0.740** | **+0.500** | 0.542 |

*Note: In the full reference pipeline execution, Random Forest achieved an overall Precision@50 of 0.740 (+0.500 lift over baseline), representing a **~3.1x lift in top-queue precision**.*

### Error Analysis
- **Top False Positives:** High-impression pillar pages that maintain Page 1 rankings despite being stale (>200 days). The model penalizes high impression volume when paired with staleness, even though high domain authority occasionally insulates these pages from traffic loss.
- **Top False Negatives:** Long-tail, low-impression articles (<300 impressions) that dropped sharply in percentage terms (e.g. from 80 to 20 impressions). These drops represent significant relative decay but minimal absolute opportunity.

---

## 6. Interpretation & Findings

### Top Feature Importances (Random Forest Gini Importance)
1. `days_with_impressions` (0.1606): Frequency of daily search visibility is the single strongest indicator of content stability.
2. `log_impressions_90d` (0.1285): High historical demand scale creates the primary exposure to measurable decay.
3. `avg_position` (0.1084): Rank placement governs vulnerability; positions between 4 and 30 exhibit the highest sensitivity to staleness.
4. `content_age_days` (0.0950): Older content displays higher compounding decay risk.
5. `word_count` & `char_count` (0.0810 combined): Comprehensive depth provides baseline resilience against rank volatility.
6. `ctr` (0.0332) & `log_clicks_90d` (0.0325): Direct user demand response.

### Empirical Signal Verdicts
- **Freshness vs Decay:** **CONFIRMED.** Pages un-updated for 181+ days show elevated decline rates (67.8% vs 44.1% for content updated within 31–90 days).
- **Visibility vs Stability:** **OPPOSITE / COUNTER-INTUITIVE.** High-impression tiers experience higher rates of downward trend (>65%) compared to low-impression tiers (~45%), because high visibility creates higher statistical exposure to competitive displacement.
- **Position vs Click Capture:** **CONFIRMED.** Click capture plummets monotonically from Top 3 (weighted CTR ~4.2%) to Page 2–3 (weighted CTR <0.4%).

---

## 7. Operational Recommendations: Content Action Playbook

### Tiered Action Queue & Reason Codes
Predictions are translated into five explicit editorial action tiers:
1. **`refresh_and_review_ctr` (Top Priority):** Pages with high predicted decline risk, striking-distance position (4–30), and sub-par CTR (<2.0%). *Work order:* Overhaul title tags, refine meta descriptions to target search intent, and implement rich schema snippets.
2. **`refresh_and_review_engagement`:** Stale, visible content with engagement rates below 30%. *Work order:* Restructure visual hierarchy, introduce executive summaries, and add interactive assets to curb bounce.
3. **`expand_and_refresh`:** Decaying content with thin body copy (<1000 words). *Work order:* Deepen topical authority, resolve semantic questions, and add concrete case studies.
4. **`refresh`:** High-confidence decaying mature content. *Work order:* Routine editorial refresh: verify dead links, update statistics, and refresh publication dates.
5. **`monitor`:** Low-risk, stable content. No immediate intervention required.

### Mandatory No-Go Rules (Human Review Guardrails)
- **Do NOT automate content rewrites:** The model provides decision support, not autonomous publishing.
- **Exclude Seasonal Assets:** Holiday, annual tax, or Black Friday pages naturally exhibit temporary impression drops during off-season months.
- **Protect Canonical Pillar URLs:** Pages with critical sitewide backlink equity must never undergo URL changes or structural stripping without technical SEO approval.
- **Protect Paid Landing Pages:** Ensure URLs involved in active paid campaigns are reviewed by performance marketing prior to copy alterations.

### Monitoring & Retrain Triggers
- **Google Broad Core Update:** Freeze queue generation during active algorithmic rollouts; retrain models 30 days post-rollout once SERP volatility subsides.
- **Covariate Shift / SERP Changes:** If Google AI Overviews or layout changes trigger portfolio-wide CTR drops exceeding 25%, recalibrate rate features.
- **Performance Degradation:** Retrain if quarterly audit shows Precision@50 slipping below 0.60 on holdout validation.

---

## 8. Reproducibility

### Environment & Execution Instructions
To reproduce all outputs, models, and evaluation tables from a fresh repository clone:

```bash
# 1. Clone repository
git clone https://github.com/Sumit2003-dev/flyrank-ml-internship.git
cd flyrank-ml-internship

# 2. Install dependencies
pip install -r requirements.txt

# 3. Execute the full reference pipeline
python scripts/run_all.py
```

### Random Seeds & Environment Parameters
- Random seed fixed across all scripts and notebooks: `random_state = 42`.
- Python version: `Python 3.12+` / `Python 3.14.0`.
- Key package versions: `scikit-learn >= 1.4.0`, `pandas >= 2.2.0`, `numpy >= 1.26.0`, `duckdb >= 1.0.0`, `reportlab >= 4.0.0`.

### Data Verification
- Raw dataset: `data/raw/content_refresh_anonymized.csv` (SHA-256 verified, 30,000 rows × 44 columns).
- Cleaned feature vector: `data/processed/refresh_feature_vector.csv` (30,000 rows × 52 columns).
- Output artifacts generated in `outputs/` and `work/outputs/`:
  - `model_report.md`
  - `model_report.pdf`
  - `refresh_queue.csv`
  - `baseline_action_score.csv`
  - `action_playbook_queue.csv`
  - `charts/roc_curve.png`
  - `charts/precision_at_k.png`
  - `charts/feature_importance.png`

---

## 9. 5-Minute Demo Outline (Week-8 Showcase)

A structured 5-minute walkthrough of the project, ready for presentation:

- **Minute 1: The Question & Problem Context**
  - *Context:* FlyRank manages content as infrastructure across tens of thousands of enterprise pages. In Google Search, content inevitably experiences decay.
  - *Operational Decision:* With limited human editorial bandwidth in bi-weekly sprints, which pages should an editor refresh first?
  - *Status Quo:* Production relied on hand-written heuristic flags (`days_since_update >= 90 AND ctr < 2.0%`) that produced massive score ties and low top-queue precision.

- **Minute 2: Data Safety & Leakage-Free Methodology**
  - *Data Scope:* 30,000 pseudonymized URLs across 32 brand domains drawn from FlyRank's 79M-row search warehouse. Fully public-safe (no client names, queries, or URLs).
  - *Leakage Hunt:* 44 columns partitioned strictly; 24 historical 90-day trailing signals retained. Outcome windows (last 30d impressions/clicks) strictly banned.
  - *Validation Design:* 5-fold `GroupKFold` grouped strictly by `client_id` to evaluate genuine generalization to unseen client domains.

- **Minute 3: Headline Chart & Model Comparison**
  - *Visual:* Present Figure 2 (`img/model_benchmark_comparison.png`).
  - *Comparison:* Hand-crafted heuristic baseline achieves **Precision@50 of 0.240** on held-out clients (38 out of 50 editor reviews wasted). Our Random Forest ensemble delivers **Precision@50 of 0.740** against a 0.542 base rate.

- **Minute 4: Honest Result & Scientific Limitations**
  - *The Headline Lift:* A **~3.1× precision improvement (+0.500 absolute)** in top-50 editorial review batches (37 true declining items vs. 12).
  - *Honest Limitations:* Observational ranking model, not a causal guarantee. We predict decay risk, not Google's ranking algorithm. Re-ranking prioritization requires editorial judgment.

- **Minute 5: Actionable Recommendation (Playbook)**
  - *Operational Hand-Off:* Translate probabilities into the Content Action Playbook with transparent reason codes.
  - *Key Action:* Top priority `refresh_and_review_ctr` (decaying pages in striking distance 4–30 with low CTR) gets meta tag and search intent overhauls, while thin pages get expansion.
  - *Guardrails:* Mandatory human-in-the-loop review; seasonal and core pillar URLs protected from automated alterations.

---

## 10. Shareable Cuts (Social & Employer Framing)

Two concise, honest summaries tailored for distinct professional audiences:

### Cut 1: Short Social Post (Methodology-Focused)
> **How we beat heuristic SEO rules by 3.1× using honest machine learning.**
>
> In enterprise content operations, teams manage 30,000+ pages. When search traffic dips, static heuristic rules (like "flag if older than 90 days") waste scarce editorial hours—yielding only 24% precision in top-50 review batches.
>
> At @FlyRank, we engineered a supervised ranking engine across 32 enterprise domains:
> 1. Partitioned a 44-column schema to enforce strict leakage prevention (banning outcome observation windows).
> 2. Evaluated on unseen client domains using 5-fold `GroupKFold`.
> 3. Trained a Random Forest ensemble achieving 0.740 Precision@50 (+0.500 lift over baseline).
>
> Honest takeaway: It’s a high-precision prioritization engine for human editors, not an autonomous rewrite bot.
>
> 📄 Read the open research paper: https://sumit2003-dev.github.io/flyrank-ml-internship/
> 💻 Reproducible code & notebooks: https://github.com/Sumit2003-dev/flyrank-ml-internship
>
> #MachineLearning #DataScience #AppliedML #SEO #SearchIntelligence

---

### Cut 2: 3-Sentence Employer Summary ("What I built · On what data · What it showed")
> - **What I built:** I developed a supervised decay-risk ranking pipeline and operational Content Action Playbook that prioritizes enterprise pages for bi-weekly editorial refresh sprints, replacing heuristic flags with continuous probability scoring and explainable reason codes.
> - **On what data:** Built on 30,000 pseudonymized enterprise URLs across 32 client domains from FlyRank's 79-million-row search intelligence warehouse, using a leak-free 24-feature data contract evaluated under grouped client-holdout cross-validation (`GroupKFold`).
> - **What it showed:** The ensemble model achieved a Precision@50 of 0.740 on unseen client domains compared to 0.240 for standard heuristic rules—delivering a ~3.1× operational efficiency lift (+0.500 absolute) while adhering to strict non-causal claim boundaries.

---

## 11. Acknowledgments & Data Credit

This research was **Built on the FlyRank ML Internship dataset**, provided by [FlyRank (https://flyrank.ai)](https://flyrank.ai).
Dataset access and academic usage granted through [FlyRank/internship-warehouse](https://huggingface.co/datasets/FlyRank/internship-warehouse).
