# Group 05 — UNSW-NB15 — Presentation Script & Q&A Prep

**CSE475 · Track 1 (GNN) · 20-minute slot (≈5–7 min talk + Q&A)**

How to use this file: Section A is the spoken walkthrough — read it almost word for word, it is timed for about six minutes. Section B is the question bank — the answers are written the way you should say them out loud. Section C is a one-line cheat sheet to memorise the night before.

---

## SECTION A — The Walkthrough (spoken script, ≈6 minutes)

### 0. Opening — 20 seconds

"Good morning. We are Group 5. Our project is on the UNSW-NB15 network intrusion dataset, and we are on Track 1 — graph neural networks on tabular data. Our research question is simple to state: **does turning network traffic into a graph give us any classification signal that a normal tabular model cannot already get?** We answer that question honestly, and I'll show you exactly where the graph helps and where it does not."

### 1. The problem and why the metric matters — 45 seconds

"Every row in UNSW-NB15 is one network flow — a summary of the packets between a source and a destination. Each flow has a label with **ten classes**: Normal plus nine attack families. Our job is to put each flow into the right one of those ten classes.

The single most important fact about this dataset is that it is **massively imbalanced**. About 87% of all traffic is Normal, and the ratio between the biggest and smallest class is roughly **twelve thousand to one**. Worms, the rarest attack, has only 174 examples out of 2.5 million.

Because of that, **we do not use accuracy**. A model that labels everything 'Normal' already scores 87% accuracy while catching zero attacks. We use **macro-F1**, which gives every class equal weight, so our score actually reflects how well we detect the rare attacks — which is the whole point of intrusion detection."

### 2. EDA — what we found before touching a model — 50 seconds

"We started with exploratory analysis on the raw flow files — 2.5 million flows, 49 columns. Four findings shaped everything we did afterwards.

**One** — almost 19% of the rows were exact duplicates. If we had left those in, identical rows would end up in both training and testing, and the model would score high just by memorising.

**Two** — the imbalance I just mentioned, which locked in our choice of macro-F1.

**Three** — one raw feature, `sttl`, was correlated with the label at 0.90. That is a **leakage smell** — a single feature almost predicting the answer. We flagged it, kept it because it is genuinely predictive, but we watched its influence instead of quietly exploiting it.

**Four** — there are only **49 unique hosts** carrying all 2.5 million flows. That told us a host-based graph would be tiny and lopsided, and — as you'll see — it is the reason we changed our whole graph design."

### 3. Preprocessing — 40 seconds

"Our preprocessing is built around one principle: **no information leaks from the future or from the test set into training.**

We removed duplicates first. We sequestered the identifiers — IP addresses and timestamps — into a separate side table so they stay out of the features but can be recovered to build the graph. We then split the data **by time**, not randomly: we sort by timestamp and the last 20% becomes the test set, so the test set is genuinely future traffic. Every transform after that — filling missing values, encoding categories, scaling, and computing class weights — is **fit on the training set only** and then applied to test. We dropped redundant and near-constant columns, and we handled imbalance with **class weights rather than SMOTE**, because SMOTE would invent fake rows that have no real place in the graph."

### 4. Baseline — why XGBoost, and why not the others — 45 seconds

"Before the graph, we ran five standard models on the exact same ten-class task. **XGBoost won with a macro-F1 of 0.60.** Random Forest was right behind at 0.59. The others were well back — the MLP at 0.46, logistic regression at 0.41, linear SVM at 0.35.

The pattern is clear. **The two tree ensembles win**, and here's why: this is tabular data with features on wildly different scales and non-linear boundaries. Trees split on individual feature thresholds, so they don't care about scale and they carve sharp, axis-aligned decision regions. The **linear models** — logistic regression and linear SVM — simply cannot draw those non-linear boundaries, so they lose. The **MLP** could in principle, but scikit-learn's MLP does not accept class weights, so it ignores the rare classes and its rare-class recall collapses. XGBoost edges out Random Forest because boosting focuses each new tree on the examples the previous ones got wrong — which are exactly the rare attacks. So **XGBoost at 0.60 is our number to beat.**"

### 5. The new idea — Flow-SAGE — 70 seconds

