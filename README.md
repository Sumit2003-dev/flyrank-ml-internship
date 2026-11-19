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

*© 2026 Sumit Kumar · Released under the [MIT License](LICENSE).*
