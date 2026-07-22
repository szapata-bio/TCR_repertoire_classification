# Methods

Repertoire-level validation of a frozen TCR foundation model, across three analyses:
CD4/CD8 classification, CD4/CD8 deconvolution, and α/β chain analysis in two cohorts.

Shared pipeline for every analysis: **preprocess → embed (frozen model) → mean+cov descriptor → 
`ref_size_sweep_auroc`**. No model is trained.

---

## Shared method

- **Descriptor.** Each repertoire (cloud of clonotypes) → one fixed vector: weighted **mean +
  covariance** of its L2-normalized clonotype embeddings. A second descriptor, **occupancy** (soft
  histogram over 256 landmarks), is used for comparison and deconvolution.
- **Metric.** `ref_size_sweep_auroc`: pick `r` reference repertoires of one class, score every other
  by max cosine to that set, take the AUROC, average over draws (`n_draws=100`) and classes. Cosine on
  L2-normalized descriptors. Deterministic (`seed=0`).
- **Controls.** Depth (rarefaction), depth-only baseline, second descriptor (occupancy), V-gene-usage
  baseline, per-cohort split.

---

## Part 1–2 · CD4/CD8 (classification + deconvolution)

**Data.** TCR-β repertoires, three cohorts (HD/MS/T1D). CD4/CD8 separated by **FACS** → labels are
cytometry ground truth. **Bulk with UMI** (MiXCR `generic-amplicon-with-umi`, RNA); UMI gives abundance,
but no single-cell resolution and no α–β pairing. 68 repertoires, 37 patients.

**Preprocessing (from raw MiXCR).** Per repertoire: PCR-error correction (V-gene + CDR3 Hamming≤1 merge);
drop non-productive; robustness filter (reads≥2 & UMI≥2); CDR3 starts with C and ≥8 aa; dedup to
(CDR3, V) summing counts; weights `w_log = log(1+count)` normalized. CD4/CD8 overlap is **not** removed.

**Results.** Repertoires separate CD4/CD8 at AUROC 0.93 (1 reference) → 0.99 (25). Signal survives
rarefaction (not depth), tracks across descriptors (occupancy ≈ mean+cov), beats the V-gene baseline,
and holds within each cohort.

**Deconvolution (bonus).** The descriptor is **compositional**: CD4/CD8 fractions of a mixture are
recoverable cross-patient (leave-one-patient-out, MAE ≈ 0.147; occupancy slightly better at 0.132, from
its additivity), with flat error (~0.13) in the physiological range (CD8 ≈ 0.25–0.40).
**Scope:** the "bulk" is a *synthetic in-silico mix of FACS-sorted CD4/CD8* — labels are ground truth,
but it is not real unsorted bulk. Proof of concept; next step is real bulk with cytometry-measured
fractions.

**Variance (descriptive).** Within-group dispersion: CD8 > CD4 (more heterogeneous between patients).
PCA: PC1 (~44% of variance) is the CD4/CD8 axis; disease cohorts do not form separate clusters.

---

## Part 3 · α/β chain analysis (Rosati & Russell)

Same pipeline, label = **chain (alpha vs beta)**, across two independent bulk cohorts. See
[`methods_alphabeta.md`](methods_alphabeta.md) for full dataset tables and the UMI decision.

**Chain = sanity check (positive control).** Alpha/beta use distinct loci → must separate near-perfectly
if the pipeline works. AUROC = **1.000** in both cohorts, robust to depth (rarefaction), format (MiXCR
vs TCRdist), and absence of abundance (uniform weights in Russell). This is what makes the downstream
~0.5 results interpretable as genuine absences of signal, not failures.

- **Rosati** (PRJEB50045, IBD): MiXCR with UMI, 209 donors, groups CD/UC/Healthy. Two UMI thresholds
  (≥1 and ≥2) run and compared — identical conclusions.
- **Russell** (Nicaragua): TCRdist, pre-filtered UMI≥5, no abundance → uniform weights, 150 donors,
  homogeneous cohort (no groups).

**Expected negatives (reported, not highlighted).**
- *Donor signature* (cross-chain retrieval): AUROC ≈ 0.52 in both cohorts → no shared donor signature
  across chains. Expected for bulk (no molecular pairing); replicates across two cohorts.
- *Disease group* (Rosati): AUROC 0.51–0.53 → no global disease signal. Consistent with the Crohn signal
  being clonotype-specific, not global — a whole-cloud descriptor dilutes it.

**Message.** The descriptor captures **global** repertoire differences (lineage, chain) robustly, but not
**fine, dispersed** signal (donor in bulk, disease clonotypes).

---

## Design notes

- `n_draws=100` reduces AUROC sampling variance; `seed=0` → fully reproducible.
- mean+cov is more robust at low depth; occupancy is better for deconvolution (additivity). Descriptor
  choice is task-dependent.
- Weights affect only the descriptor step, never the model (which embeds each sequence independently of
  abundance).
