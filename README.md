# Applied Search Intelligence: Google Search Ranking & Discoverability

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.12+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/)
[![Research Paper](https://img.shields.io/badge/Research%20Paper-Live%20on%20GitHub%20Pages-brightgreen)](https://sumit2003-dev.github.io/flyrank-ml-internship/)
[![Demo Video Guide](https://img.shields.io/badge/Demo%20Video-3--5%20min%20Walkthrough-red)](work/RETROSPECTIVE.md#5-35-minute-live-video-demo-script-for-recording)
[![Data Source](https://img.shields.io/badge/Data-FlyRank%20Warehouse-blueviolet)](https://flyrank.ai)

> **Live Deployed Research Paper:** [https://sumit2003-dev.github.io/flyrank-ml-internship/](https://sumit2003-dev.github.io/flyrank-ml-internship/)  
> **Author:** Sumit Kumar (FlyRank Applied ML & General AI Fluency Internship)  
> **Track:** Refresh / Content Opportunity Scoring  
> **Final Retrospective & Deliverables Index:** [`work/RETROSPECTIVE.md`](work/RETROSPECTIVE.md)  
> **Demo Video Script & Breakdown:** [`work/RETROSPECTIVE.md#5`](work/RETROSPECTIVE.md#5-35-minute-live-video-demo-script-for-recording)

---

## 1. Overview: What It Does and For Whom

Modern enterprise content marketing and SEO teams manage expansive portfolios spanning 10,000 to over 100,000 URLs. Over time, indexed content inevitably suffers from **search ranking decay**: rankings slip, impressions drop, and click volume erodes. However, manual editorial audits cannot scale across the whole catalog—human editorial teams typically have enough sprint bandwidth to audit only **50 URLs every two weeks**.

**What this system does:**
This project replaces coarse, static heuristic SEO rules with a leak-free supervised machine learning ranking engine. It continuously evaluates historical 90-day search signals across tens of thousands of pages, predicts the probability of downward search momentum, and outputs a prioritized **Content Action Playbook** equipped with transparent diagnostic reason codes (`refresh_and_review_ctr`, `expand_and_refresh`, `monitor`).

**For whom:**
- **Enterprise SEO Directors & Content Strategists:** Who must allocate scarce bi-weekly editorial sprint hours to the highest-yield pages.
- **Content Operations & Editorial Teams:** Who need clear, actionable work orders (meta tag rewrites vs. depth expansion) rather than unranked lists.

---

## 2. System Architecture & Data Flow

```text
┌────────────────────────────────────────────────────────────────────────┐
│             FlyRank Search Warehouse (79M Telemetry Rows)              │
│       Google Search Console + Google Analytics (BigQuery / DuckDB)     │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Strict Data Contract Partition                       │
│    24 Trailing Signals (90d)  │  4 Context (client_id) │ Banned Outcome│
│  (impressions, position, ctr) │  (GroupKFold Splitting)│ (last 30d GSC)│
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                 Client-Holdout Grouped Validation                      │
│        Train on 80% Client Domains ──► Evaluate on 20% Held-Out        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Supervised Ensemble vs. Baseline                     │
│    Heuristic Rule: P@50 = 0.240  vs.  Random Forest: P@50 = 0.740      │
│                     (~3.1x Operational Lift, +0.500)                   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│              Action Playbook Queue & Operational Orders                │
│   • refresh_and_review_ctr (positions 4-30 + sub-2% CTR: rewrite meta) │
│   • expand_and_refresh (thin content <1000 words: add topical depth)   │
│   • refresh / monitor (evergreen pillar verification / low-risk queue) │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. V2 Evaluation Results (Client-Holdout Benchmark)

To reflect real-world deployment on unseen client brand portfolios, all candidate models were evaluated on identical **5-fold `GroupKFold` splits grouped by `client_id`** against the enterprise baseline heuristic rule:

| Model / Strategy | Validation Split | ROC AUC | Precision@20 | Precision@50 | Lift vs. Baseline (P@50) | Holdout Base Rate |
|---|---|:---:|:---:|:---:|:---:|:---:|
| **Baseline Heuristic Rule** | Client Holdout | 0.627 | 0.250 | **0.240** | — | 0.542 |
| **Logistic Regression (L2)** | Client Holdout | 0.700 | 0.750 | **0.740** | +0.500 | 0.542 |
| **Decision Tree (depth=5)** | Client Holdout | 0.742 | 0.700 | **0.700** | +0.460 | 0.542 |
| **Random Forest Ensemble** | **Client Holdout** | **0.747–0.763** | **0.750** | **0.740** | **+0.500 (~3.1× lift)** | 0.542 |

**Key Takeaway:** The heuristic baseline produces massive tied score clusters, resulting in only 12 true declining pages in the top 50 review batch (38 wasted editor audits). The learned ensemble delivers **37 true declining pages in the top 50**, saving over 150 editorial hours annually per enterprise client.

---

## 4. Scientific Limitations & Honest Framing

1. **Non-Causal Design:** This is an observational ranking model. It predicts which pages are currently experiencing decay momentum; it does *not* causally guarantee that executing a refresh will automatically reverse ranking loss without a prospective A/B test.
2. **Proxy Target Horizon:** The label evaluates historical downward trajectory within the observation export rather than tracking forward post-refresh recovery.
3. **No Algorithmic Reverse-Engineering:** We do not claim to "model Google's algorithm." Search ranking involves hundreds of real-time signals beyond this observational slice.
4. **Human Review Required:** Model rankings provide decision-support, not autonomous publishing. High-equity pillar pages and seasonal content must never undergo automated rewrites.

---

## 5. Stranger-Reproducible Setup (Quickstart)

Any stranger can clone this repository and reproduce all tables, figures, models, and PDF reports in under 3 minutes:

### Local Installation
```bash
# 1. Clone repository
git clone https://github.com/Sumit2003-dev/flyrank-ml-internship.git
cd flyrank-ml-internship

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # On Windows: .\venv\Scripts\activate

# 3. Install pinned dependencies
pip install -r requirements.txt

# 4. Execute the full reference pipeline
python scripts/run_all.py
```

### Google Colab (One-Click Cloud Execution)
Open any milestone notebook in your browser with zero local installation:

| Week | Milestone | Notebook | Open in Colab |
|:---:|:---:|---|:---:|
| **W1** | ML-02 | Research Question & Initial Discovery | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w01_research_question.ipynb) |
| **W2** | ML-03 | ML Task Framing & Error Asymmetry | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w02_ml_task_framing.ipynb) |
| **W3** | ML-04 | 44-Column Data Contract & Partitioning | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w03_data_contract.ipynb) |
| **W3** | ML-05 | Feature Leakage Check & Banned Windows | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w03_feature_leakage_check.ipynb) |
| **W4** | ML-06 | Signal Auditing (Freshness vs. Visibility) | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w04_signal_audit.ipynb) |
| **W4** | ML-07 | Heuristic Baseline Formulation | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w04_baseline_score.ipynb) |
| **W5** | ML-08 | Model Training & Comparison | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w05_model.ipynb) |
| **W6** | ML-09 | Validation Audit (`GroupKFold` vs. Random) | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w06_validation_audit.ipynb) |
| **W7** | ML-10 | Content Action Playbook Queue & Reason Codes | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/w07_action_playbook.ipynb) |
| **W8** | ML-11 | Capstone Master Notebook | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Sumit2003-dev/flyrank-ml-internship/blob/main/work/notebooks/capstone.ipynb) |

---

## 6. Usage & Generated Artifacts

Executing `python scripts/run_all.py` runs the pipeline and outputs:
- `outputs/refresh_queue.csv`: Prioritized production review queue with decay probability scores.
- `work/outputs/action_playbook_queue.csv`: Action queue mapped to operational work orders and diagnostic reason codes.
- `outputs/flyrank_refresh_model_results.pdf`: Publication-ready executive presentation report.
- `work/capstone_report.md`: Complete scientific write-up matching the deployed research paper.
- `work/figures/model_benchmark_comparison.png`: High-resolution benchmark comparison figure.

---

## 7. AI Transparency Diligence

In accordance with AI Fluency transparency standards:
- **Build Partner:** Built with AI assistance (Google DeepMind Antigravity / Claude Code) serving as an interactive pair-programmer for code scaffolding, repetitive data transformation, and responsive CSS/HTML styling.
- **Human Verification:** All core statistical hypothesis tests, the 44-column data contract taxonomy, the 5-fold `GroupKFold` holdout logic, leakage elimination rules, heuristic baseline formulations, model hyperparameter bounds, and non-causal claims were audited, verified, and validated directly by the author.

---

## 8. Acknowledgments & Data Credit

This research was **Built on the FlyRank ML Internship dataset**, provided by [FlyRank (https://flyrank.ai)](https://flyrank.ai).
- Underlying Warehouse: 79M rows of enterprise search telemetry across multi-client brand networks.
- Academic Access: Provided via [FlyRank/internship-warehouse on Hugging Face](https://huggingface.co/datasets/FlyRank/internship-warehouse).
- Mentorship: Mirza Bilal & the FlyRank Research Team.

---

## 9. Comprehensive Project Deep Dive (For New Viewers & Evaluators)

Welcome! If you are a recruiter, engineering manager, or peer reviewer visiting this project for the first time, this section provides an all-in-one summary of the **problem**, **data scale**, **engineering milestones**, **benchmark statistics**, and **operational impact** of this work.

---

### Project Key Statistics at a Glance

| Metric / Dimension | Value | Operational Context |
|---|---|---|
| **Warehouse Telemetry** | **79,000,000+ rows** | BigQuery/DuckDB production Google Search Console & GA telemetry |
| **Analyzed Study Scope** | **30,000 unique URLs** | Curated research sample across **32 enterprise brand domains** |
| **Raw Schema Columns** | **44 raw columns** | Partitioned into strict data contract categories |
| **Leak-Free Features** | **24 signals** | 90-day trailing aggregates available before prediction moment $T$ |
| **Banned Leakage Fields** | **14 outcome columns** | 30-day forward windows & coarse target bins strictly banned |
| **Validation Methodology** | **5-Fold `GroupKFold`** | Split strictly by `client_id` (evaluated on held-out brand domains) |
| **Portfolio Base Rate** | **54.207%** | 16,262 declining content URLs out of 30,000 |
| **Baseline Rule Precision@50** | **0.240 (24.0%)** | Legacy heuristic rules (`days >= 90 AND ctr < 2.0%`) |
| **Random Forest Precision@50** | **0.740 (74.0%)** | Our trained ensemble model evaluated on unseen clients |
| **Top-Queue Operational Lift** | **+0.500 absolute (~3.1×)** | **37 true declining pages in top 50** vs. 12 under legacy rules |
| **Model ROC AUC** | **0.747 – 0.763** | Generalization across diverse multi-client search distributions |
| **Action Playbook Tiers** | **5 operational work orders** | Direct mapping from probability scores to editorial actions |

---

### What I Have Done (The Complete Engineering Journey)

Over 8 weeks in the FlyRank Applied ML track, I engineered this project from raw search telemetry to an operationalized production decision system:

1. **Problem Framing & Error Asymmetry (Weeks 1–2):**
   - Modeled the operational reality of enterprise publishing: 30,000 pages vs. an editorial budget of 50 bi-weekly reviews.
   - Proved that decision error costs are highly asymmetric: False Positives cost 3–5 minutes of editor inspection; False Negatives (missed decay) compound into an 80%+ traffic loss as high-impression URLs slip off Google Page 1.
2. **Data Safety & Leakage Prevention (Week 3):**
   - Designed a formal 44-column Data Contract dividing the schema into Features (24), Labels (2), Context (4), and Excluded (14).
   - Eliminated target leakage by strictly banning forward 30-day evaluation windows (`impressions_last_30d`, `clicks_last_30d`) from the feature vector.
   - Handled structural missingness and sentinel zeroes (`avg_position = 0` indicating unranked pages rather than rank zero).
3. **Empirical Signal Auditing & Baseline Formulation (Week 4):**
   - Tested real-world SEO assumptions against data ($N \ge 50$): confirmed freshness vs. decay (pages un-updated for 180+ days have a 67.8% decline rate vs. 44.1% for fresh content); discovered counter-intuitive visibility dynamics (high-impression pages decay at >65% due to competitive displacement).
   - Formulated a mathematical heuristic baseline reflecting industry best practice to establish an honest benchmark.
4. **Supervised Model Engineering (Week 5):**
   - Trained L2 Logistic Regression, depth-constrained Decision Trees (`max_depth=5`), and Random Forest ensembles (100 trees, `min_samples_leaf=10`).
   - Discovered that daily search presence (`days_with_impressions`) and demand scale (`log_impressions_90d`) are the primary predictors of ranking stability.
5. **Validation Audit & Leakage Hunting (Week 6):**
   - Demonstrated that standard random train/test splits caused client-domain memorization.
   - Enforced 5-fold `GroupKFold` grouped by `client_id` to evaluate genuine out-of-domain transfer on brand domains never seen during training.
   - Performed deep error analysis on top false positives (high-authority evergreen pillars) and false negatives (long-tail pages with low absolute counts).
6. **Content Action Playbook & Human Guardrails (Week 7):**
   - Translated raw probability scores into 5 human-readable editorial work order tiers.
   - Built diagnostic reason codes (`striking_distance_low_ctr`, `stale_unupdated`, `low_engagement_depth`).
   - Implemented mandatory operational guardrails: no automated rewrites, seasonal traffic drop exclusions, and sitewide pillar URL protection.
7. **End-to-End Capstone & Deployment (Week 8):**
   - Generated reproducible automated pipelines (`scripts/run_all.py`), publication-grade PDF executive reports, and ranked CSV queues.
   - Deployed the full scientific research paper to GitHub Pages ([Live URL](https://sumit2003-dev.github.io/flyrank-ml-internship/)).
   - Authored the 5-minute showcase demo outline, shareable social/employer cuts, and 8-week retrospective.

---

### The Content Action Playbook (How Teams Use This System)

Instead of handing editors an unranked spreadsheet of 30,000 rows, the system outputs an prioritized queue mapped to concrete actions:

```text
Ranked URLs ──► Model Decay Probability ──► Diagnostic Reason Codes ──► Action Work Order
```

| Action Tier | Priority | Trigger Conditions | Editorial Work Order |
|---|:---:|---|---|
| **`refresh_and_review_ctr`** | **1 (Top)** | High decay probability + striking distance rank (positions 4–30) + sub-par CTR (<2.0%) | **SERP Snippet Overhaul:** Rewrite meta title and description for search intent; implement FAQ schema to win Page 1 click equity. |
| **`refresh_and_review_engagement`** | **2** | High decay probability + low on-page engagement (<30%) + high bounce | **UX & Layout Restructuring:** Add executive bullet summary, re-order headings, embed interactive media, remove conversion friction. |
| **`expand_and_refresh`** | **3** | High decay probability + thin content (<1,000 words) | **Depth & Topical Authority:** Deepen semantic breadth, resolve secondary user questions, add authoritative references. |
| **`refresh`** | **4** | Mature decaying pillar content with established traffic equity | **Fact & Currency Refresh:** Update outdated statistics, verify broken links, add current-year examples, refresh publication date. |
| **`monitor`** | **5** | Stable content or low-confidence predictions | **Passive Monitoring:** No human intervention required. |

---

### Project Deliverables & Navigation Directory

- **Live Research Paper:** [https://sumit2003-dev.github.io/flyrank-ml-internship/](https://sumit2003-dev.github.io/flyrank-ml-internship/)
- **Comprehensive Written Capstone Report:** [`work/capstone_report.md`](work/capstone_report.md)
- **Executive PDF Report:** [`outputs/flyrank_refresh_model_results.pdf`](outputs/flyrank_refresh_model_results.pdf)
- **Full Retrospective & Deliverables Master Index:** [`work/RETROSPECTIVE.md`](work/RETROSPECTIVE.md)
- **The Plan to Keep Building:** [`work/PLAN_TO_KEEP_BUILDING.md`](work/PLAN_TO_KEEP_BUILDING.md)
- **Production Ranked Queues:** [`outputs/refresh_queue.csv`](outputs/refresh_queue.csv) & [`work/outputs/action_playbook_queue.csv`](work/outputs/action_playbook_queue.csv)
- **All Assignment Notebooks:** [`work/notebooks/`](work/notebooks/) (`w01` through `capstone.ipynb`)

---

*© 2026 Sumit Kumar · Released under the [MIT License](LICENSE).*