"Now the graph, and this is our main contribution.

Attacks are not isolated. A scan or a probe shows up as a **burst of related flows from the same host in a few seconds**. A tabular model looks at each flow alone and cannot see that burst. A graph can.

Our **first attempt used hosts as nodes** — 49 nodes — and classified each host as attack or normal. That had two problems: it scored per-host and binary, so we could not compare it fairly to the per-flow ten-class baseline; and 49 nodes is far too few to train a stable model. It scored 0.39.

So we redesigned it. In our final model, **Flow-SAGE, every flow is a node.** Now the graph model and XGBoost solve the *identical* task, so the comparison is finally fair. Just changing from host-nodes to flow-nodes lifted our score by **+0.083**, up to 0.4706.

Here is how it works. Each flow-node carries its own 29 features. We connect each flow to its **five nearest neighbours in time that share the same source or destination host** — and, importantly, we build those edges from **time and host only, never from the labels**, so the structure can't leak the answer. We use a **two-layer GraphSAGE** network, which learns by sampling and aggregating from a node's neighbours — that sampling is what lets it scale to millions of flow-nodes. Finally, our classifier head **concatenates the learned neighbourhood embedding with the flow's original features**, so the model adds context on top of the raw signal instead of washing it away."

### 6. Results and the honest verdict — 60 seconds

"So does the graph help? To answer properly, we trained a **second model from scratch with the edges removed** — same architecture, no neighbours. That is the true features-only floor. With the graph we score 0.4706; without it, 0.4470. **The graph adds 0.0236 overall** — and the result is stable across three random seeds.

But the average hides the real story. The graph's biggest win is **Reconnaissance, up 0.131** — which is exactly the burst-of-scans attack our edges were designed to capture. It slightly *hurts* Backdoors, and we report that too, because it's honest.

Now the two conclusions we stand behind. **First**, against the published literature on the *same* task — ten-class UNSW macro-F1 on a clean split — the strongest comparable result we found is a model called nCMD at 0.374, and **we beat it by nearly 0.10**. **Second**, our own XGBoost baseline still beats our GNN, 0.60 to 0.47, and a McNemar significance test confirms that gap is real.

We do not treat that as a failure. Our question was whether the graph carries signal — and it does, measurably. But on this particular tabular dataset, a well-tuned tree is still the stronger detector, and we'd rather report that truthfully than dress up a leaky high score. Thank you — happy to take questions."

