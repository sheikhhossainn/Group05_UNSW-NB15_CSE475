# UNSW-NB15 — Task 2 Summary

Group 5 · CSE475 · Track 1 (GNN)

Task 2 turns the raw flows explored in Task 1 into a trained model. Four notebooks, run in order:

| # | Notebook | Role |
|---|---|---|
| 1 | `Preprocessing` | Clean + split + scale flows; sequester identifiers; compute class weights |
| 2 | `Graph_construction` | Build host-level graphs (temporal + contact) from the processed flows |
| 3 | `baselines` | Five non-graph models on the processed flows — the score to beat |
| 4 | `Proposed_Method` | GAT on the host graph — the research contribution |

Each section below records the **methodology**, the **result**, and **what the result means**. `RANDOM_SEED = 42` throughout. Every fitted transform (fill values, encoder, scaler, class weights) is fit on **train only** and applied to test — no leakage.

---

## 1. Preprocessing

**Method.** Load the four raw CSVs with the schema header, then a single pipeline: global exact-duplicate removal → normalize `attack_cat` → sequester `srcip/sport/dstip/dsport/Stime/Ltime` into a **side table** linked by `flow_id` (so identifiers leave the feature matrix but stay recoverable) → **time-based split** (sort by `Stime`, no shuffle, 0.8 cutoff) → per-split clean (missing-fill → encode → scale → drop columns) → class weights from the train split → save. Split is time-based rather than host-based because only 49 hosts exist and a host split would strand whole rare classes in one side.

**Result.**

| Step | Outcome |
|---|---|
| Dedup | 2,540,047 → **2,059,415** rows (480,632 / 18.92% removed); `flow_id` = row index |
| `attack_cat` | `Backdoor`→`Backdoors`, blank/NaN→`Normal` → **10 clean classes** |
| Split | Train **1,647,532** / Test **411,883** |
| Fill | `is_ftp_login`, `ct_flw_http_mthd` → 0 (structural: not FTP/HTTP); other numeric → train median; categorical → `'unknown'` |
| Encode | `proto/service/state` OrdinalEncoder, unseen → −1 (proto 135, service 13, state 16 categories) |
| Scale | 37 numeric columns, StandardScaler (train-fit) |
| Drop | `is_sm_ips_ports` (near-constant) + 11 high-correlation columns (`dwin`, `Dpkts`, `dloss`, `sloss`, `synack`, `ackdat`, `ct_state_ttl`, `ct_src_dport_ltm`, `ct_dst_ltm`, `ct_srv_dst`, `ct_dst_src_ltm`); `sttl` **kept** (predictive, not redundant) |
| Final shape | Train **(1,647,532, 32)** / Test **(411,883, 32)** |

Class weights (train only): binary `{0: 0.521, 1: 12.44}`; multiclass `attack_cat` from Worms **1432.6** down to Normal **0.104**. Label balance: train 96.0% / 4.0%, test 91.9% / 8.1%.

Outputs (pkl + csv): `train_processed`, `test_processed`, `side_train`, `side_test`, `fitted_transforms`.

**Understand.** The split is the leakage-critical decision: time-based + train-only fitting means test is genuine future data. The side table is what makes Task 2's graph possible — features stay clean, but `flow_id` re-attaches IPs and timestamps when needed. Class weights (not SMOTE) are chosen deliberately so graph topology stays intact for notebook 2. One thing to flag: test missingness on the FTP/HTTP columns is far higher than train (`is_ftp_login` 97.9% vs 37.1%) — the time split put a different traffic mix in test, which partly explains why test is harder.

---

## 2. Graph construction

**Method.** Consume the preprocessing outputs; **nodes = hosts (IPs)**. Merge `train_processed`/`test_processed` with the side table on `flow_id` (validated one-to-one), then expand each flow into two host-events (source, destination). Build **two graphs per split**:
- **Temporal transition graph** (main) — directed. Within each host's timeline, consecutive flows inside a **300 s window** create an edge from the previous flow's partner to the current flow's partner ("via" the shared host).
- **Contact graph** (comparison) — undirected `A↔B` collapsed; plain "who talked to whom".

Node features per host: `num_connections`, `avg_duration`, `total_bytes`, `num_as_src`, `num_as_dst`, `num_unique_partners`. A leakage check confirms no train/test edge crossover; node indices are remapped to contiguous ints for PyG.

**Result.**

| | Train | Test |
|---|---|---|
| Nodes (active / total) | 49 / 49 | 47 / 49 (missing `127.0.0.1`, `192.168.241.243`) |
| Temporal edges | 2,678 (302 self-loops) | 2,634 (298 self-loops) |
| Contact edges | 169 | 161 |
| Avg temporal degree | 109.3 | 112.1 |
| Attack rate (flows) | 4.02% | 8.11% |

Temporal edge features: `time_gap`, `proto_changed`, `port_changed`, `byte_diff`, `pkt_diff`, `transition_count`. Diagnostic: **100% of consecutive per-host flow pairs fall inside the 300 s window** (train p99 gap = 6 s, test = 10 s). Outputs: `nodes_*`, `edges_temporal_*`, `edges_contact_*`, `graph_build_config`.

