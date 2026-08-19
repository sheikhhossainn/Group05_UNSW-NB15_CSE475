# UNSW-NB15 — Task 1 EDA

Group 5 · CSE475 · Track 1

Exploratory analysis of the raw UNSW-NB15 flow files. Each section records the **methodology** used, the **result** it produced, and **what the result means** for Task 2. No preprocessing (dedup, split, scaling) happens here — exploration only.

---

## Dataset

| Property | Value |
|---|---|
| Dataset | UNSW-NB15 (UNSW Canberra, IXIA PerfectStorm) |
| Task | Network intrusion detection |
| Files | `UNSW-NB15_1.csv`–`_4.csv` (raw flows); schema from `NUSW-NB15_features.csv` |
| Total flows | 2,540,047 |
| Columns | 49 (40 numeric, 9 categorical) |
| Targets | `attack_cat` (10 classes), `Label` (0 = normal, 1 = attack) |
| Unique hosts | 49 |

One row = one network flow (a summary of many packets between a source and destination). The raw files are used instead of the pre-split train/test CSVs because they keep `srcip`, `sport`, `dstip`, `dsport`, `Stime`, `Ltime` — needed later for a host/time split and for graph construction. The pre-split files drop these.

---

## Setup

**Method.** The four raw CSVs have no header row, so column names are read from `NUSW-NB15_features.csv` and applied on load; the four frames are concatenated. `attack_cat` is then normalized: blank/NaN → `Normal`, whitespace collapsed, `Backdoor` merged into `Backdoors`.

**Result.** Combined frame 2,540,047 × 49. Before normalization `attack_cat` held 11 distinct strings (spacing/casing duplicates); after, exactly 10 clean classes.

**Understand.** Without normalization the class count is inflated and rare classes split further — every downstream count and the model's label set would be wrong.

---

## A. Summary table

**Method.** One-glance fact sheet: sample count, class count, numeric/categorical split, unique hosts, class balance, overall missing %, duplicate %.

**Result.**

| Metric | Value |
|---|---|
| Total samples | 2,540,047 |
| Classes (`attack_cat`) | 10 |
| Numeric features | 40 |
| Categorical features | 9 |
| Unique hosts (`srcip ∪ dstip`) | 49 |
| Class balance (majority : minority) | 2,218,764 : 174 |
| Rows with any missing value | 57.20% |
| Duplicate rows | 18.92% |

**Understand.** Three problems are visible before any plot: extreme class imbalance, heavy missingness, and a large duplicate fraction. All three are addressed in Task 2.

---

## B. Numeric statistics

**Method.** `describe()` per numeric column, extended with skew, kurtosis, and IQR outlier share (fraction outside `Q1 − 1.5·IQR … Q3 + 1.5·IQR`), sorted by skew.

**Result.** Byte/rate/duration columns (`sbytes`, `dbytes`, `Sload`, `Dload`, `dur`, `Sjit`) are heavily right-skewed (|skew| ≫ 2) with large outlier shares and means far above their medians.

**Understand.** These features span many orders of magnitude and cannot be used raw — they need log/robust scaling in Task 2. The skew is expected for network counters (most flows tiny, a few enormous), so it is a scaling issue, not corrupt data.

---

## C. Data quality

**Method.** Four checks — per-column missing %, exact duplicate rows, near-constant columns (dominant value > 99% of rows), and numeric pairs with |r| > 0.9 (upper triangle only, so each pair listed once).

**Result.**

- **Missing:** `is_ftp_login` 56.29%, `ct_flw_http_mthd` 53.08%. No other column missing.
- **Duplicates:** 480,632 rows (18.92%).
- **Near-constant:** `is_sm_ips_ports` — one value covers 99.83% of rows.
- **Highly correlated pairs (|r| > 0.9):**

  | Pair | \|r\| |
  |---|---|
  | `Stime` – `Ltime` | 1.000 |
  | `swin` – `dwin` | 0.997 |
  | `dloss` – `Dpkts` | 0.992 |
  | `dbytes` – `dloss` | 0.991 |
  | `dbytes` – `Dpkts` | 0.971 |
  | `ct_dst_ltm` – `ct_src_dport_ltm` | 0.960 |
  | `ct_srv_src` – `ct_srv_dst` | 0.957 |
  | `sbytes` – `sloss` | 0.953 |
  | `ct_srv_dst` – `ct_dst_src_ltm` | 0.951 |
  | `sttl` – `ct_state_ttl` | 0.906 |
  | `sttl` – `Label` | 0.904 |

  (further `ct_*` pairs fall in the 0.91–0.95 range)

**Understand.**
- The two ~53–56% missing columns are FTP/HTTP-specific — missing means "not that protocol", so they need indicator/zero imputation, not row deletion.
- The near-constant column carries almost no separating signal → drop candidate.
- Each correlated pair is redundant; keep the one more correlated with `Label`, drop the other. `Stime`/`Ltime` at 1.000 are effectively the same field.
- **`sttl` – `Label` = 0.904 is a leakage smell:** one raw feature almost predicts the target. It reflects how the attack traffic was generated, not a general rule — must be reported and watched in Task 2, not silently exploited.

