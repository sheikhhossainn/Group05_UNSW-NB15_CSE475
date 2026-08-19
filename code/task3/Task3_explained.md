# UNSW-NB15 — Task 3 Summary

Group 5 · CSE475 · Track 1 (GNN)

Task 3 does not train a new production model. It **interprets and validates** the two models shipped in Task 2 — the XGBoost baseline (per-flow, 10-class) and the GAT (per-host, binary). Two notebooks:

| # | Notebook | Role |
|---|---|---|
| 1 | `Explainability` | SHAP + LIME on XGBoost, GNNExplainer on the GAT — one correct, one wrong prediction each |
| 2 | `Improvement_Adaption` | Host-grouped CV, GAT ablations, and a McNemar significance test — GNN vs baseline on the same hosts |

Everything reloads Task 2's saved artifacts (`train/test_processed`, side tables, graphs, `label_encoder`, `XGBoost_model`, `gat_temporal_model.pt`, saved predictions). `SEED = 42`. The recurring `ct_ftp_cmd` literal-space fix is reapplied wherever a raw processed frame is loaded. Each section records **methodology**, **result**, and **what it means**.

---

## 1. Explainability

**Method.** Two model families need two tools.
- **XGBoost (tabular)** → **SHAP** (`TreeExplainer`, exact tree attributions) and **LIME** (`LimeTabularExplainer`, local surrogate). Standard textbook use case.
- **GAT (graph)** → **GNNExplainer** (Ying et al., NeurIPS 2019 — in the course reference list). SHAP/LIME assume flat feature vectors and cannot handle message-passing input, so GNNExplainer is the track-appropriate substitute, not a shortcut.

For each model, explain **one correct and one wrong** test prediction so the report can contrast signal vs failure. The GAT graph + architecture are rebuilt from scratch here (same 2-layer `GATv2Conv`, `heads=2`, `hidden=8`, `dropout=0.3`) and `gat_temporal_model.pt` is loaded into it.

**Result.**

| Model | Correct sample | Wrong sample |
|---|---|---|
| XGBoost | idx 0 — true `Normal`, pred `Normal` | idx 86 — true `Reconnaissance`, pred `Exploits` |
| GAT | first correct host node | first misclassified host node |

Test graph rebuilt as `Data(x=[47, 6], edge_index=[2, 2634], edge_attr=[2634, 6], y=[47])`. Outputs are six saved plots — `shap_{correct,wrong}.png`, `lime_{correct,wrong}.png`, `gnnexplainer_{correct,wrong}.png`: SHAP/LIME top-10 signed feature contributions for the predicted class; GNNExplainer per-feature node-mask importances over the six host features.

**Understand.** The `Reconnaissance`→`Exploits` error is exactly the class overlap Task 1's 2-D projections and Task 2's confusion matrix already flagged — the explainer confirms the model leans on features shared by both attack types. The plots are the place to **verify the `sttl` leakage smell** carried forward from Task 1/2: if `sttl` dominates the SHAP ranking, that is the report's evidence to discuss it. GNNExplainer output is thin because the temporal graph itself is weak (Task 2) — little structure to attribute.

---

## 2. Host-grouped cross-validation (baseline robustness)

**Method.** 5-fold `GroupKFold` on the XGBoost baseline, **grouped by host** so no host appears in both train and validation — the honest generalization estimate (a plain k-fold would leak host identity through many flows). XGBoost `n_estimators=300, max_depth=8, lr=0.1`, sqrt-damped class weights reused from `fitted_transforms`.

**Result.**

| Fold | Macro-F1 | Accuracy |
|---|---|---|
| 0 | 0.583 | 0.971 |
| 1 | 0.567 | 0.978 |
| 2 | 0.617 | 0.977 |
| 3 | 0.602 | 0.982 |
| 4 | **0.200** | 0.9999 |
| **Mean** | **0.514 ± 0.176** | 0.982 ± 0.011 |

