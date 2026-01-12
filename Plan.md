# Project Proposal: Chem2Gen-Bench v2.1

**Title**: *Bridging the Gap: Benchmarking the Transferability Between Chemical and Genetic Perturbations in Transcriptomic Space & Stress-Testing AI Fidelity*

---

# Part 1. Motivation and Scientific Premise (Revised v3.1)

### 1.1 The Industrial Dogma and the "Translation Gap"

Modern drug discovery is built on a fundamental, yet often unverified, axiom:

> **Chemical perturbation should phenocopy genetic perturbation.**

* **The Pipeline Logic:** A target is genetically validated (e.g., CRISPR knockout of Gene X yields a desired phenotype), followed by chemical screening for a small molecule that inhibits Protein X. The implicit assumption is that the drug will reproduce the downstream phenotype of the genetic perturbation.
* **The Reality:** This translation from Genetics (DNA/RNA-level perturbation) to Chemistry (protein-level modulation) incurs **Translation Loss**: a drug is not a “genetic scissors,” but an exogenous chemical entity that triggers distinct kinetic, structural, and systemic responses.
* **The Problem:** While we have binding affinity, occupancy, and potency metrics (e.g., IC50/EC50), we lack a systematic **Fidelity Metric** to quantify how faithfully a drug’s transcriptomic impact mimics its intended genetic target.

### 1.2 Deconstructing the Biological Mismatch

We posit that discordance between chemical and genetic signatures is not random noise, but reflects fundamental biological distinctions:

1. **Structural vs. Catalytic (the “Scaffold” Hypothesis):** Small molecules often inhibit catalytic activity, whereas CRISPR/RNAi can deplete the entire protein. If a protein plays scaffold/structural roles, catalytic inhibition may not reproduce the loss-of-function transcriptome.
2. **Acute Shock vs. Genetic Compensation:** Drugs induce acute, synchronized stress responses (minutes–hours). Genetic perturbations are often chronic, allowing rewiring and **genetic compensation**.
3. **The “Dirty” Reality:** Drugs are promiscuous. **Polypharmacology** and **xenobiotic stress/toxicity** confound mechanistic interpretation. Distinguishing “on-target mechanism” from “systemic stress” is central for pharmacological transcriptomes.

### 1.3 The AI Imperative: A “Turing Test” for Virtual Cells

With the rise of **Virtual Cell** modeling and **Foundation Models (FMs)** for in silico screening:

* **The Risk:** Models trained on mixed perturbation data can “hallucinate” alignment, implicitly enforcing `Drug(Target X) ≈ Gene(Target X)` during representation learning.
* **The Consequence:** If a model cannot distinguish “faithful” vs. “divergent” chemical perturbations for the same nominal target, it will generate false positives in virtual screens and produce biologically invalid druggability assumptions.
* **Our Solution:** **Chem2Gen-Bench** provides a **ground-truth evaluation oracle**: it does not merely measure similarity; it supports (i) a **Scoreboard of Chemical→Genetic transferability**, and (ii) a **stress test** for AI systems to ensure they respect biological boundaries between chemical and genetic interventions.

---

# Part 2. Data Assets & Representation

### 2.1 Harmonized Feature Space

All analyses operate in a curated gene universe (~2.5k genes), including L1000 landmarks and shared target genes, to reduce modality shift and stabilize cross-source comparisons.

### 2.2 Stratified Subsets (Task 1 & 2 Benchmark)

Defined in `1_DataSelection.py` using multi-source benchmark data (LINCS + scPerturb):

| Level | Definition | Role |
| --- | --- | --- |
| **L1** | **Same Cell + Same Target** | **Gold Pairs**: primary gap measurement and retrieval ground truth. |
| **L2** | Different Cell + Same Target | **Pointer Pools**: used for target-centric generalization tests. |
| **L3** | Same Cell + Different Target | **Background Pool**: used for estimating cell-specific background signatures. |

> Implementation note: the benchmark is saved as a split bundle (parquet + split tensors + manifest) and loaded via `functions.py` (`load_chem2gen_bundle_from_split`) to enable scalable I/O and lazy tensor loading.

