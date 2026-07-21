# TCR_repertoire_classification
Repertoire-level CD4/CD8 classification using a frozen TCR foundation model

Extending clonotype-level TCR classification to whole repertoires, using a frozen foundation model (CD4_CD8_classification) with no retraining. This repository reformulates the CD4/CD8 problem at the repertoire level: each repertoire — a donor's cloud of thousands of clonotypes — is summarized into a single fixed descriptor (mean + covariance of its L2-normalized clonotype embeddings, weighted by clonal abundance), then classified with a reference-set-size retrieval metric (ref_size_sweep_auroc). 

The contribution here is the repertoire-level formulation, the full preprocessing pipeline from raw data, the evaluation against standard confounds, and a proof-of-concept CD4/CD8 deconvolution from synthetic bulk mixtures. Developed with assistance from Claude (Anthropic).

Key results
Question	Result
Do repertoires separate CD4/CD8?	Yes — AUROC 0.93 (1 reference) → 0.99 (25 references)
Depth artifact?	No — rarefied ≈ full depth
Descriptor-specific?	No — occupancy tracks mean+cov
Beyond V-gene usage?	Yes — foundation > V-gene baseline (esp. at low reference counts)
Cohort-driven?	No — holds within HD, MS, T1D separately
Compositional (deconvolution)?	Yes — CD8 fraction recoverable cross-patient, ~0.13 MAE

Takeaway. The frozen clonotype model carries a CD4/CD8 signal that aggregates coherently to the repertoire level and is compositional enough to read immune fractions from mixtures.

Repo structure
.
├── notebooks/
│   ├── 01_cd4cd8_repertoire_evaluation.ipynb   # main: sweep + controls + deconvolution
│   └── 02_deconvolution.ipynb                  # full CD4/CD8 deconvolution experiment
├── docs/
│   └── methods.md                              # datasets, preprocessing, design decisions
├── figures/                                    # exported plots (safe to share)
├── requirements.txt
└── README.md
Pipeline (high level)
Preprocess raw clonotype files → clean per-(donor, chain) clouds (productive filter, locus filter, dedup, abundance weights). See docs/methods.md.
Embed each clonotype with the frozen foundation model → 128-d vectors per clonotype.
Describe each repertoire → one mean+cov descriptor per cloud.
Evaluate with ref_size_sweep_auroc + controls (rarefaction, V-gene baseline, per-cohort).
Deconvolve CD4/CD8 fractions from synthetic bulk mixtures (leave-one-patient-out).
Reproducibility

The notebooks load pre-embedded clouds from disk and are deterministic (seed=0, n_draws=100). Re-running gives identical numbers. The embedding step (run once) freezes the data; the analysis is computed on that frozen result.

Data

Patient repertoire data is NOT included (human clinical data). The notebooks show the full method and the resulting figures; the raw/embedded clouds stay local. Paths in the notebooks are generic placeholders — edit them to your local checkout.

Requirements
numpy
pandas
matplotlib
scipy
scikit-learn
pyarrow

Plus Ilya's rep_* library (from his repo) on the path.

Developed with assistance from Claude (Anthropic). Foundation model and rep_* library by Ilya (CD4_CD8_classification).
