# TCR_repertoire_classification

[![Built with Claude](https://img.shields.io/badge/Built_with-Claude-D97757)](https://claude.ai)
![Python](https://img.shields.io/badge/Python-3.10-blue)
![License](https://img.shields.io/badge/License-MIT-green)

Extending clonotype-level TCR classification to whole repertoires, using a frozen foundation model ([CD4_CD8_classification](https://github.com/ilyada/CD4_CD8_classification)) with no retraining. Each repertoire — a donor's cloud of thousands of clonotypes — is summarized into a single fixed descriptor (mean + covariance of its L2-normalized clonotype embeddings, weighted by clonal abundance), then analyzed with a reference-set-size retrieval metric (`ref_size_sweep_auroc`).

The contribution here is the repertoire-level formulation, the full preprocessing pipelines from raw data, the evaluation against standard confounds, a proof-of-concept CD4/CD8 deconvolution, and a two-cohort α/β chain validation. Developed with assistance from Claude (Anthropic).

## Three parts

**1 · CD4/CD8 classification.** Can a whole repertoire be classified CD4 vs CD8? Yes — AUROC 0.93 (1 reference) → 0.99 (25), robust to depth, descriptor choice, a V-gene baseline, and cohort.

**2 · CD4/CD8 deconvolution.** Is the descriptor compositional? Starting from FACS-sorted CD4/CD8 populations mixed in silico at known fractions, the CD4/CD8 fraction is recoverable cross-patient (~0.13 MAE) — a proof of concept toward reading immune composition from bulk, with real unsorted validation as the next step.

**3 · α/β chain validation (Rosati & Russell).** The same pipeline on a new label — chain (alpha vs beta) — across two independent bulk cohorts. Chain separates perfectly (AUROC 1.000) as a **sanity check** confirming the pipeline is robust to data format (MiXCR vs TCRdist) and to the absence of abundance. Donor and disease-group labels give no signal, as expected for bulk (reported for completeness).

## Key results

| Question | Result |
|---|---|
| Do repertoires separate CD4/CD8? | **Yes** — AUROC 0.93 (1 reference) → 0.99 (25 references) |
| Depth artifact? | **No** — rarefied ≈ full depth |
| Descriptor-specific? | **No** — occupancy tracks mean+cov |
| Beyond V-gene usage? | **Yes** — foundation > V-gene baseline (esp. at low reference counts) |
| Cohort-driven? | **No** — holds within HD, MS, T1D separately |
| Compositional (deconvolution)? | **Yes** — CD8 fraction recoverable cross-patient, ~0.13 MAE |
| Pipeline robust across cohorts/formats? | **Yes** — chain AUROC 1.000 in both Rosati & Russell |

**Takeaway.** The frozen clonotype model carries a signal that **aggregates coherently to the repertoire level**: it captures *global* differences (CD4/CD8 lineage, chain identity) robustly and is compositional enough to read immune fractions from mixtures. It does not capture *fine, dispersed* signal (donor identity in bulk, disease-specific clonotypes) — an expected and informative boundary.

## Repo structure
```

├── notebooks/
│   ├── cd4cd8/
│   │   ├── 01_preprocess.ipynb                   # raw MiXCR -> clean clouds; depth confound
│   │   ├── 02_cd4cd8_repertoire_evaluation.ipynb # main sweep + controls
│   │   ├── 03_variance_analysis.ipynb            # within-group dispersion + PCA
│   │   └── 04_deconvolution.ipynb                # CD4/CD8 fraction recovery
│   ├── rosati/
│   │   ├── rosati_01_preprocess.ipynb            # MiXCR, two UMI thresholds
│   │   ├── rosati_umi_histograms_EN.ipynb        # UMI-threshold diagnostic
│   │   └── rosati_02_chain_sweep.ipynb           # chain sanity check
│   └── russell/
│       ├── russell_01_preprocess.ipynb           # TCRdist, uniform weights
│       └── russell_02_chain_sweep.ipynb          # chain sanity check
├── docs/
│   ├── methods.md                                # full pipeline, all three parts
│   └── methods_alphabeta.md                      # α/β datasets, UMI decision, details
├── figures/                                      # exported plots (safe to share)
├── requirements.txt
└── README.md

```

## Pipeline (high level)

1. **Preprocess** raw clonotype files → clean per-(donor, chain) clouds (productive filter, locus filter, dedup, abundance weights). See [`docs/methods.md`](docs/methods.md).
2. **Embed** each clonotype with the frozen foundation model → 128-d vectors per clonotype.
3. **Describe** each repertoire → one mean+cov descriptor per cloud.
4. **Evaluate** with `ref_size_sweep_auroc` + controls (rarefaction, V-gene baseline, per-cohort).
5. **Deconvolve** / **validate** — CD4/CD8 fractions from mixtures; chain/donor/group across cohorts.

## Reproducibility

The notebooks load pre-embedded clouds from disk and are deterministic (`seed=0`, `n_draws=100`). Re-running gives identical numbers. The embedding step (run once) freezes the data; the analysis is computed on that frozen result.

## Data

Patient repertoire data is **not included** (human clinical data). The notebooks show the full method and the resulting figures; the raw/embedded clouds stay local. Paths in the notebooks are generic placeholders — edit them to your local checkout.

## Requirements

​```
numpy
pandas
matplotlib
scipy
scikit-learn
pyarrow
​```

Plus the `rep_*` library (from the referenced repository) on the path.

## Acknowledgements

Built on the foundation model and `rep_*` library from [CD4_CD8_classification](https://github.com/ilyada/CD4_CD8_classification). Developed with assistance from Claude (Anthropic), used as a support tool for experimental design, code review, and analysis.