*(Total ≈ 6 minutes at a calm pace. If you're short on time, section 2 and 3 can be compressed to one sentence each.)*

---

## SECTION B — Question Bank (say these out loud)

### Group 1 — Metric and evaluation

**Q: Why macro-F1 and not accuracy?**
"Because the data is 87% Normal. A model that predicts Normal for everything scores 87% accuracy and catches no attacks. Macro-F1 averages the F1 of each class equally, so a rare attack counts as much as the majority class. It measures the thing we actually care about — catching the rare attacks."

**Q: Why a time-based split instead of a random split?**
"Two reasons. First, realism — in the real world you train on past traffic and detect future traffic, so the test set should be the future, which a time split gives you. Second, leakage — with 19% duplicate rows, a random split would put identical flows in both train and test and inflate the score. Sorting by time and holding out the last 20% avoids both."

**Q: Why is your score lower than the 95–99% you see in many UNSW-NB15 papers?**
"Because most of those numbers are not the same task as ours. They're often binary attack-versus-normal, or they use accuracy, or they use the default split that still contains duplicates. We deliberately removed the leakage, so our number is lower but honest. When we compare only against studies doing the same ten-class macro-F1 on a clean split, we're actually at the top."

### Group 2 — Preprocessing

**Q: Why class weights instead of SMOTE or oversampling?**
"SMOTE creates synthetic rows by interpolating between real ones. In a tabular model that's fine, but in our graph those fake rows have no real timestamp and no real host, so they can't be placed in the graph correctly — they'd break the structure. Class weights achieve the same goal, telling the model to pay more attention to rare classes, without touching the graph."

**Q: You kept `sttl` even though it looks like leakage. Why?**
"`sttl` correlates with the label at 0.90, which is a warning sign. But it's a legitimate feature — it reflects how the traffic was generated — so dropping it would throw away real signal. Our compromise was to keep it but flag it, and in Task 3 we used SHAP to check how much the model leans on it. We report the risk rather than hide it or blindly exploit it."

**Q: How did you handle the missing values?**
"Two columns were missing 50%+ of the time — but that missingness is structural: `is_ftp_login` is only defined for FTP traffic, and the HTTP-method column only for HTTP. So 'missing' means 'not that protocol', and we fill those with zero. Everything else we fill with the training-set median for numbers and an 'unknown' category for text — again, computed from training only."

### Group 3 — Why the baseline wins

**Q: Why does XGBoost beat all the other tabular models?**
"Because this is tabular data with mixed scales and non-linear class boundaries. Trees split on one feature at a time, so scale doesn't matter and they can carve non-linear regions. Linear models — logistic regression and linear SVM — can only draw straight boundaries, so they underperform. The MLP could do better but scikit-learn's version can't take class weights, so it ignores rare classes. XGBoost slightly beats Random Forest because boosting concentrates on the hard, misclassified examples — the rare attacks."

**Q: Then why bother with a GNN at all if the tree wins?**
"Because the goal of the project isn't to win a leaderboard — it's to test a hypothesis: does relational structure between flows add information? A tree cannot use that structure by design. The only way to measure the value of the graph is to build the graph. And it turns out the graph does add signal — just not enough to overtake the tree on this dataset."

### Group 4 — Core architecture questions (the ones a professor loves)

**Q: Why two GraphSAGE layers? Why not one, why not five?**
"Each layer lets a node see one more hop of neighbours. Two layers means a flow can gather information from its neighbours and its neighbours' neighbours, which covers the local burst we care about. If we go deeper, we hit **over-smoothing** — with too many layers every node's embedding converges toward the same average and the classes become indistinguishable. We tested it; two layers gave the best macro-F1, so we stopped there."

**Q: Why K=5 neighbours per flow?**
"K controls how much context each flow sees. Attacks are local bursts, so the *nearest* few flows in time on the same host are the informative ones. A small K keeps the neighbourhood focused on genuinely related flows. A large K pulls in unrelated traffic, which adds noise and blurs the signal — and costs more compute. Five was the sweet spot between enough context and staying relevant."

**Q: Why GraphSAGE specifically — why not GCN or GAT?**
"Two reasons. Scale — GraphSAGE learns by *sampling* a fixed number of neighbours and aggregating them, so it scales to millions of flow-nodes. A plain GCN needs the full adjacency matrix, which doesn't scale here. Second, GraphSAGE is **inductive**: it learns an aggregation function rather than fixed node embeddings, which is exactly what we need because our train and test graphs are separate and the test nodes are never seen during training. We actually did try attention — GAT — on the earlier host-node design, but at flow scale the sampling of GraphSAGE is the right tool."

**Q: Why the residual head that concatenates the original features?**
"Because message passing tends to smooth a node toward its neighbours, and if the neighbourhood is unhelpful, that smoothing destroys the flow's own signal. By concatenating the raw 29 features back in at the end, we let the model *add* neighbourhood context on top of the original features rather than replace them. If the graph is useless for a given flow, the model can still fall back on the features — which is also why our with-graph and without-graph scores are close."

**Q: Why build edges from time and host, and not from anything label-related?**
"If we used labels to decide which flows connect, the graph structure itself would encode the answer — that's leakage. Edges must be built only from information a real detector would have at inference time, which is timing and host identity. That keeps the evaluation honest."

**Q: Why separate graphs for train and test?**
"If train and test flows lived in one graph, a test node could pull information across an edge from a training node during message passing — that's leakage through the graph. Keeping the graphs separate guarantees a test node only ever sees other test nodes."

### Group 5 — Why the graph underperforms in multiclass

**Q: Why does your graph model do worse than the tree on the ten-class problem?**
"Two structural reasons. First, message passing **averages** information from neighbours, which smooths decision boundaries — but tabular attack detection rewards *sharp* boundaries on individual features, which is exactly what trees give you. Second, the graph only helps when a flow's neighbours share its class — that's the homophily assumption. It holds beautifully for burst attacks like Reconnaissance, but it breaks for rare, scattered attacks like Backdoors and Worms, where neighbours are usually a different class and message passing actively misleads the model. Since macro-F1 weights those rare classes heavily, their poor performance drags the whole score down."

**Q: Why is Reconnaissance the class that improves most?**
"Because reconnaissance is textbook burst behaviour — many scan flows from the same host in a short window. That's precisely the pattern our time-and-host edges connect, so a recon flow's neighbours are usually also recon. Strong homophily means message passing helps a lot — plus 0.131 F1 for that class."

**Q: Why does the graph actually hurt Backdoors?**
"Backdoors are stealthy and spread out — they don't form tight bursts. So a backdoor flow's nearest neighbours in time are usually normal or other-attack traffic. Message passing then mixes in the wrong-class signal, and the model gets slightly worse. We report the −0.043 openly because it's part of understanding when the method works and when it doesn't."

### Group 6 — Honesty and rigour

**Q: You mentioned your first ablation was wrong. What happened?**
"Our first attempt to measure the graph's value removed the edges only at test time, while keeping weights that were trained *with* the graph. That's a broken, half-disabled model, and it gave a misleadingly huge gain of +0.25. The correct way — which we did — is to train a completely separate model from scratch with no edges at all. That honest floor gives the real graph contribution of +0.0236. We document the correction rather than quietly keeping the flattering number."

**Q: How do you know your +0.0236 gain isn't just random noise from one lucky run?**
"We ran the model over three random seeds and got 0.4646 with a standard deviation of only 0.0029 — very tight. The graph gain sits outside that noise band, and the per-class pattern — big gains exactly on the burst attacks — is a mechanistic result, not a coincidence."

**Q: How can you claim to 'beat' the literature while losing to your own baseline?**
"They're two different comparisons. The comparison against published work is on the same task — ten-class UNSW macro-F1 on a clean split — and there we beat the strongest comparable result, nCMD at 0.374. The comparison against our own XGBoost is an internal honesty check with a significance test; we lose that one and we say so. Both statements are true at the same time, and reporting both is the honest thing to do."

### Group 7 — Extensions (if asked "what next")

**Q: How would you improve the GNN?**
"A few directions: use richer edge features that capture the timing gap and protocol change between flows; add attention so the model learns which neighbours matter instead of treating all five equally; try a hybrid that feeds the GNN embedding into a tree ensemble to combine relational context with sharp boundaries; and test on a dataset with far more than 49 hosts, since our tiny host universe is a real limitation."

**Q: Why did you use the raw files instead of the official train/test split?**
"The official split drops the IP and timestamp columns and still contains duplicates. We needed those columns to build the graph and to make an honest time-based split, so we went back to the raw flow files and built the pipeline ourselves."

---

## SECTION C — One-line cheat sheet (memorise these)

- **Task:** per-flow, 10-class, macro-F1 (because 87% Normal makes accuracy meaningless).
- **New idea:** every *flow* is a graph node (not the host); edges = 5 nearest-in-time flows sharing a host; built label-free.
- **Model:** 2-layer GraphSAGE (sampling → scales; inductive → clean train/test split) + residual head that keeps the raw 29 features.
- **Numbers:** XGBoost **0.6042** > Flow-SAGE **0.4706** > no-graph **0.4470** (graph = **+0.0236**) > old host-node **0.3877**.
- **Graph win:** Reconnaissance **+0.131** (burst attack, high homophily). **Graph loss:** Backdoors **−0.043** (scattered, low homophily).
- **Literature:** beat nCMD **0.374** by +0.097 on the same task; refuse to compare against binary/accuracy papers at 0.92–0.99.
- **Why tree wins:** tabular + non-linear + mixed-scale rewards sharp tree splits; message passing smooths and hurts sharp boundaries.
- **Why 2 layers / K=5:** more layers → over-smoothing; larger K → noise; both tuned empirically.
- **Honesty:** McNemar (χ²≈3395) confirms XGBoost wins; from-scratch ablation fixed our earlier invalid +0.25; multi-seed 0.4646 ± 0.0029 proves stability.