### 2.3 Two Views of the Signature (Bias-Aware Evaluation)

To test whether systematic bias inflates apparent alignment, we operationalize two views:

1. **Standard View:** the raw differential signature.
2. **Systema View:** a background-subtracted signature intended to remove shared stress/systematic components.

For Task 1, Systema background is estimated using the **L3 background pool** (cell-centric reference) as implemented in `3_Task1_Pairwise_Test.py`.

### 2.4 The “Gold Standard” Matched Evaluation Set (Task 3 Specific)

A separate, strictly matched dataset constructed from **Replogle 2022 (GWPS)** and **Srivatsan 2020 (Sci-Plex)** in **K562** cells.

* **Structure:** matched **(Chemical vs. Genetic)** raw counts + full controls per modality.
* **Canonical Target Label (`target_std`):** manually curated to ensure consistent target naming and correct handling of compound annotations. This field is treated as the **ground-truth target label** in Task 3 evaluation.
* **Specificity Tier (`tier`):** stores the stratification of compound specificity (used for subgroup evaluation):
  * **Cleanest:** single-target drugs
  * **Family:** pan-family inhibitors
  * **Promiscuous:** multi-target / polypharmacology-heavy compounds

---

# Part 3. Tasks and Analysis Logic

## Task 0: Data Curation

**Script:** `1_DataSelection.py`  
**Output:** split benchmark bundle (parquet + split tensors + manifest)

* **Action:** merges LINCS (bulk) and scPerturb (single-cell) metadata; filters for valid modalities and constructs L1/L2/L3 subsets.

---

## Task 1: Unsupervised Gap Quantification (Measurement)

**Scripts:** `3_Task1_Pairwise_Test.py` & `4_Task1_Retrieval_Test.py`

* **1A — Metric-Based Alignment:** quantifies geometric (`cosine`, `cosine_z`) and distributional (`edist`) alignment between chemical and genetic vectors across **Standard** and **Systema** views.
* **1B — Multi-Scenario Retrieval:** diagnoses retrieval failures across:
  * **Scenario A (Target Retrieval):** does a chemical query retrieve the correct genetic target (and vice versa)?
  * **Scenario B (Cell Dependency Retrieval):** does a target’s alignment generalize across cells?
  * **Scenario C (Global Specificity):** can true target signal be recovered against a broad background?

---

## Task 2: Attribution and Mechanism (Explaining the Gap)

**Script:** `5_Task2_Analysis_Attribution.py`

* **Step 1 — Scoreboard:** classifies contexts into **Consistently_Good**, **Consistently_Bad**, and **Conditionally_Good**.
* **Step 2 — Biological Attribution:** identifies drivers associated with high/low transferability (e.g., gene- or pathway-level signatures linked to robust alignment).
* **Step 3 — Protocol Sensitivity:** evaluates dose/time dependence and context sensitivity for conditional targets.

---

## Task 3: Foundation Model Stress Test (AI Evaluation)

**Dataset:** `Evaluation_Set_K562` (matched Gold Standard)  
**Model scripts:** `8_Task3_scGPT.py`, `8_Task3_geneformer.py`, `8_Task3_scBERT.py`, `8_Task3_UCE.py`, `8_Task3_STATE.py`, `8_Task3_Tahoe-x1.py`, `8_Task3_scFoundation.py`  
**Delta preparation:** `9_Task3_PrepareAnalysis.py`  
**Evaluation:** `10_Task3_AnalyzeOutputs.py`  
**Collection:** `11_Task3_OutputCollect.py`  
**Visualization:** `12_Visualization_Task3.r`

### Objective

Evaluate whether SOTA foundation models capture the **biological truth** of chemical-genetic transferability *without fine-tuning*.

### Methodology: Latent-Space (and Gene-Space) Delta Arithmetic

We evaluate models using **cell-level deltas** computed against matched controls:

For each modality and each cell-level perturbation instance *i*,
* **Latent delta (FM track):**
  * `Δ_i = z_i - mean(z_controls)` where `z` is the FM embedding.
* **Gene-space delta (reference track):**
  * `Δ_i = x_i - mean(x_controls)` where `x` is normalized expression / counts-derived representation.