---

## D. Class balance

**Method.** Value counts of `attack_cat` and `Label`, plus majority : minority ratio.

**Result.**

| Class | Count |
|---|---|
| Normal | 2,218,764 |
| Generic | 215,481 |
| Exploits | 44,525 |
| Fuzzers | 24,246 |
| DoS | 16,353 |
| Reconnaissance | 13,987 |
| Analysis | 2,677 |
| Backdoors | 2,329 |
| Shellcode | 1,511 |
| Worms | 174 |

Majority : minority ratio = **12,752 : 1**. `Label` is ~87% normal / ~13% attack.

**Understand.** A model that always predicts `Normal` scores ~87% accuracy while catching zero attacks, so accuracy is useless here. Macro-F1 and per-class recall are the metrics that reflect performance on the rare attacks (Worms, Shellcode, Backdoors, Analysis).

---

## E. Feature distributions

**Method.** For key flow features (`dur`, `sbytes`, `dbytes`, `Sload`, `Dload`, `Spkts`, `Dpkts`): log-scale histograms split by `Label`, plus boxplots and violins on a symlog axis. Histograms use the full 2.54M rows (binning is N-independent); box/violin use a 200,000-row random sample, which would otherwise draw one marker per outlier and be unreadable.

**Result.** `sbytes`, `dbytes`, and `Sload` show visibly different distributions between normal and attack traffic; several other features overlap heavily across both classes.

**Understand.** The separating features carry real class signal and are kept. The overlapping ones add little and join the drop-candidate list from section C.

---

## F. Correlation heatmap

**Method.** Full 40×40 numeric correlation matrix rendered as a heatmap (`center=0`), a visual view of the redundancy quantified in section C.

**Result.** Clear correlated blocks — the connection-tracking `ct_*` counters, and the byte/packet/loss group (`sbytes`/`sloss`; `dbytes`/`dloss`/`Dpkts`).

**Understand.** Confirms the numeric feature space is partly redundant; dropping one column per correlated pair cuts dimensionality with little information loss.

---

## G. Dimensionality reduction (PCA / t-SNE / UMAP)

**Method.** Stratified sample of ≤3,000 rows per class (24,691 total, so rare classes stay visible), `fillna(0)`, `StandardScaler`, then 2-D projection by PCA (linear), t-SNE and UMAP (non-linear).

**Result.** Normal and Generic separate cleanly; Exploits / Fuzzers / DoS / Reconnaissance overlap heavily across all three projections.

**Understand.** The cleanly separated classes will be easy to classify. The overlapping ones are where confusion concentrates — row-wise features alone can't split them, which is the argument for a relational GNN using host/service structure in Task 2.

---

## H. Interactive plots (Plotly)

**Method.** Hoverable versions of the class-distribution bar chart, the PCA scatter, and a `sbytes`-by-category boxplot (log y).

**Result.** Same findings as the static plots, explorable interactively (per-bar counts, per-point category, per-class spread).

**Understand.** Presentation aid — no new result; makes it easy to see which classes pile onto Normal and to inspect specific outliers.

---

## Host count — graph-construction feasibility

**Method.** Count unique source IPs, destination IPs, and their union; compute average flows per host.

**Result.** 43 unique `srcip`, 47 unique `dstip`, **49 unique hosts** in the union, averaging **~51,838 flows per host**.

**Understand.** Only 49 hosts carry 2.54M flows, so a host-level graph is tiny and hub-heavy — a few very high-degree nodes. For Task 2 this argues for finer-grained `IP:port` nodes, or an explicit justification of the hub structure. It is also why the raw files (which keep `srcip`/`dstip`) were needed over the pre-split CSVs.

---

## Carries into Task 2

| Finding (this notebook) | Task 2 action |
|---|---|
| 18.92% duplicate rows | Deduplicate |
| `is_ftp_login`, `ct_flw_http_mthd` ~53–56% missing | Indicator / zero imputation (missing = protocol absent) |
| `is_sm_ips_ports` near-constant (99.83%) | Drop |
| ~11 correlated pairs (\|r\| > 0.9) | Drop one per pair, keep the higher-`Label`-correlated one |
| Heavy right-skew on byte/rate/duration columns | Scale / log-transform |
| 12,752 : 1 class imbalance | Class weights / resample train split only; evaluate with macro-F1 + per-class recall |
| `Stime`/`Ltime`, `sttl`–`Label` = 0.904 leakage smell | Exclude timestamps from features; split by host/time; report the leakage risk |
| Class overlap in 2-D (Exploits/Fuzzers/DoS/Recon) | Relational GNN — the research gap |
| 49 hosts, ~51.8k flows each | Choose node granularity; handle hub-heavy graph |