**Understand.** Confirms Task 1's warning — the graph is **tiny and hub-heavy** (49 nodes carrying millions of flows). Because inter-flow gaps are almost all a few seconds, the 300 s window swallows nearly everything, so the temporal edges are dense but the *timing* signal inside them is weak. Building both graphs is the point: it lets notebook 4 test whether temporal transitions actually beat plain connectivity.

---

## 3. Baselines

**Method.** Target = `attack_cat` (**10-class, per-flow**). Train five models on `train_processed`, evaluate on `test_processed`, rank by **macro-F1** (imbalance makes accuracy meaningless — see Task 1). Class weights reused from `fitted_transforms`, square-root-damped. Kernel SVC does not scale to 1.6M rows, so `LinearSVC` (calibrated on a 200k stratified subsample) substitutes — **stated in the report**. `ct_ftp_cmd` coerced to numeric (raw data held literal spaces).

**Result.**

| Model | Accuracy | Macro-F1 | ROC-AUC | Train (s) |
|---|---|---|---|---|
| **XGBoost** | 0.965 | **0.604** | 0.996 | 490 |
| RandomForest | 0.972 | 0.595 | 0.990 | 283 |
| MLP | 0.965 | 0.460 | 0.994 | 2535 |
| LogisticRegression | 0.948 | 0.408 | 0.987 | 583 |
| LinearSVM | 0.954 | 0.352 | 0.987 | 134 |

**Baseline to beat: XGBoost, macro-F1 = 0.604.** Per-class, XGBoost still struggles on the rare/overlapping classes (Analysis F1 0.23, Backdoors 0.25, DoS 0.35) while Normal and Generic are near-perfect. MLP's rare-class recall collapses (Analysis 0.04, DoS 0.05) because `MLPClassifier` accepts no class weights — a known limitation, not a bug. Saves best-model predictions, the model object, and the label encoder for the significance test and Task 3 (SHAP/LIME).

**Understand.** Tree ensembles win because they handle mixed-scale features and imbalance best. The 0.95–0.97 accuracy vs 0.35–0.60 macro-F1 gap is the whole story: models nail the majority classes and still miss the rare attacks even with weighting. Confusion concentrates exactly on the classes Task 1's 2-D projections showed overlapping (Analysis / Backdoors / DoS / Fuzzers) — the motivation for a relational model.

---

## 4. Proposed method — GAT

**Method.** Two-layer **GATv2Conv** with edge features, attention `heads=2`, `hidden_dim=8`, dropout 0.3, ELU, Adam (`lr=1e-3`, `wd=5e-4`), up to 200 epochs with early stopping (patience 20), weighted cross-entropy (sqrt inverse-frequency). Node and edge features re-scaled (train-fit). Labels are **per-host binary**: a host = 1 if it appears in *any* attack flow in its own split. Trained on both graphs (temporal = main, contact = comparison), across 5 seeds plus a final early-stopped seed-42 run. Train/val split = 40 / 9 nodes; test = 47 nodes.

**Result.** Host-label balance: train 71/29, test 70/30. Across seeds:

| Graph | Macro-F1 range | ROC-AUC range |
|---|---|---|
| Temporal (main) | ~0.38 – 0.72 | ~0.48 – 0.66 |
| Contact (comparison) | ~0.70 – 0.88 | ~0.88 – 0.93 |

The temporal model is unstable and near-random (ROC-AUC ≈ 0.5); its confusion matrices show it collapsing toward predicting "attack" for almost every host (attack recall 1.0, normal recall low). The **contact graph consistently wins** (macro-F1 up to ~0.88, ROC-AUC ~0.93). Saves the temporal model, per-node predictions, and the metrics bundle.

**Understand.** The "comparison" graph beats the "main" one — a real finding worth reporting honestly. Two reasons: (1) the 300 s window makes temporal edges carry almost no timing signal (section 2), leaving noise plus self-loops; (2) a 49-node graph with only 40 training nodes gives huge seed-to-seed variance. Plain host connectivity turns out to be the stronger relational signal here.

**Critical caveat — not an apples-to-apples comparison.** The baselines score **per-flow, 10-class** macro-F1; the GAT scores **per-host, binary** macro-F1. The numbers cannot be compared directly. For a fair claim against the XGBoost baseline, the baseline's flow predictions must be aggregated to host level (or the GAT reformulated to multiclass / edge-level). State this explicitly in the report.

---

## Key caveats for the report

- **Scope mismatch:** baseline = per-flow 10-class; GAT = per-host binary. Aggregate before comparing.
- **Temporal window:** 300 s includes ~all traffic, so temporal edges add little over contact edges.
- **Tiny graph:** 49 nodes → high variance; report multi-seed spread, not a single run.
- **SVM substitution:** `LinearSVC` on a 200k subsample stands in for kernel SVC.
- **MLP imbalance:** no class-weight support → weak rare-class recall by design.
- **`sttl` kept:** flagged as a leakage smell in Task 1; retained as predictive, watch its influence in Task 3.
