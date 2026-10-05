# HOSEN

**Hierarchical Open-world Security detection with Escalation and Novelty scoring**

A research framework for **open-world intrusion detection**: it classifies known attacks, distinguishes previously observed unknown traffic from genuinely unseen attack families, and spends expensive LLM compute only when a bounded escalation controller decides it is worth it.

---

## What HOSEN is

Modern intrusion detection systems are trained in a closed world — they assume the classes seen at training are the classes seen at deployment. Real networks violate this assumption continuously. HOSEN addresses it with a three-tier pipeline:

| Tier | Mechanism | Cost |
|---|---|---|
| **Tier 1** — Fast path | Four-signal fused classifier: ModernBERT confidence + per-class Mahalanobis geometry + background Mahalanobis geometry + RAG similarity, combined by multinomial logistic regression into `{KNOWN, RAG-UNKNOWN, NOVEL}`. | ~116 ms |
| **Tier 2** — Budget controller | Adaptive Semantic Budget Controller: admits at most *K* escalations per time window, ranked by novelty. Stable under burst load, not gameable by adversarial traffic shaping. | negligible |
| **Tier 3** — Verification | Six specialist agents (`llama3.2:1b` via Ollama) that can **confirm** or **abstain**, but cannot unilaterally override the fast path. | ~1.6 s per call (cited) |

The design principle: **an LLM that can override a calibrated classifier on a per-sample basis is a liability; an LLM that can only withhold an override when consensus is reached is a safety mechanism.**

---

## Headline results

| Metric | Value |
|---|---|
| Three-class accuracy (KNOWN / RAG-UNKNOWN / NOVEL) | **0.9922** |
| Macro F1 | 0.9922 |
| Per-class F1 | 0.9899 / 0.9983 / 0.9885 |
| AUC (Known vs NOVEL, 95% CI) | 0.9993 [0.9978, 1.0000] |
| Per-source detection (4 independent Kaggle benchmarks) | 100% on all four |
| Escalation rate to the LLM tier | 3.5% |
| Agent abstention rate | 21 / 21 |
| Speedup at 24% escalation vs pure LLM | 4.0× |

Every number in the paper traces back to a cell in the notebook. Nothing is estimated.

---

## The evolution: why we are sharing the failures

This repository contains the **full Colab notebook with the entire evolution of the project** — from the first idea, through its failure, through the diagnosis, and on to the final architecture. We are sharing it in this complete form so you can learn from our **failures** as well as our **success**.

The evolution has four acts:

**Act I — The original architecture fails.**
The first version used a hand-weighted ensemble of five novelty signals and tested zero-day detection against *synthetic* attack strings (`CRYPTO_MINING`, `AI_POISONING`, `SUPPLY_CHAIN`). It reported 99.90% zero-day detection. It was rejected at the main conference track. The reason: the synthetic test set was constructed by filtering against the same retrieval memory that later served as a novelty signal. The result was **guaranteed by construction**, not demonstrated. The baseline comparison in the notebook shows the honest number: the original detector scores **43.1% accuracy and 0.28 unknown-F1** on real data — worse than ODIN.

**Act II — Diagnosis: confidence is the wrong tool.**
We measured what the base model actually does on unknown traffic. It reaches 98.3% on KNOWN attacks but is systematically overconfident on everything else: mean confidence is 0.9874 on KNOWN, 0.8275 on RAG-UNKNOWN, and 0.7806 on NOVEL. It collapses 90% of RAG-UNKNOWN onto a single class (FTP Brute-Force) and 52.3% of NOVEL onto another (ICMP Flood). Softmax confidence is not a novelty signal. Per-class Ledoit-Wolf Mahalanobis geometry is. Neither alone is sufficient.

**Act III — Fusion, then the fix that mattered.**
A four-feature multinomial logistic model reaches 0.9967 — on *overlapping* calibration and evaluation pools. That number was too good to be true. The fix was to rebuild the evaluation with strictly disjoint splits and a deduplicated retrieval memory. The honest held-out number: **0.9944**. The number we report in the paper, on 900 disjoint samples: **0.9922**.

**Act IV — The full pipeline.**
The W-series experiments (W1–W22) build the complete pipeline: ablation ladder, per-source generalization, statistical tests, threshold sweep, budget controller, latency measurement. The AG-series (AG1–AG8) adds the six specialist agents and the escalation tier. A four-tier leakage audit (exact hash, prefix hash, embedding cosine, kNN) verifies that the NOVEL evaluation set is disjoint from training and retrieval memory.

**The most important negative result in this repository:** RAG similarity alone has AUC **0.5276** on the Known-vs-NOVEL task — essentially chance. It is the dominant feature for the RAG-UNKNOWN class (coefficient +3.97) and contributes to the fusion, but it is **not** a standalone novelty detector. Treating it as one — as our rejected first version did — produces a classifier that looks excellent on a synthetic benchmark and fails on real traffic.

If you are building an open-world detector, we hope our failures save you time.

---

## Repository contents

```
HOSEN/
├── CNCC_2027_(Workshop).ipynb   ← Full notebook with all experiments (W-series, AG-series, FIX cells)
├── README.md                    ← This file
├── training_dataset.csv         ← 4,500 labeled logs across 9 attack classes
├── master_security_dataset.rar  ← 356,734 unlabeled records (retrieval memory, primary)
├── sample_security_dataset.csv  ← 807 unlabeled records (retrieval memory, secondary)
└── UNSW_NB15_testing-set.csv    ← 82,332 records (retrieval memory, secondary cross-check)
```

The notebook is self-contained. Open it in Google Colab, upload the datasets when prompted, and run the cells in order.

---

## What you will find in the notebook

- **Diagnostic cells** that show *why* the original approach failed (base-model behavior on three data types; class collapse; baseline comparison).
- **FIX cells** that rebuild the classifier on solid ground (per-class Ledoit-Wolf Mahalanobis, four-feature multinomial fusion, disjoint calibration/evaluation splits).
- **W1–W22** — the complete experiment suite: canonical partition table, ablation ladder, traffic-to-text spec, decision policy, latency, per-source generalization, RAG audit, statistical tests, extended ablation, cost-quality curve, budget controller, ROC, score distributions, decision regions, confusion matrix, threshold sweep, error analysis, paper-ready summary.
- **AG1–AG8** — the agentic pipeline: escalation setup, overlap audit, fast path, specialist escalation, per-class JSON output, final accuracy, example inspection, full dump.
- **The complete leakage audit** across every pair of pools and every method.

---
sources (UNSW-NB15, and the four Kaggle benchmark datasets used for the NOVEL evaluation pool: `CIC_IDS_Collection`, `BoT_IoT`, `BoTNeTIoT_L01`, `CIC_IoMT_2024`).

For questions, open an issue or contact the corresponding author at the email listed in the paper.
