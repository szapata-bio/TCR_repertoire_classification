# Methods — α/β chain analysis (Rosati & Russell)

Extending the repertoire-level validation to a new label — **chain (alpha vs beta)** — across two
independent bulk cohorts. Same machinery as the CD4/CD8 work (preprocess → embed → mean+cov descriptor
→ `ref_size_sweep_auroc`), with the label swapped.

## Objective

Validate the repertoire pipeline on a different axis and in different data. The **chain** label is a
**positive control (sanity check)**: alpha and beta use distinct loci (TRAV vs TRBV), so they must
separate near-perfectly if the pipeline works. Passing this in two cohorts of different format and
processing confirms the method is robust. Two further labels — **donor** and **disease group** — probe
whether the descriptor captures finer, dispersed signal (it does not, as expected for bulk; see
Highlights below).

Bulk paired data — not single-cell. There is no molecular α–β pairing; the donor test asks whether the
two chains of a donor share a *repertoire-level* signature, not a molecular one.

## Datasets

### Rosati (PRJEB50045, IBD)
- **Source:** MiXCR (`readCount` + `uniqueMoleculeCount`) → abundance available.
- **Subset:** Study 1, bulk PBMC blood, AlphaBeta scheme, v2 version.
- **Donors:** 209 with both chains (CD 88, Healthy 85, UC 36). Groups: yes (CD/UC/Healthy).
- **Two UMI versions** (see "The UMI decision" below):

| | UMI≥1 | UMI≥2 |
|---|---|---|
| Complete pairs | 208 | 207 |
| Clouds | 417 | 416 |
| Median clonotypes/cloud | ~26,200 | ~2,510 |
| % singletons (median) | 90–92% | — |
| Clonotypes α (total) | 5.76M | 712k |
| Clonotypes β (total) | 7.44M | 817k |
| V-resolved | 99.5–99.7% | 99.7–99.8% |
| Depth ratio α/β | 0.88 | ~1.0 |

### Russell (Nicaragua, HLA)
- **Source:** TCRdist, pre-filtered to **UMI≥5**, count column dropped → **no abundance, uniform weights**.
- **Donors:** 150 with both chains. Groups: none (homogeneous cohort).
- **Single version:**

| | UMI≥5 (uniform) |
|---|---|
| Complete pairs | 150 |
| Clouds | 300 |
| Clonotypes/cloud range | ~4,300 – 26,300 |
| Clonotypes α (clean, total) | 1.37M |
| Clonotypes β (clean, total) | 1.94M |
| V-resolved | 99.5–99.9% |
| Depth ratio α/β | ~1.0 (balanced) |
| Cleaning example (1 donor) | α 8,751→7,574 · β 9,808→9,373 |

## The UMI decision (why two versions in Rosati, uniform weights in Russell)

**Rosati** keeps molecular counts (UMI), so abundance is real. But 90–92% of clonotypes are singletons
(UMI=1). Filtering at **UMI≥2** yields cleaner but ~10× smaller clouds (~2,510 vs ~26,200 clonotypes);
UMI≥1 keeps everything but is dominated by singletons. Rather than pick blindly, both versions were run
and compared — the conclusions are identical across thresholds (chain = 1.0 in both), which is itself
evidence of robustness.

**Russell** was delivered already filtered to UMI≥5, with the count column dropped. No abundance survives,
so weights are **uniform** (`w = 1/N`). This does not affect the *model* (it embeds each sequence
regardless of weight); it only affects the *descriptor* step (the weighted average summarizing the cloud).
For global differences (chain, donor) it changes no conclusions.

## Chain label as a sanity check

The chain sweep is the pipeline's **positive control**. It confirms that the full chain
(preprocessing → embedding → descriptor → metric) works on each dataset: if chain did **not** give ~1.0,
something would be broken. It validates that the descriptor captures signal, that cosine distances are
reliable, and that `ref_size_sweep_auroc` runs correctly. Passing in **both** cohorts confirms the
pipeline is robust to format (MiXCR vs TCRdist) and to the absence of abundance (uniform weights in Russell).

**Rosati ↔ Russell — similarities:**
- Chain AUROC = 1.000 in both, across all reference-set sizes.
- Depth-only baseline ≈ chance (Rosati 0.50–0.54, Russell 0.57) → separation is **biological, not depth**.
- Rarefaction changes nothing (1.0 down to very low K: 78 in Rosati, 409 in Russell).
- V-gene baseline = 1.0 (trivial: TRAV vs TRBV are distinct genes).
- Balanced α/β depth (~1.0) → no cross-chain depth confound.

**Key difference — occupancy.** In both datasets occupancy fell below mean+cov for chain (Russell
~0.92–0.95; Rosati UMI≥1 ~0.975, UMI≥2 ~0.79), while mean+cov was always 1.0. The occupancy landmarks
were fit on **beta** repertoires, so the histogram is biased toward β and separates chains slightly
worse; the effect worsens with small clouds (Rosati UMI≥2 → 0.79) because a 256-landmark histogram gets
noisy with few clonotypes. **Takeaway:** mean+cov is more robust than occupancy, especially at low depth.

## Highlights — donor and group (expected negatives)

These were run but show no notable signal, as expected for bulk. Reported here for completeness; the
chain sweep is the notebook of record for the α/β analysis.

- **Donor signature (cross-chain retrieval).** Can a donor's alpha cloud retrieve its own beta cloud
  among all betas? **No** — AUROC ≈ 0.52 in both cohorts (Rosati α→β 0.517 / β→α 0.526; Russell 0.509 /
  0.542), median rank ≈ half, top-1 ≈ chance. The rank histograms are flat. **Expected:** bulk has no
  molecular pairing; the two chains share only the donor's immune history, which is too subtle for the
  global descriptor. The negative **replicates across two independent cohorts** (different populations,
  formats, processing) → robust.

- **Disease group (Rosati only).** Do repertoires cluster by CD/UC/Healthy? **No** — AUROC 0.51–0.53
  (multiclass and pairwise, alpha and beta). **Expected & consistent** with the original Rosati finding:
  the Crohn signal lives in *specific shared clonotypes*, not in the *global* repertoire composition that
  mean+cov summarizes. A whole-cloud descriptor dilutes a needle-in-a-haystack clonotype signal.

**Overall message.** The repertoire descriptor captures **global** differences robustly (lineage in
CD4/CD8, chain identity here) but not **fine, dispersed** signal (donor identity in bulk, disease-specific
clonotypes). The chain sanity check is what lets us read the donor/group ~0.5 as a genuine absence of
signal rather than a pipeline failure.
