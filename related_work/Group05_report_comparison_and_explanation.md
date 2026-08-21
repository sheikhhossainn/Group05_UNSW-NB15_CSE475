# Group 05 — UNSW-NB15 Task 2: Results, Comparison, and Design Justification

This section presents our model, experimental results, comparison with related work, design decisions, and limitations. We focus on fair comparison and report both the strengths and weaknesses of our approach.

## 1. What We Built

We represent each network flow as a node in a graph. Each flow retains its 29 features. We connect a flow to a small number of other flows that occur close to it in time and share either the same source host or the same destination host.

We use a two-layer GraphSAGE model. The model uses information from neighbouring flows before assigning one of the 10 classes: the 9 attack categories or Normal.

We compare our graph model, which we call **Flow-SAGE**, with a strong tabular baseline, XGBoost, and with five published GNN-based intrusion detection studies.

## 2. Our Results

Our main results are:

| Model | Macro-F1 (10-class) |
|---|---:|
| XGBoost baseline | **0.6042** |
| Flow-SAGE | **0.4706** |
| Flow-SAGE without graph | **0.4470** |
| Previous host-node GNN | 0.3877 |

Flow-SAGE is also stable across different random seeds, with a macro-F1 of **0.4646 ± 0.0029** over seeds 1, 2, and 3.

Changing from the previous host-node representation to the flow-node representation improved macro-F1 by **+0.083**. This is one of the main improvements in our design.

However, Flow-SAGE performs worse than our XGBoost baseline by **0.1337** macro-F1. A McNemar test confirms that this difference is statistically significant (χ² ≈ 3395, p ≈ 0).

Therefore, our main conclusion is that gradient-boosted trees still perform better on this tabular intrusion-detection dataset. We do not treat this as a failure of the project. Instead, our objective is to measure whether graph information adds useful information beyond the original flow features, and to identify where that information helps.

## 3. Comparison with Related Work

### 3.1 Fair Comparison

Many UNSW-NB15 studies report scores between 95% and 99%. However, these results are not necessarily comparable with our result.

Before making a comparison, we consider four factors:

1. **Task:** We use 10-class classification. A binary attack-versus-normal result cannot be directly compared with our result because binary classification is an easier task.

2. **Metric:** We use macro-F1 rather than accuracy or weighted-F1. Accuracy can remain high when the model performs poorly on rare classes. Macro-F1 gives equal importance to every class and therefore reflects rare-class performance more clearly.

3. **Dataset version:** UNSW-NB15 is available in different forms, including the standard 49-feature CSV data, the NetFlow version, and different predefined train/test partitions. These versions can have different levels of difficulty.

4. **Data split:** The default UNSW-NB15 train/test partition contains duplicate rows. This can allow a model to effectively memorize examples that also appear in the test set. We removed duplicates and used a host- and time-based split to reduce this type of leakage.

For this reason, we only claim that our model outperforms another study when the study reports **10-class macro-F1 on a comparable dataset version and uses a leakage-free or sufficiently comparable split**.

### 3.2 Comparison Table

Our five originally cited papers do not all measure the same task as ours. Some use different datasets, binary classification, or different evaluation metrics.

#### Group 1 — Originally Cited Papers

| # | Paper | Task | Metric | Reported Result | Reason It Is Not Directly Comparable |
|---|---|---|---|---|---|
| 1 | E-GraphSAGE (Lo et al., 2021) | Multiclass | Weighted-F1 | ~0.85–1.00 | Uses BoT-IoT / NF-BoT-IoT / ToN-IoT rather than UNSW-NB15 |
| 2 | SAGEConv + Transformer (2026, MDPI) | Binary | Macro-F1 | 0.9749 | Attack-versus-normal classification, not 10-class classification |
| 3 | GCN + AE + KNN (2025) | Binary | F1 | 0.9993 | Binary classification on a balanced 22k/22k subset |
| 4 | Network Topology GNN (Yang, 2026) | Multiclass | Precision / FPR | N/A | Uses NSL-KDD / CICIDS2017 rather than UNSW-NB15 |
| 5 | CAGN-GAT Fusion (2025) | Binary | Macro-F1 | 0.9181 | Binary classification on a downsampled subset |

These papers are still useful as methodological references for GNN-based network intrusion detection, but their reported scores should not be directly compared with our 10-class UNSW-NB15 macro-F1.