**Understand.** Folds 0–3 cluster tightly (~0.57–0.62 macro-F1), close to the single-split baseline (0.604). Fold 4 collapses: a host group holding almost one class only → accuracy 0.9999 but macro-F1 0.20. That single fold drives the large ±0.176 spread. This is the tiny-graph / hub-heavy warning from Task 2 showing up numerically — with only 49 hosts, one grouping can strand a class. Report the mean **with** the spread; do not quote a single fold.

---

## 3. Ablation — GAT design choices

**Method.** Retrain the GAT under three configurations on the temporal graph, single run each, evaluate on the 47-host test graph: dropout 0.0 vs 0.3 (regularization), and a **GCN variant that ignores edge features** (isolates whether the edge attributes and attention actually help).

**Result.**

| Variant | Macro-F1 | ROC-AUC |
|---|---|---|
| GAT, dropout 0.0 | 0.530 | 0.632 |
| GAT, dropout 0.3 | 0.530 | **0.978** |
| GCN, no edge features | 0.507 | 0.567 |

**Understand.** Two takeaways: (1) **dropout is doing real work** — macro-F1 is flat but ROC-AUC jumps 0.63→0.98, i.e. without dropout the model's ranking/confidence is unreliable even when hard labels look similar; (2) **edge features + attention beat plain GCN** — GCN is worst on both metrics, so the edge attributes carry signal. Caveat: these are **single runs on 47 test nodes**, and Task 2's multi-seed temporal ROC-AUC sat at ~0.48–0.66 — so the 0.978 is likely a lucky seed, not a stable result. Treat the ranking (dropout helps, edge features help) as the finding, not the absolute 0.978.

---

## 4. Significance test — GNN vs baseline

**Method.** Make the two Task 2 models **comparable on the same units**: collapse XGBoost's 10-class flow predictions to binary (`≠ Normal` = attack), aggregate flow predictions up to **host level** (a host = attack if any of its flows is predicted attack), and align to the 47 test hosts the GAT scores. Then a **McNemar exact test** (paired, appropriate for small n) on the two models' per-host correctness.

**Result.**

| | GNN correct | GNN wrong |
|---|---|---|
| **Baseline correct** | 38 | 9 |
| **Baseline wrong** | 0 | 0 |

Statistic 0.0000, **p = 0.0039** → significant at α=0.05. The aggregated XGBoost baseline is correct on **all 47/47** hosts; the temporal GAT is correct on **38/47**. Discordant pairs run 9-to-0 entirely in the baseline's favour — the GNN never rescues a host the baseline gets wrong.

**Understand.** The honest headline: **the per-flow baseline, aggregated to hosts, significantly beats the GAT — the graph model adds nothing it misses.** This is consistent with Task 2 (the temporal GAT was near-random). Two caveats keep it fair: (1) the test uses the **temporal** GAT — Task 2's stronger *contact* GAT was not entered into this paired test, so "GNN loses" is really "temporal GNN loses"; (2) **n = 47** is underpowered — a 9-vs-0 split reaches significance but on a tiny sample, so state it as suggestive, not conclusive. The report's fair claim: on this dataset a well-tuned gradient-boosting baseline is the stronger detector, and the relational model is not yet justified.

---

## Key caveats for the report

- **Baseline wins the head-to-head:** McNemar p=0.0039, baseline 47/47 vs temporal GAT 38/47. Report this honestly — it is the central Task 3 finding.
- **Temporal, not contact:** the significance test used the weaker temporal GAT; note that the contact GAT (Task 2's better variant) was not tested here.
- **Underpowered:** n=47 hosts — significance on a small sample; frame as suggestive.
- **CV variance:** mean macro-F1 0.514 ± 0.176; one fold collapses to 0.20 — quote mean **and** spread, driven by the 49-host limit.
- **Ablation is single-run:** the 0.978 ROC-AUC is likely seed luck; the reliable finding is the *ranking* — dropout helps, edge features help (GAT > GCN).
- **Explainability = plots:** SHAP/LIME/GNNExplainer outputs are the six saved PNGs; use the SHAP ranking to check the `sttl` leakage smell flagged in Task 1/2.
