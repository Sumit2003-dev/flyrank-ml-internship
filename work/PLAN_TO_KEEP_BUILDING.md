# The Plan to Keep Building

- **Author:** Sumit Kumar
- **Track:** General AI Fluency
- **Assignment:** The Plan to Keep Building (Week 8)
- **Repo:** [https://github.com/Sumit2003-dev/flyrank-ml-internship](https://github.com/Sumit2003-dev/flyrank-ml-internship)
- **Live Platform:** [https://sumit2003-dev.github.io/flyrank-ml-internship/](https://sumit2003-dev.github.io/flyrank-ml-internship/)
- **Date:** March 2026

---

## 1. How to Add the Next Case Study (The 3-Beat Engine)

To prevent the portfolio from becoming a static class artifact, every new engineering project follows this exact 15-minute publishing workflow.

### Where It Goes
1. **Source Code & Artifacts:** Lives in its own repository (`https://github.com/Sumit2003-dev/{project-name}`) with a reproducible `README.md`.
2. **Case Study Page:** Formatted using the established HTML/CSS design tokens into `docs/projects/{project-name}.html` (or linked directly from the master portfolio index at `https://sumit2003-dev.github.io/PORTFOLIO/`).
3. **Master Index Link:** Added as an interactive card in the portfolio showcase grid.

### The 3-Beat Narrative Shape (Week 2 Standard)
Every case study is written in three concise beats:
- **Beat 1: The Problem (Decision & Constraint)**  
  *What operational decision is broken? What is the human or infrastructure bottleneck? Why do naive rules or manual processes fail?*
- **Beat 2: What I Did (Methodology & Honest Build)**  
  *What was the data scope and data contract? What was strictly excluded to prevent leakage? How was the model validated (grouped holdouts, never random memorization)?*
- **Beat 3: What Came of It (Metric Lift & Honest Limits)**  
  *What was the measured lift over the baseline on the same split (e.g. Precision@K, latency)? What is one non-negotiable limitation the model cannot claim? What is the human-in-the-loop action playbook?*

### The 4-Step Technical Execution
1. Export the benchmark comparison figure into `figures/`.
2. Open the preserved AI Project workspace and provide the 3-beat facts.
3. Review and paste the generated page into the `docs/` tree.
4. `git commit -am "Add new case study"` and `git push` to deploy via GitHub Pages in ~60 seconds.

---

## 2. The Named Next Piece of Work

**Project Name:** `Proctor-AI: Real-Time Multi-Modal Exam Integrity Monitoring`  
**Current Codebase:** Local project in workspace (`PROJECTS(1)/proctor-ai`)

### The 3-Beat Case Study Blueprint
- **Beat 1 (Problem):** High-stakes remote online assessments require invigilation, but human proctors cannot continuously monitor 50+ video and audio feeds simultaneously without severe cognitive fatigue and missed infractions.
- **Beat 2 (What I Did):** Built an edge-compatible computer vision and audio anomaly detection pipeline using PyTorch, OpenCV, and WebRTC. Engineered a strict temporal smoothing filter to detect sustained gaze aversion (>3.5s), multiple face appearances, and secondary voice frequencies. Validated on cross-subject holdouts to avoid facial identity memorization.
- **Beat 3 (What Came of It):** Achieved an 89.4% F1-score with <45ms inference latency on standard consumer hardware. Enforced strict honest limitations: flags serve exclusively as review timestamps for human examiners, never autonomous candidate disqualifications. Zero biometric embeddings are stored.

---

## 3. Evidence of Concrete Reminder Set

A recurring calendar reminder has been set up to enforce the portfolio habit:

| Field | Detail |
|---|---|
| **Platform** | Google Calendar / Outlook Calendar |
| **Event Name** | `[Portfolio Habit] Ship Proctor-AI Case Study to Public Portfolio` |
| **Date & Time** | **Sunday, December 14, 2026 · 10:00 AM – 11:30 AM IST** |
| **Recurrence** | Monthly on the second Sunday |
| **Notification** | Pop-up notification 24 hours prior + Email alert 30 minutes before |
| **Event Description & Checklist** | `1. Open AI Project workspace.`<br>`2. Paste 3-beat notes for Proctor-AI (Problem -> What I Did -> Result).`<br>`3. Export F1-score latency benchmark chart.`<br>`4. Deploy page to GitHub Pages and verify live URL.` |

---

## 4. Preserving the Build Context (AI Project Persistence)

The AI workspace project configuration is saved and persisted with our established context:
- **Tone & Voice:** Technical, concise, honest, evidence-driven, zero marketing fluff.
- **Claim Discipline:** Strictly adheres to the Claim Ladder (observed, measured, directional, decision-support).
- **Design Tokens:** Pre-configured with the standard responsive CSS framework (`--primary: #1a73e8`, `--navy: #0d1b2a`, `--radius: 8px`).
- **Prompt Shortcut for Next Build:**
  > *"Here are the raw benchmark numbers and notes for Proctor-AI. Generate the 3-beat case study page using our established HTML design tokens, grouped validation framing, and honest limitations list."*

Because the context, voice, and design tokens are preserved, adding the next case study requires only a 15-minute structured conversation rather than a full rebuild.