#### Group 2 — Studies Reporting 10-Class UNSW-NB15 Macro-F1

| Paper | Dataset / Split | Macro-F1 (10-class) | Comparison with Flow-SAGE |
|---|---|---:|---|
| nCMD (Ahmad & Ahmed, 2026, preprint) | 49-feature, canonical split | **0.374** | Flow-SAGE is higher by **0.097** |
| Federated Naive Bayes (2026, preprint) | Federated, non-IID | 0.211 | Flow-SAGE is higher |
| IGRF-RFE (Yin et al., 2023) | 49-feature, deduplicated | ~0.40–0.44* | Similar or slightly lower |
| Sarhan et al. (2020) | NF-NetFlow, 5-fold CV | ~0.59* | Higher than Flow-SAGE, but uses a different data version |

\* These values are estimates calculated from the published per-class F1 values. They are not macro-F1 values explicitly reported by the authors, so we describe them as estimates.

The nCMD and Federated Naive Bayes studies are 2026 preprints and have not yet been peer-reviewed. We therefore identify them as preprints when discussing them.

The strongest strictly comparable result available to us is **nCMD with a macro-F1 of 0.374** on the standard 49-feature partition. Our Flow-SAGE result of **0.4706** is higher by **0.097**.

We also corrected two dataset descriptions in our related-work document. E-GraphSAGE uses BoT-IoT, NF-BoT-IoT, and ToN-IoT, not UNSW-NB15. The Yang topology study uses NSL-KDD and CICIDS2017, not UNSW-NB15. These corrections were verified from the respective papers.

E-GraphSAGE and the Yang study can therefore remain as methodological GNN-NIDS references. If the related-work section must contain only studies using UNSW-NB15, they can instead be replaced with nCMD, IGRF-RFE, and Sarhan et al.

### 3.3 Our Position Relative to Related Work

We use the following basis for the main comparison: **10-class UNSW-NB15 `attack_cat` macro-F1 on the standard 49-feature partition**.

Under this basis, our Flow-SAGE model achieves **0.4706**, which is higher than the strongest accessible comparable result, nCMD at **0.374**.

The higher scores reported by some other studies, such as 0.92–0.99, should not be interpreted as direct evidence that those models are better than ours. Those studies use binary classification, accuracy or weighted-F1, a different version of the dataset, or different datasets.

Therefore, our result should be stated as follows:

> Restricting the comparison to 10-class UNSW-NB15 `attack_cat` macro-F1 on the standard 49-feature partition, our proposed Flow-SAGE model achieves a macro-F1 of 0.4706, exceeding the strongest accessible comparable result, nCMD at 0.374. Our evaluation also uses a stricter leakage-free split. We do not directly compare this result with higher published scores when those studies use binary classification, accuracy, weighted-F1, different dataset versions, or different datasets. Our Flow-SAGE model remains below our XGBoost baseline at 0.6042 and the estimated NetFlow-variant Extra-Trees result of approximately 0.59.

## 4. Did the Graph Actually Help?

To measure the contribution of the graph, we trained a second model from scratch. This model uses the same architecture and features but does not contain graph edges. This gives us a proper features-only baseline.

The results are:

- Flow-SAGE with graph: **0.4706**
- Flow-SAGE without graph: **0.4470**
- Improvement from graph information: **+0.0236**

The improvement is relatively small at the overall level, but it is much larger for some classes.

| Class | Graph Gain (Graph − No Graph) |
|---|---:|
| Reconnaissance | **+0.131** |
| Exploits | +0.037 |
| Shellcode | +0.036 |
| Analysis | +0.029 |
| Fuzzers | +0.025 |
| Backdoors | **−0.043** |

The largest improvement occurs for **Reconnaissance**. This is consistent with the way reconnaissance activity occurs. Scans and probes often appear as bursts of activity from the same host within a short time period. Our graph connects such flows, allowing the model to use information from related flows.

The graph does not help every class. For example, Backdoors show a decrease of **0.043**. We report this result rather than hiding it because it is important for understanding the actual behaviour of the model.

### Earlier Ablation Error

Our first ablation experiment was incorrect. We removed the graph edges only during testing while keeping the weights learned by the graph model. This produced a macro-F1 of **0.2281** and incorrectly suggested that the graph contributed approximately **+0.249** macro-F1.

