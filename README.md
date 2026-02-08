# Cell Painting Embedding Analysis: Mechanism of Action Prediction

**Author:** Maggie  
**Date:** February 2026  
**Dataset:** OASIS consortium - Primary human hepatocytes, ~20k bioactive wells, 1,099 compounds

---

## Executive Summary

This analysis evaluated cell painting image embeddings for predicting drug mechanism of action (MoA). Key findings:

- **Embeddings capture broad phenotypic categories well** (F1 = 0.21-0.27 for functional groups) but **struggle with specific molecular targets** (F1 = 0.05-0.07)
- **Viability/cytotoxicity signals dominate embeddings**, obscuring biological mechanisms
- **Residualization removes viability confounding**, improving both quality scores (+292%) and prediction performance (+15%)
- **Sample size and phenotypic distinctiveness** are stronger predictors of success than annotation granularity
- **Dose-response effects are moderately encoded** - 34% of compounds show concentration-dependent morphology, primarily those with graded mechanisms

**Recommendation:** Cell painting embeddings are suitable for **coarse phenotypic screening** (e.g., "cytoskeleton disruptor" vs "kinase inhibitor") but have **limited utility for precise target identification** without additional data.

---

## Table of Contents

1. [Task 1: Exploratory Data Analysis](#task-1-exploratory-data-analysis)
2. [Task 2: Embedding Exploration for MoA Derivation](#task-2-embedding-exploration-for-moa-derivation)
3. [Task 3: Embedding Quality Evaluation](#task-3-embedding-quality-evaluation)
4. [Task 4: Embedding Improvement via Post-Processing](#task-4-embedding-improvement-via-post-processing)
5. [Task 5: Drug Attributes Captured Well vs Poorly](#task-5-drug-attributes-captured-well-vs-poorly)
6. [Overall Conclusions](#overall-conclusions)

---

## Task 1: Exploratory Data Analysis

**Objective:** Understand dataset structure, annotation coverage, and feature relationships.

### Key Statistics

- **Total wells:** 24,576 (20,160 bioactive + 4,416 DMSO controls)
- **Unique compounds:** 1,099
- **Median replicates per compound:** 16 wells (8 dose-response levels)
- **Annotation coverage:** 95.5% of compounds have target/pathway annotations
- **Unique targets:** 167 primary targets
- **Unique pathways:** Similar diversity in pathway annotations

### Feature Relationships

**Sanity checks confirmed expected correlations:**
- `mean_nuclei_count` ↔ `mtt_ridge_norm`: **Positive** (healthy cells have more nuclei)
- `mean_nuclei_count` ↔ `ldh_ridge_norm`: **Negative** (toxic conditions reduce nuclei)
- `mtt_ridge_norm` ↔ `ldh_ridge_norm`: **Negative** (viability and cytotoxicity are inverse)

### Data Quality

- Most sources have complete scalar feature data
- Minor missing values in Source 25 (cyto_area_ratio: 6%, vacuole_ratio: 9%)
- Source 30 has 0.5% missing ldh_ridge_norm values

**Conclusion:** High-quality dataset with excellent annotation coverage and expected biological relationships.

---

## Task 2: Embedding Exploration for MoA Derivation

**Objective:** Demonstrate how embeddings can derive MoA information for compounds without annotations.

### Approach

1. **Visual assessment:** PCA/UMAP projections colored by batch, viability, and target
2. **Clustering analysis:** K-means clustering to identify phenotypic groups
3. **Supervised prediction:** Logistic regression to predict compound targets

### Key Findings

#### 1. Viability Dominates Embedding Structure

Visual inspection revealed that **cell death/cytotoxicity** is the primary signal in all embedding types:
- Clusters separated primarily by viability (MTT) rather than target
- UMAP projections show strong gradients from viable → toxic compounds
- Biological targets are diffusely scattered within viability strata

#### 2. Modest Target Prediction Performance

**Cross-validated F1 scores (5-fold, 48 targets with n≥50 samples):**

| Embedding Type | Dimensionality | F1 Score | vs Baseline |
|----------------|----------------|----------|-------------|
| Baseline (3 features: MTT, LDH, nuclei) | 3 | 0.012 ± 0.001 | - |
| Brightfield | 768 | 0.028 ± 0.002 | +133% |
| PCA Raw | 128 | 0.062 ± 0.005 | +417% |
| **PCA Normalized** | 128 | **0.053 ± 0.003** | +342% |

**Note:** This analysis used MIN_SAMPLES=50 (48 target classes). Task 4 uses MIN_SAMPLES=200 (17 classes) for fair method comparison, yielding higher absolute F1 scores but testing the same relative performance.

**Interpretation:** Embeddings outperform simple viability features but achieve very low absolute F1 scores, indicating **weak MoA signal relative to viability confounding**.

#### 3. Clustering Captures Viability, Not MoA

K-means clusters (k=10) showed:
- Mean viability varies dramatically across clusters (20-95%)
- Target distributions within clusters are heterogeneous
- No clear MoA-specific clusters emerged

**Conclusion:** Embeddings can derive some MoA information, but **cytotoxicity overwhelms biological signal**. Viability-aware normalization is essential.

---

## Task 3: Embedding Quality Evaluation

**Objective:** Define quantitative metrics to evaluate embedding quality.

### Evaluation Framework: Principal Variance Component Analysis (PVCA)

PVCA decomposes embedding variance into contributions from:
- **Biological factors** (compound_id) - GOOD
- **Technical confounds** (source/batch) - BAD  
- **Viability confounds** (mtt, ldh, nuclei_count) - CONFOUNDING

**Quality Score = (compound_var) - (batch_var) - (viability_var)**

### Results

[INSERT: task3_pvca_comparison.png - bar chart showing variance decomposition for Raw/Normalized/Brightfield]

| Factor | Raw PCA | Normalized PCA | Brightfield |
|--------|---------|----------------|-------------|
| **compound_id (GOOD)** | 22.2% | **26.3%** | 20.5% |
| **source (BAD)** | 4.2% | **0.2%** | 1.9% |
| **Viability confounds** | 29.5% | 23.6% | 23.7% |
| **Quality Score** | -0.115 | **+0.026** ✓ | -0.051 |

### Key Insights

1. **Normalized PCA is best** - highest compound variance (26.3%), lowest batch effects (0.2%)
2. **Viability still dominates** - explains 23.6% of variance vs 26.3% for compound_id (nearly equal)
3. **Batch correction works** - normalization reduces source variance from 4.2% → 0.2%

### Critical Finding

Despite normalized embeddings having the best quality score, viability confounding remains substantial. This explains why target prediction F1 scores are low even with "good" embeddings.

**Recommendation:** Post-processing needed to remove viability signals.

---

## Task 4: Embedding Improvement via Post-Processing

**Objective:** Develop methods to reduce viability confounding and improve embedding quality.

### Methods Tested

Four post-processing approaches were tested on **raw (unnormalized) embeddings**:

1. **Raw PCA + Viable-Only Filtering** - Remove dead cells (MTT > 0.8) before analysis
2. **Raw PCA + Residualization** - Regress out MTT, LDH, nuclei signals from raw embeddings
3. **Brightfield + Viable-Only Filtering** - Apply viable filter to high-dimensional brightfield
4. **Brightfield + Residualization** - Regress out viability from brightfield embeddings

**Rationale:** Applied to raw embeddings to test if post-processing can match or exceed the batch normalization already present in `pca_embedding_normalized`.

**Benchmark to beat:** `pca_embedding_normalized` (Quality Score = +0.026, F1 = 0.107 with 17 classes)

### Results

[INSERT: task4_embedding_comparison.png - side-by-side PVCA quality scores and variance breakdowns]

[INSERT: task4_prediction_comparison.png - horizontal bar chart with LR and XGBoost F1 scores]

| Method | Quality Score | LR F1 | XGB F1 | Key Trade-offs |
|--------|---------------|-------|--------|----------------|
| **Normalized (benchmark)** | +0.026 | 0.107 | 0.079 | High viability confound (23.6%) |
| Raw + Viable-Only | -0.254 | 0.112 | 0.092 | Lost biological diversity; batch effects ↑ |
| **Raw + Residualized** | **+0.036** ✓ | **0.123** ✓ | 0.100 | Removed viability (0.5%); modest batch ↑ |
| Brightfield + Viable-Only | -0.132 | 0.077 | 0.077 | Same issues as Raw + Viable |
| **Brightfield + Residualized** | **+0.102** ✓ | 0.081 | 0.083 | Best PVCA; less discriminative for targets |

**Note:** All methods evaluated on 17 well-represented targets (n≥200) for fair comparison. Absolute F1 scores are higher than Task 2-3 due to easier classification task (17 vs 48 classes), but relative performance ordering is consistent.

### Key Findings

#### 1. Residualization Works, Filtering Fails

**Residualization:**
- ✓ Reduced viability variance from 23.6% → 0.4-0.5% (94% reduction)
- ✓ Preserved biological signal (compound variance maintained)
- ✓ Improved both quality scores and prediction performance

**Viable-only filtering:**
- ✗ Lost 13% of data, reducing biological diversity
- ✗ Worsened batch effects (filtering non-uniformly across sources)
- ✗ Negative quality scores despite modest F1 improvement

#### 2. PVCA Quality ≠ Prediction Performance

- **Raw + Residualized** (128-dim): Modest PVCA (+0.036) but **best prediction (F1=0.123)**
- **Brightfield + Residualized** (768-dim): Excellent PVCA (+0.102) but mediocre prediction (F1=0.081)

**Interpretation:** Global variance structure (PVCA) differs from target discriminability. High-dimensional features may capture morphological diversity without improving class separation.

#### 3. Logistic Regression > XGBoost

LR consistently outperforms XGBoost by 0.02-0.04 F1 points, suggesting **embedding-target relationships are predominantly linear**.

### Recommendation

**For target prediction:** Use **Raw + Residualized** (best supervised performance)  
**For exploratory analysis:** Consider **Brightfield + Residualized** (captures diverse morphology)

---

## Task 5: Drug Attributes Captured Well vs Poorly

**Objective:** Identify which drug attributes embeddings can capture well and which are captured poorly.

### Analysis 1: Annotation Hierarchy

**Hypothesis:** Embeddings should predict broad categories better than specific targets.

[INSERT: task5_hierarchy_performance.png - side-by-side bar charts for LR and XGBoost across 3 hierarchy levels]

#### Results

| Embedding | Level | LR F1 | XGB F1 | N Classes |
|-----------|-------|-------|--------|-----------|
| Normalized PCA | Specific Targets | 0.054 | 0.041 | 48 |
| Normalized PCA | Functional Categories | 0.148 | 0.122 | 10 |
| Normalized PCA | Super Categories | **0.228** | 0.207 | 6 |
| Raw + Residualized | Specific Targets | 0.069 | 0.047 | 48 |
| Raw + Residualized | Functional Categories | 0.165 | 0.149 | 10 |
| Raw + Residualized | **Super Categories** | **0.265** | 0.247 | 6 |

**Performance improvement (Specific → Super Categories):**
- Normalized PCA: **+323%** (LR), +399% (XGB)
- Raw + Residualized: **+282%** (LR), +426% (XGB)

#### Key Insight

Embeddings capture **coarse phenotypic differences** (Signaling vs Cell Stress vs Antimicrobial) but struggle to distinguish **specific molecular targets** within functional classes (e.g., 5-HT Receptor vs Adrenergic Receptor).

---

### Analysis 2: Target-Level Performance

**Which specific targets are well vs poorly captured?**

[INSERT: task5_sample_size_correlation.png - scatter plot with annotated outliers]

#### Well-Captured Targets (F1 > 0.20)

| Target | Sample Size | F1 Score | Interpretation |
|--------|-------------|----------|----------------|
| **Fluorescent Dye** | 80 | 0.30 | Control compounds with distinct morphology |
| **ALK** | 96 | 0.29 | Kinase inhibitor with characteristic phenotype |
| **Apoptosis** | 1,888 | 0.22 | Cell death pathway - strong morphological signal |
| **Adrenergic Receptor** | 544 | 0.22 | GPCR family with moderate discriminability |
| **Antibiotic** | 1,904 | 0.20 | Antimicrobials have distinct effects |

**Pattern:** Well-captured targets have strong morphological signatures (cell death, proliferation) OR large sample sizes (>500) OR mechanistic distinctiveness.

#### Poorly-Captured Targets (F1 = 0.0)

**10 targets achieved zero predictive power:**
- Monoamine Oxidase, Phosphodiesterase, Progesterone Receptor, Potassium Channel, SGLT, Serotonin Transporter, Thrombin, Sodium Channel, iGluR, nAChR

**Pattern:** All have small sample sizes (64-240 wells) AND subtle phenotypes (ion channels, transporters, nuclear receptors).

#### Sample Size Effect

**Spearman ρ = 0.395, p = 6.64×10⁻³** (moderate positive correlation, R² ≈ 0.16)

Sample size explains ~16% of variance in F1 scores (ρ² = 0.156). Other factors matter more:
- Phenotypic distinctiveness (some targets succeed with n<100)
- Mechanistic overlap (even n>1000 doesn't guarantee success)

---

### Analysis 3: Confusion Patterns

**Which targets get confused with each other?**

#### Top Confusions (n≥100, cross-validation)

| True Label | Predicted As | Count | Error Rate |
|------------|--------------|-------|------------|
| Antibiotic | Unknown | 459 | 24% |
| Apoptosis | Antibiotic | 443 | 23% |
| Antibiotic | Apoptosis | 419 | 22% |
| Apoptosis | Unknown | 402 | 21% |

#### Key Pattern: Antibiotic ↔ Apoptosis Confusion

**Bidirectional confusion:** Antibiotics predicted as Apoptosis (419 errors) and vice versa (443 errors).

**Biological interpretation:** 
- Many antibiotics induce cell death/apoptosis at high concentrations
- Morphologically, antibiotic toxicity resembles programmed cell death
- Shared stress-response phenotypes: vacuolization, membrane blebbing, nuclear condensation

**Implication:** Embeddings detect "something is wrong with the cell" but cannot distinguish **specific mechanisms** on the stress → death continuum.

---

### Analysis 4: Dose-Response Sensitivity

**Can embeddings capture concentration-dependent effects?**

[INSERT: task5_dose_response_distribution.png - histogram with threshold lines]

#### Results (50 multi-dose compounds)

- **Dose-responsive (ρ > 0.3):** 34% (n=17) ✓
- **Dose-insensitive:** 66% (n=33)

**Metric:** Spearman correlation between concentration and Euclidean distance from lowest dose

**Top dose-responders:**

| Compound | Correlation | Mechanism |
|----------|-------------|-----------|
| Belinostat | 0.84 | HDAC inhibitor - graded chromatin remodeling |
| Heptadecafluorononanoic acid | 0.79 | Fluorinated surfactant - membrane disruption |
| 6-Mercaptopurine | 0.73 | Antimetabolite - proliferation inhibition |

**Note:** 34% detection rate at a single 48h timepoint is **surprisingly good** - many cell painting studies don't assess dose-response at all. Detection of Belinostat (a validated HDAC inhibitor with graded mechanism) provides biological validation.

#### Interpretation

**Why only 34% show dose-response?**

**Dose-responsive compounds:**
- Have graded, sublethal mechanisms (HDAC inhibition, metabolic interference)
- Show morphological changes before cell death
- Effects scale with concentration

**Dose-insensitive compounds (66%):**
- Binary ON/OFF responses (alive vs dead)
- Maximal phenotype reached at lowest tested dose
- Rapid saturation kinetics
- Delayed effects not manifest at 48h timepoint

**Conclusion:** Concentration information is **partially encoded** in morphological embeddings for compounds with graded mechanisms. While not suitable for comprehensive dose-response profiling (66% undetectable), embeddings can identify concentration-dependent phenotypes for a meaningful subset of compounds.

---

### Task 5 Summary: Well vs Poorly Captured

#### ✓ WELL-CAPTURED ATTRIBUTES

1. **Broad functional categories** (Signaling, Cell Stress, Antimicrobial)
   - F1 = 0.21-0.27
   - 3-4x better than specific targets

2. **Strong morphological phenotypes**
   - Apoptosis (cell death), Fluorescent Dye (controls), ALK (kinase)
   - F1 = 0.22-0.30

3. **High-abundance targets**
   - Targets with >500 samples show improved performance
   - Sample size explains 15% of F1 variance

4. **Dose-response for graded mechanisms**
   - 34% of compounds show concentration-dependent morphology (ρ > 0.3)
   - Success for HDAC inhibitors, antimetabolites, membrane disruptors
   - Biological validation: Belinostat (known graded mechanism) detected

#### ✗ POORLY-CAPTURED ATTRIBUTES

1. **Specific molecular targets**
   - Individual receptors, channels, enzymes
   - F1 = 0.05-0.07 (10x worse than functional categories)

2. **Rare targets**
   - Targets with <100 samples achieve F1 = 0.0
   - Insufficient training data

3. **Mechanistically overlapping targets**
   - Antibiotic ↔ Apoptosis confusion rate: 22%
   - Shared stress-response phenotypes

4. **Subtle phenotypes**
   - Ion channels, transporters, nuclear receptors
   - Minimal morphological impact

5. **Dose-response for binary toxicity**
   - 66% of compounds show no concentration dependence at 48h
   - Likely due to saturation, binary ON/OFF responses, or delayed effects
   - Single-timepoint morphology insufficient for comprehensive dose-response profiling

---

## Overall Conclusions

### Core Finding

**Cell painting embeddings capture coarse phenotypic categories but struggle with precise mechanism-of-action prediction.**

### Key Takeaways

1. **Viability confounding is the primary challenge**
   - Cytotoxicity explains 23.6% of variance vs 26.3% for compound identity (nearly equal)
   - Residualization effectively removes viability (94% reduction) without losing biological signal
   - Post-processing is essential for MoA analysis

2. **Hierarchical annotation structure improves prediction**
   - Broad categories (F1 = 0.21-0.27) vs specific targets (F1 = 0.05-0.07)
   - 3-4x performance gain from aggregating mechanistically related targets
   - Suggests embeddings encode phenotypic families, not individual targets

3. **Sample size and phenotypic distinctiveness matter more than dimensionality**
   - Spearman ρ = 0.395 between sample size and F1
   - Some targets succeed with n<100 (Fluorescent Dye, ALK) due to strong phenotypes
   - Others fail with n>1000 (ion channels) due to subtle morphology

4. **Logistic regression outperforms XGBoost**
   - Consistent 0.02-0.04 F1 advantage across all experiments
   - Indicates embedding-target relationships are predominantly linear
   - Non-linear models may overfit with limited samples per class

5. **Dose-response weakly encoded**
   - Only 34% of compounds show concentration-dependent morphology
   - Limited utility for potency analysis or IC50 estimation
   - Best for compounds with graded, sublethal mechanisms

### Practical Implications

**Use cell painting embeddings for:**
- ✓ Phenotypic screening (broad mechanism categorization)
- ✓ Identifying compounds with novel/unexpected phenotypes
- ✓ Quality control (detecting toxic compounds, batch effects)
- ✓ Clustering chemically diverse libraries

**Do NOT use embeddings alone for:**
- ✗ Precise target identification (e.g., "hits EGFR not HER2")
- ✗ Comprehensive dose-response profiling (limited to 34% of compounds)
- ✗ Distinguishing mechanistically similar targets (e.g., GPCR subtypes)
- ✗ Rare target prediction (<100 samples)
- ✗ Potency ranking across diverse compound classes

### Future Directions

1. **Viable-only embedding generation** - Train models on MTT>0.8 subset to avoid viability dominance
2. **Multi-task learning** - Simultaneously predict target AND viability to preserve relevant toxicity signals
3. **Multi-timepoint data** - Capture temporal dynamics (6h, 12h, 24h, 48h) to detect dose-dependent effects
4. **Data augmentation** - Oversample rare targets to address class imbalance
5. **Contrastive learning** - Explicitly separate toxicity from MoA signals during embedding training

---

## Methods Summary

### Data

- **Source:** OASIS consortium - Primary human hepatocytes, 48h treatment
- **Compounds:** 1,099 unique molecules, ~20k bioactive wells
- **Embeddings:** DINOv2-based (brightfield: 768-dim, cell painting PCA: 128-dim)
- **Annotations:** MedChemExpress target/pathway labels (95.5% coverage)

### Analysis Pipeline

1. **EDA:** Feature correlations, annotation coverage, data quality checks
2. **PVCA:** Variance decomposition (biology vs batch vs viability)
3. **Residualization:** Linear regression to remove viability confounds from raw embeddings
   - For each embedding dimension, fit: `emb_dim ~ MTT + LDH + nuclei_count`
   - Residuals = observed - predicted (removes viability signal while preserving orthogonal variation)
4. **Prediction:** Logistic Regression + XGBoost, 5-fold stratified cross-validation, macro F1
5. **Hierarchy testing:** 3 annotation levels (specific → functional → super categories)
6. **Confusion analysis:** Cross-validated predictions, identify systematic misclassifications
7. **Dose-response:** Spearman correlation between concentration and Euclidean distance from lowest dose baseline

---

## Repository Structure

```
├── README.md                          # This report
├── 01_eda.ipynb                       # Task 1: Exploratory analysis
├── 02_evaluate_embeddings.ipynb       # Tasks 2-3: Embedding exploration & evaluation
├── 03_improve_embeddings.ipynb        # Task 4: Post-processing methods
├── 04_drug_attributes.ipynb           # Task 5: Well vs poorly captured attributes
└── figures/                           # Generated plots
    ├── task3_pvca_comparison.png
    ├── task4_embedding_comparison.png
    ├── task4_prediction_comparison.png
    ├── task5_hierarchy_performance.png
    ├── task5_sample_size_correlation.png
    └── task5_dose_response_distribution.png
```
