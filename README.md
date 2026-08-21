# Group 5 — UNSW-NB15 — CSE475 (Summer 2026)

**Track:** Track 1 — Graph Neural Network (GNN) on table-style data
**Dataset:** [UNSW-NB15](https://research.unsw.edu.au/projects/unsw-nb15-dataset) (network intrusion detection)
**Task we solve:** per-flow **10-class** attack categorisation (`attack_cat`), scored by **macro-F1**

This project asks a focused research question: **does modelling network flows as a graph add classification signal beyond the flow's own features?** We answer it honestly. We build a graph neural network (**Flow-SAGE**), compare it against a strong tabular baseline (**XGBoost**) and against published GNN intrusion-detection studies, and we report where the graph helps, where it does not, and where our model stands relative to the literature.

**Headline results (per-flow, 10-class, macro-F1):**

| Model | Macro-F1 | Note |
|---|---:|---|
| XGBoost baseline | **0.6042** | strongest overall — our own baseline |
| **Flow-SAGE (proposed GNN)** | **0.4706** | beats the strongest comparable published result (nCMD, 0.374) |
| Flow-SAGE without graph edges | 0.4470 | features-only floor — graph adds **+0.0236** |
| Previous host-node GNN | 0.3877 | earlier design, superseded |

Two honest conclusions follow, and we defend both: **(1)** on this tabular dataset a gradient-boosted tree still wins, but **(2)** the graph provides real, measurable signal for temporally-clustered attacks, and against studies that measure the *same* task on a *comparable* split, Flow-SAGE is competitive-to-winning.

---

## 1. Problem and task definition

Each row of UNSW-NB15 is one **network flow** — a summary of the packets exchanged between a source and a destination. Every flow carries a label `attack_cat` with **10 possible values**: `Normal` plus 9 attack families (`Generic`, `Exploits`, `Fuzzers`, `DoS`, `Reconnaissance`, `Analysis`, `Backdoors`, `Shellcode`, `Worms`).

We classify each flow into one of these 10 classes and we score with **macro-F1**, not accuracy. The dataset is extremely imbalanced (majority-to-minority ratio ≈ **12,752 : 1**; ~87% of traffic is `Normal`). A model that predicts `Normal` for everything already scores ~87% accuracy while catching zero attacks. Macro-F1 weights every class equally, so it reflects performance on the rare attacks (`Worms`, `Shellcode`, `Backdoors`, `Analysis`) that actually matter for intrusion detection.

---

## 2. Repository layout

```
README.md                                  ← this file (start here)

code/
  task1/  Group05_UNSW-NB15_task1_eda.ipynb          EDA notebook
          EDA_explained.md                            per-section EDA writeup
  task2/  Group05_UNSW-NB15_task2_Preprocessing.ipynb clean + split + scale + class weights
          Group05_UNSW-NB15_task2_Graph_construction.ipynb  build graphs from flows
          Group05_UNSW-NB15_task2_baselines.ipynb     five non-graph baselines
          Group05_UNSW-NB15_task2_Proposed_Method.ipynb     initial host-node GAT
          Group05_UNSW_NB15_task2_flowsage.ipynb      final proposed model (Flow-SAGE)
          Task2_explained.md                          per-notebook Task-2 writeup
  task3/  Group05_UNSW-NB15_task3_Explainability.ipynb        SHAP / LIME on XGBoost, GNNExplainer on host-node GAT
          Group05_UNSW-NB15_task3_FlowSAGE_Explainability.ipynb  SHAP + LIME on Flow-SAGE (fixed-neighbourhood wrapper)
          Group05_UNSW-NB15_task3_Improvement_Adaption.ipynb  CV, ablation, significance test
          Task3_explained.md                          per-notebook Task-3 writeup

related_work/
  Group05_report_comparison_and_explanation.md   ← authoritative results + comparison + design justification
  Group_05_UNSW_NB15_Related_Work.docx           5-paper related-work table
  papers/                                         paper PDFs

data/    (not tracked — see "How to run")
```

Trained artifacts (processed frames, fitted transforms, model checkpoints, saved predictions) are produced by the notebooks at runtime and are **not** committed. Raw CSVs and `data/` are git-ignored.

> **Where the definitive numbers live:** the single authoritative writeup of the final model, its results, the fair comparison with related work, and the design justification is [`related_work/Group05_report_comparison_and_explanation.md`](related_work/Group05_report_comparison_and_explanation.md). The `*_explained.md` files under `code/` document each notebook's methodology in lab-notebook detail, including the earlier host-node iteration.

---

## 3. Task 1 — Exploratory Data Analysis

*Full detail: [`code/task1/EDA_explained.md`](code/task1/EDA_explained.md).*

We explored the four raw flow files (2,540,047 flows, 49 columns) rather than the pre-split train/test CSVs, because the raw files keep `srcip`, `sport`, `dstip`, `dsport`, `Stime`, `Ltime` — the identifiers and timestamps we need for a leakage-safe split and for graph construction. The EDA drove every downstream design decision:

| EDA finding | Design consequence in Task 2 |
|---|---|
| **18.92% exact-duplicate rows** | Removed before splitting — otherwise identical rows leak between train and test. |
| **Class imbalance 12,752 : 1** | Score with macro-F1; use train-only class weights (not SMOTE, so graph structure stays intact). |
| **`sttl` – `Label` correlation = 0.904** | Flagged as a **leakage smell**: one raw feature nearly predicts the target. Reported and watched, not silently exploited. |
| **~11 feature pairs with \|r\| > 0.9** | Drop one column per redundant pair, keep the one more correlated with the label. |
| **Heavy right-skew** on byte/rate/duration columns | Standard-scale (train-fit) so no feature dominates by magnitude. |
| **Class overlap** (Exploits / Fuzzers / DoS / Reconnaissance) in PCA/t-SNE/UMAP | These overlapping classes are exactly where a **relational** model could help — the research motivation. |
| **Only 49 unique hosts** carry all 2.54M flows | A host-level graph is tiny and hub-heavy — this warning is why we ultimately moved to a **flow-level** graph. |

---

## 4. Task 2 — the model

Task 2 turns the explored flows into trained models. Every fitted transform (fill values, encoders, scaler, class weights) is fit on **train only** and applied to test — no leakage.

### 4.1 Preprocessing

Global exact-duplicate removal (2,540,047 → 2,059,415 rows), `attack_cat` normalisation to 10 clean classes, identifiers sequestered into a side table linked by `flow_id` (so features stay clean but IPs/timestamps are recoverable for the graph), a **time-based split** (sort by `Stime`, 80/20, no shuffle — test is genuine future traffic), then per-split fill → encode → scale → drop-redundant-columns, and train-only class weights.

### 4.2 Baseline — XGBoost

We trained five non-graph models on the per-flow 10-class task and ranked by macro-F1:

| Model | Macro-F1 |
|---|---:|
| **XGBoost** | **0.6042** |
| RandomForest | 0.595 |
| MLP | 0.460 |
| LogisticRegression | 0.408 |
| LinearSVM | 0.352 |

**XGBoost at 0.6042 is the score to beat.** Tree ensembles win because they handle mixed-scale features and imbalance well. Even so, XGBoost still struggles on the rare/overlapping classes — the same classes the EDA projections showed tangled together.

### 4.3 Why the graph, and why the design changed

Network attacks rarely occur as isolated events. Scans, probes and floods appear as **bursts of related flows** from the same host within a short window. A tabular model treats each flow independently and cannot see this structure. That is the gap a graph can fill.

Our **first** graph design used **hosts as nodes** (49 nodes) and classified each host as attack/normal. This had a fatal comparison problem: it scored *per-host, binary*, which cannot be compared apples-to-apples with the *per-flow, 10-class* baseline, and the 49-node graph was too small for stable training (macro-F1 **0.3877**).

Our **final** design, **Flow-SAGE**, fixes this by making **each flow a node**. Now the GNN and XGBoost solve the *identical* task — per-flow, 10-class — so every comparison is fair. Moving from the host-node to the flow-node representation improved macro-F1 by **+0.083** (0.3877 → 0.4706). This representation change is one of the central design contributions of the project.

### 4.4 How Flow-SAGE works

*Full detail: [`related_work/Group05_report_comparison_and_explanation.md`](related_work/Group05_report_comparison_and_explanation.md).*

- **Nodes** — one per network flow, carrying that flow's **29 features**.
- **Edges** — each flow is connected to a small number (**K = 5**) of other flows that occur closest to it **in time** and share either the **same source host** or the **same destination host**. Edges are built from time and host identity **only** — labels never influence the graph structure.
- **Model** — a **two-layer GraphSAGE** (two `SAGEConv` layers). GraphSAGE learns by *sampling and aggregating* from a node's neighbours, which scales to a graph of millions of flow-nodes where full-graph convolutions would not.
- **Residual head** — the classifier head concatenates the learned neighbourhood embedding with the flow's **original features** (`[h, x]`). This lets the model keep the raw per-flow signal while *adding* neighbourhood context, rather than replacing one with the other.
- **Training** — weighted cross-entropy (train-only weights) against the 10-class imbalance; **separate graphs for train and test** so a test node can never read information from a training node.

### 4.5 Results

| Model | Macro-F1 (10-class) |
|---|---:|
| XGBoost baseline | **0.6042** |
| **Flow-SAGE** | **0.4706** |
| Flow-SAGE without graph | 0.4470 |
| Previous host-node GNN | 0.3877 |

Flow-SAGE is **stable across seeds**: macro-F1 **0.4646 ± 0.0029** over seeds 1, 2, 3.

Flow-SAGE is below XGBoost by **0.1337**, and a **McNemar test confirms this gap is statistically significant** (χ² ≈ 3395, p ≈ 0). We do not hide this. Our objective was to measure whether graph structure adds signal, not to claim the GNN is the best possible detector — and on that question the answer is a qualified *yes* (next section).

### 4.6 Did the graph actually help?

To isolate the graph's contribution we trained a **second model from scratch with the graph edges removed** — same architecture, same features, no neighbours. This is the correct features-only floor.

- Flow-SAGE **with** graph: **0.4706**
- Flow-SAGE **without** graph: **0.4470**
- **Graph contribution: +0.0236** overall

The overall gain is modest but it is **highly class-dependent**:

| Class | Graph gain (with − without) |
|---|---:|
| **Reconnaissance** | **+0.131** |
| Exploits | +0.037 |
| Shellcode | +0.036 |
| Analysis | +0.029 |
| Fuzzers | +0.025 |
| Backdoors | **−0.043** |

The biggest win is **Reconnaissance** (+0.131) — exactly the attack that manifests as bursts of scans from one host in a short window, which is precisely what our temporal/host edges connect. The graph *hurts* Backdoors slightly (−0.043); we report this rather than hide it, because it explains the model's real behaviour.

> **Note on an earlier mistake we corrected.** Our first ablation removed edges only at *test* time while keeping graph-trained weights. That produced a misleadingly large "+0.249" gain — it measured a broken model, not a features-only floor. The honest, from-scratch ablation gives **+0.0236**. We document this correction openly.

---

## 5. Task 3 — interpretation and validation

*Full detail: [`code/task3/Task3_explained.md`](code/task3/Task3_explained.md).*

Task 3 trains no new production model; it **interprets and stress-tests** the Task-2 models.

- **Explainability.** SHAP (`TreeExplainer`) and LIME on the tabular XGBoost — the textbook flat-feature case — and **GNNExplainer** (Ying et al., NeurIPS 2019) on the earlier host-node GAT. Because SHAP/LIME assume independent feature vectors and cannot see message-passing, a dedicated notebook ([`FlowSAGE_Explainability.ipynb`](code/task3/Group05_UNSW-NB15_task3_FlowSAGE_Explainability.ipynb)) applies **SHAP + LIME to the proposed Flow-SAGE model itself** via a fixed-neighbourhood black-box wrapper: for a target flow we freeze its real 2-hop neighbourhood and perturb only its own 29 features, so the attributions faithfully rank that flow's own-feature pathway (`x` in the residual `[h, x]` head). These attributions explain the own-feature pathway only — the graph's contribution stays quantified by the **+0.0236** ablation, never by a SHAP value. We explain one correct and one wrong prediction for each model; the XGBoost explainers confirm the `Reconnaissance → Exploits` confusion the EDA and confusion matrices already flagged, and we use the SHAP ranking to re-check the `sttl` leakage smell carried from Task 1.
- **Robustness — host-grouped cross-validation.** 5-fold `GroupKFold` grouped by host (no host in both train and validation) gives mean macro-F1 **0.514 ± 0.176**. Four folds cluster near the single-split baseline; one fold collapses because a host group holds almost one class only. We report the **mean with its spread** — the large variance is the numeric signature of the 49-host limitation.
- **Significance test.** A **McNemar** paired test (the required internal model-vs-baseline check) confirms XGBoost significantly outperforms Flow-SAGE (χ² ≈ 3395, p ≈ 0). Honest headline: the tuned tree baseline is the stronger detector on this dataset.

---

## 6. Comparison with related work

*Full detail and the exact claim wording: [`related_work/Group05_report_comparison_and_explanation.md`](related_work/Group05_report_comparison_and_explanation.md).*

Many UNSW-NB15 papers report 95–99%. **These are usually not comparable to our result**, and treating them as if they were would be dishonest. Before comparing, we normalise for four things:

1. **Task** — binary attack-vs-normal is easier than our 10-class problem.
2. **Metric** — we use macro-F1; accuracy/weighted-F1 stay high even when rare classes fail.
3. **Dataset version** — 49-feature CSV vs NetFlow vs different partitions differ in difficulty.
4. **Split** — the default partition contains duplicates that leak; we deduplicated and split by time/host.

**We only claim to beat a study when it reports 10-class UNSW-NB15 macro-F1 on a comparable, leakage-safe split.** Under that rule:

| Comparable study (10-class UNSW macro-F1) | Macro-F1 | vs Flow-SAGE (0.4706) |
|---|---:|---|
| **nCMD** (2026 preprint, 49-feature canonical split) | **0.374** | **we beat by +0.097** |
| Federated Naive Bayes (2026 preprint) | 0.211 | we beat |
| IGRF-RFE (Yin et al., 2023) | ~0.40–0.44* | comparable / narrow beat |
| Sarhan et al. (2020), NetFlow variant, 5-fold CV | ~0.59* | above us, but different data version |

\* estimated from published per-class F1 (not an author-reported macro-F1).

**The strongest strictly-comparable published result we could access is nCMD at 0.374; Flow-SAGE beats it by +0.097 on a cleaner split.** We deliberately do **not** compare our macro-F1 against the 0.92–0.99 binary/accuracy numbers, because those measure a different task.

We also **corrected two dataset errors** in our related-work document, verified against the source papers: E-GraphSAGE uses BoT-IoT / NF-BoT-IoT / ToN-IoT (not UNSW-NB15), and Yang's topology study uses NSL-KDD / CICIDS2017 (not UNSW-NB15).

---

## 7. Why our results can be trusted

Our numbers are *lower* than leaky studies precisely because we removed the leakage. The controls:

1. **Separate train/test graphs** — a test flow-node connects only to other test nodes; it cannot read a training node.
2. **Duplicate removal** — no identical row appears in both splits.
3. **Train-only fitting** — fill values, encoders, scaler and class weights are computed from train data only.
4. **Labels never build edges** — graph structure uses time and host identity only.
5. **Time-based split** — test is genuine future traffic, not a random shuffle.

We accept lower scores as the price of an honest evaluation.

---

## 8. Limitations and conclusion

- **XGBoost still wins** on this tabular dataset (0.6042 vs 0.4706), significantly so.
- **Rare classes stay hard.** Flow-SAGE per-class F1 for DoS / Worms / Backdoors / Shellcode = **0.07 / 0.19 / 0.15 / 0.27**; XGBoost = **0.35 / 0.54 / 0.25 / 0.51**.
- **Small host universe (49 hosts)** drives high cross-validation variance.
- **Leakage-safe split** yields lower headline numbers than the default partition — an accepted trade-off.

**Conclusion.** Graph structure carries real, measurable signal beyond the raw flow features — strongest for temporally-clustered attacks like Reconnaissance (+0.131) — and Flow-SAGE beats the strongest comparable published result on the same task. But the gain is modest overall, and a well-tuned gradient-boosted tree remains the stronger detector on UNSW-NB15. We report this outcome as a genuine, honestly-measured finding rather than dressing up a leaky high score.

---

## 9. How to run

1. Download the UNSW-NB15 raw flow CSVs (`UNSW-NB15_1.csv`–`_4.csv`) and the schema file `NUSW-NB15_features.csv` from the [dataset page](https://research.unsw.edu.au/projects/unsw-nb15-dataset), and place them under `data/` (git-ignored, not shipped with this repo).
2. Run the notebooks in order:
   - `code/task1/` — EDA.
   - `code/task2/` — `Preprocessing` → `Graph_construction` → `baselines` → `Proposed_Method` (initial host-node GAT) → `flowsage` (final Flow-SAGE model).
   - `code/task3/` — `Explainability` → `FlowSAGE_Explainability` → `Improvement_Adaption`.
3. The graph notebooks require `torch` and `torch_geometric`; the baselines require `xgboost`, `scikit-learn`; explainability requires `shap`, `lime`.

Each notebook is paired with a `*_explained.md` file recording its methodology, results, and interpretation.