This was not a valid comparison because the no-graph model had not been trained without graph information.

We corrected the experiment by training the no-graph model from scratch. The correct result is **0.4470**, giving an actual graph contribution of **+0.0236**.

For a stronger evaluation, the no-graph model should also be tested over seeds 1, 2, and 3. This would allow us to report its mean and standard deviation and determine whether the +0.0236 improvement is stable across random seeds.

## 5. Why Our Results Can Be Trusted

We used several measures to reduce data leakage:

1. **Separate graphs for training and testing:** Test flows are connected only to other test flows. A test node cannot obtain information from a training node through the graph.

2. **Duplicate removal:** We removed duplicate rows from the dataset. This prevents the model from obtaining an artificially high score by seeing identical examples in both training and testing.

3. **Training-only class weights:** Class weights are calculated using training data only. Test labels are never used during training.

4. **Labels are not used to construct edges:** Graph edges are created using time and host information only. The class labels do not influence the graph structure.

These decisions make our evaluation more conservative than evaluations that use the default UNSW-NB15 partition with duplicated records, but they also make the reported result more reliable.

## 6. Explanation of the Main Design Questions

### Q1. Why did we choose this model and these design choices?

Network attacks often occur as related bursts from the same host within a short period. A conventional tabular model treats each flow independently and cannot directly use this relational information.

We therefore represent each flow as a node and connect flows that are close in time and share a source or destination host.

We chose GraphSAGE because it learns from sampled neighbouring nodes and is suitable for graphs containing a large number of flows. We also retain the original flow features alongside the graph representation through a residual head. This allows the model to preserve the original feature information while also using information from neighbouring flows.

### Q2. What worked and what did not?

The flow-node representation worked better than our previous host-node representation. It improved macro-F1 by **+0.083**.

The graph also added useful information for burst-based classes. The largest improvement was observed for Reconnaissance, with a gain of **+0.131**.

However, the overall Flow-SAGE score of **0.4706** is still lower than the XGBoost score of **0.6042**.

Rare classes also remain difficult. In particular, DoS, Worms, Backdoors, and Shellcode have relatively low F1 scores. The graph also reduces performance for Backdoors.

### Q3. Why does the model behave this way?

XGBoost works well on tabular data because it can directly learn sharp decision boundaries from individual features.

Our graph provides an advantage when neighbouring flows contain useful shared patterns. This is particularly useful for activities such as reconnaissance scans, where multiple related flows occur close together in time and often involve the same host.

However, graph information is less useful when a class is rare or its flows do not form a clear local pattern. Since macro-F1 gives equal importance to every class, poor performance on rare classes has a strong effect on the final score.

This explains why the graph model can provide useful additional information while still performing worse than XGBoost overall.

### Q4. Is the comparison fair, and where do we stand?

We compare results only when the task and metric are sufficiently comparable. Our main evaluation uses **10-class UNSW-NB15 macro-F1**.

Under this basis, our Flow-SAGE model achieves **0.4706**, compared with **0.374** for nCMD, the strongest accessible comparable result identified in our review.

Our model therefore performs better than that result under the stated comparison basis. At the same time, it performs worse than our own XGBoost baseline at **0.6042** and the estimated NetFlow-variant Extra-Trees result of approximately **0.59**.

We do not compare our macro-F1 directly with binary-classification or accuracy results from the five originally cited papers because those evaluations measure different tasks or use different metrics.

## 7. Limitations

Our work has several limitations:

- XGBoost still performs better than Flow-SAGE on this tabular dataset.
- Several rare classes remain difficult to classify. Our F1 scores for DoS, Worms, Backdoors, and Shellcode are **0.07, 0.19, 0.15, and 0.27**, respectively. XGBoost achieves **0.35, 0.54, 0.25, and 0.51** for the same classes.
- The current no-graph baseline has been evaluated using only one random seed. It should be repeated over three seeds for a stronger comparison.
- Our leakage-free split produces lower scores than studies that use the default UNSW-NB15 partition. We accept this trade-off because avoiding leakage is more important than obtaining an artificially high score.

Overall, our results show that graph structure provides additional information beyond the original flow features, especially for attack classes with strong temporal and host-based relationships. However, the improvement is modest at the overall level, and traditional tree-based methods remain stronger on this tabular intrusion-detection task.