* **Systema view (Task 3):**
  * subtract the shared mean background within each modality/track to reduce global stress components:
  * `Δ_i^sys = Δ_i - mean(Δ_all)`.

This yields comparable `Δ` representations across:
* **Tracks:** Gene, Pathway, and each FM (scGPT/Geneformer/…)
* **Views:** Standard vs Systema
* **Modalities:** CRISPR vs Drug

### Evaluation Logic

1. **Retrieval (Primary):**  
   *Drug→CRISPR* and *CRISPR→Drug* retrieval using similarity in delta space.  
   Goal: does a query retrieve the correct `target_std`?

2. **Target-Level Alignment (Complementary):**  
   For each target, compute centroid deltas per modality and evaluate:
   * centroid cosine alignment
   * distributional alignment (e.g., Energy Distance)

3. **The “Turing Test” Correlation:**  
   Does model-implied alignment (Task 3 Systema view) correlate with benchmark-implied alignment (Task 1 Systema scoreboard) for targets where Task 1 provides a stable ground truth?

4. **Specificity Tier Check (Key Subgroup Analysis):**  
   Evaluate metrics stratified by `tier`:
   * Cleanest vs Family vs Promiscuous  
   Hypothesis: promiscuous compounds should show systematically weaker and/or noisier alignment to single-gene perturbations.

---

# Part 4. Visualization Plan (Panelized Figures)

## Figure 1 — The Landscape of Chemical–Genetic Alignment (Task 1 & 2)

* **1A:** Workflow & data hierarchy (L1/L2/L3 and Standard/Systema views).
* **1B:** Data composition (cell line, perturbation type, source, hierarchy).
* **1C:** Global comparison heatmaps: `Standard` vs `Systema` and `Gene` vs `Pathway` alignment (debiasing effect).
* **1D:** Retrieval success across scenarios; definition of Consistently_Good / Bad / Conditional classes.
* **1E (Optional):** Embedding map (PCA/UMAP) showing chemical vs genetic delta distributions and separation under Standard vs Systema.

## Figure 2 — Attribution: Why and When It Works (Task 2)

* **2A:** Paired statistical evidence comparing `Standard` vs `Systema` / `Gene` vs `Pathway`.
* **2B:** Tracer plots: scenario-wise retrieval profiles for Consistently_Good vs Consistently_Bad exemplars.
* **2C:** Enrichment / attribution plots for drivers of success vs failure.
* **2D:** Dose/time driver analysis for conditional targets.
* **2E:** Correlation-of-correlations (Task 1 ground truth vs FM latent alignment).
* **2F:** Tier boxplots: model-predicted similarity stratified by `tier`.

## Figure 3 — Foundation Model Scoreboard (Task 3)

* **3A:** Scoreboard heatmap: models × metrics (retrieval + pairwise) in Standard/Systema.
* **3B:** Δ(Systema−Standard) improvement lollipops per model/track.
* **3C:** Target-level improvement heatmap stratified by tier (Cleanest/Family/Promiscuous).

---

# Benchmark Comparison: Where Chem2Gen-Bench Fits

| Project / paper | Primary question | Modalities | Main evaluation style | Systematic-variation awareness | How Chem2Gen-Bench differs |
| --- | --- | --- | --- | --- | --- |
| Wei et al., 2025 | Prediction accuracy | scRNA-seq (Gen+Chem) | Population metrics | Low/Medium | Focuses on **cross-modality transfer** (Chem→Gen), not only prediction. |
| Systema (2025) | Metric inflation | scRNA-seq (Genetic) | Bias quantification | High | Applies Systema logic to the **Chem–Gen translation gap**. |
| scDrugMap (2025) | Drug response prediction | scRNA-seq (Chemical) | Response classification | Low/Medium | Assesses **mechanism fidelity**, not just response/toxicity. |
| **Chem2Gen-Bench** | **Are drugs proxies for genes?** | **Bulk + single-cell** | **Retrieval + gap quantification + AI stress test** | High | 1) Task-defined gap measurement<br>2) Systema-aware debiasing<br>3) Tiered specificity analysis<br>4) FM zero-shot stress test |
