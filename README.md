# Individual-neuron spiking classification: APP/PS1 vs WT (Liam)

Can the spike timing of a **single neuron** tell us which genotype (APP/PS1 vs WT) the mouse it was
recorded from has? Neurons come from chronic multi-region recordings (ACad, CA1, RSPv, V1; RSU and FS
cells), with each neuron's interspike intervals (ISIs) split by Wake / NREM / REM. Each question is asked
separately in **young** (≤ 200 d) and **old** (> 200 d) animals.

Two classifiers are compared over the same grid of tests (sleep state × brain region × cell type, within
all / old / young animals):
1. **XGBoost on hand-crafted spike metrics.** 10 ISI statistics per state (mean, median, max, SD, CV,
   CV2, LV, LvR, gamma α/β).
2. **MLP on the ISI distribution itself.** 100 log-spaced histogram bins (1 ms–60 s) per state.

A side project fits REM functional connectivity (point-process GLM) and relates it to genotype.

These are organized, commented copies of the notebooks in `Liam_work/tests/` on the lab server
(outputs stripped, headers and section notes added, output paths pointed into this repo; analysis logic
unchanged). Every notebook opens with a header giving its original file, inputs, outputs and runtime.

## Layout

```
non_behavioral_spike_analysis/   (orig. tests/non_behavioral_spike_analysis)
  01_compute_clean_spike_metrics.ipynb        STEP 1: raw ISIs -> clean per-state metric tables
  02_feature_distributions_by_animal.ipynb    QC: every metric, per animal
  03_xgboost_spike_metric_classification.ipynb STEP 2: XGBoost, leave-two-animals-out, full grid
behavior_isi_analysis/           (orig. tests/behavior_isi_analysis)
  01_raw_isi_mlp_classification.ipynb         STEP 3: MLP on ISI histograms, same grid + extras
  02_isi_and_behavior_feature_mlp_early.ipynb earlier version (Mar 2026), superseded by 01
  03_p_ad_vs_connectivity.ipynb               per-neuron P(AD) from the MLP vs GLM connectivity
graphical_analysis/              (orig. tests/graphical_analysis)
  01_point_process_glm_connectivity.ipynb     REM GLM coupling matrices (writes data/glm_results/)
  02_gnn_classification.ipynb                 graph neural network on the connectivity graphs
  03_degeneracy_analysis.ipynb                Marder-style degeneracy / cross-region coordination
  04_degeneracy_coregulation.ipynb            co-regulation of coupling filters
data/spike_metrics/              step-1 output used by the XGBoost notebook (~0.7 MB each)
```

## Data

**Raw input:** `/media/HlabShare/Ben_work/isi/all_animal_isi_df.pkl` (13 GB).
- 5,501 neuron rows from 20 animals.
- Columns: `animal, epoch, age, sex, geno ('wt' | 'app_ps1'), brain_region, cell_type (RSU | FS),
  peak_latency, cell_no`, plus `wake_isi`, `nrem_isi`, `rem_isi`. Each of the last three is a list with
  one ISI array (s) per epoch in that state.
- About 1,500 of the ~1.6 billion ISIs are invalid (≤ 0 or non-finite). Everything downstream keeps
  only ISIs > 0.

**Cohorts after exclusions** (from the latest run; neuron counts are rows):

| group | animals | neurons |
|---|---|---|
| young WT (≤ 200 d) | bth90, bth95, bth100, zbr106 | 1,041 |
| young AD | zbr113, zbr114, zbr117, zbr118, zbr119 | 1,237 |
| old WT (> 200 d) | caf69, caf77, caf81, caf82, caf88 (+ caf42, see below) | 1,513 (+222) |
| old AD | caf71, caf73, caf74, caf75 | 1,238 |

**Exclusions**
- `caf104` (old WT) is dropped everywhere. 81 % of its neurons have within-epoch ISIs > 100 s; the
  next-worst animal has 9.7 %. This points to spike-sorting instability.
- `caf42` (WT, 207 d, right at the age cut) is dropped from hyperparameter tuning and from every
  classification run. The notebooks give the reason as "at user request; biological rationale pending."
- Features `min_isis` and `mean_ifr` are dropped.
  - `min_isis` mostly reflects the rig's sampling floor.
  - `mean_ifr` is 1 / `mean_isis`.

## Classification target

The label in **both** main notebooks is **genotype**: APP/PS1 = "AD" = 1, WT = 0. Age is **not** part
of the label. Age only splits the data into the ALL / OLD / YOUNG cohorts, and each cohort is
classified AD vs WT separately. So:
- **OLD** answers "old APP/PS1 (amyloidosis) vs old WT."
- **YOUNG** answers "can we detect the genotype before pathology?"

**Interpretation.** In young mice, classification is only slightly above chance, which may come from
APP/PS1 transgene overexpression. In old mice it is clearly better. We read that improvement as amyloid
pathology progressing and changing the spike timing of individual neurons in a way the classifiers pick
up. The main comparison is therefore OLD vs YOUNG within each config, not the ALL cohort.

## Validation and metrics (XGBoost and MLP notebooks)

- **Leave-two-animals-out (L2O).** Each fold holds out one WT and one AD animal, and every (WT, AD)
  pair is a fold: 9 × 9 = 81 folds for ALL and 4 × 5 = 20 for OLD or YOUNG. A neuron's animal is never
  in training.
- **Per-animal score.** The mean P(AD) over the animal's neurons, then averaged over every fold the
  animal was held out in. The animal is called AD when the score is ≥ 0.5.
- **Metrics.**
  - `N_Acc` / `N_BAcc`: neuron-level (balanced) accuracy pooled over folds.
  - `A_Acc`: unique animals called correctly, e.g. 16/18.
- **Hyperparameters.** One hyperopt TPE search per model (3-fold StratifiedGroupKFold by animal) on all
  analysed animals. The same params are then used for every config. ⚠ The search includes the L2O test
  animals, so report the results as "hyperparameters selected using all animals."
- **Importance.**
  - XGBoost: gain importance averaged over the fold models.
  - MLP: held-out permutation importance per histogram bin.

## Latest results (Liam's run, 2026-09-25)

Animal accuracy `A_Acc`. Neuron balanced accuracy is in parentheses.

| config | XGBoost ALL | XGBoost OLD | XGBoost YOUNG | MLP ALL | MLP OLD | MLP YOUNG |
|---|---|---|---|---|---|---|
| All states | 16/18 (60.0%) | 8/9 (65.5%) | 7/9 (57.6%) | 16/18 (60.2%) | 8/9 (69.3%) | 7/9 (60.0%) |
| Wake | 8/18 (51.1%) | 6/9 (54.8%) | 8/9 (56.2%) | 12/18 (50.6%) | 8/9 (54.6%) | 8/9 (57.5%) |
| NREM | 9/18 (50.5%) | 6/9 (56.1%) | 6/9 (55.5%) | 10/18 (50.5%) | 8/9 (61.8%) | 7/9 (58.1%) |
| REM | 16/18 (60.5%) | 8/9 (69.8%) | 7/9 (55.5%) | 12/18 (58.6%) | 8/9 (71.0%) | 7/9 (53.9%) |
| ACad | 12/18 (52.1%) | 6/9 (49.9%) | 5/9 (57.0%) | 11/18 (50.3%) | 6/9 (54.5%) | 6/9 (58.1%) |
| CA1 | 14/17 (59.9%) | 7/8 (65.4%) | 8/9 (57.1%) | 12/17 (59.4%) | 7/8 (67.7%) | 7/9 (58.3%) |
| RSPv | 11/18 (53.3%) | 5/9 (52.4%) | 6/9 (44.1%) | 12/18 (54.9%) | 7/9 (57.1%) | 5/9 (48.0%) |
| V1 | 10/18 (48.6%) | 7/9 (57.4%) | 4/9 (39.8%) | 15/18 (56.5%) | 7/9 (55.1%) | 7/9 (56.1%) |
| FS | 12/18 (54.4%) | 8/9 (56.9%) | 7/9 (55.7%) | 13/18 (57.1%) | 8/9 (60.2%) | 6/9 (51.8%) |
| RSU | 13/18 (58.8%) | 8/9 (64.9%) | 5/9 (54.3%) | 13/18 (57.9%) | 8/9 (69.1%) | 8/9 (59.7%) |

Takeaways:
- The neuron-level signal is weak, about 50–70 % balanced accuracy.
- Averaging over an animal's hundreds of neurons makes the animal call much better.
- REM-only ISIs carry most of the signal, in both models and especially in old animals.
- Wake and NREM alone are near chance in the ALL cohort.

In the MLP, the most important ISI windows are roughly 50–110 ms in Wake, 90–220 ms in NREM and
100–300 ms in REM. A model trained on only those bins does about as well as equal-sized random-bin
controls (`behavior_isi_analysis/01_raw_isi_mlp_classification`, section 10).

## Caveats worth keeping in mind
- **Sample size.** There are 18 animals: 9 in OLD and 9 in YOUNG. One animal is worth 11 percentage
  points of `A_Acc` in an age split.
- **Tuning leakage.** Hyperparameters are chosen on the same animals that are later held out (see
  above).
- **Different ISI ranges.** The MLP keeps ISIs in 1 ms–60 s; the metric tables use every positive ISI.
  So XGBoost vs MLP is not a model-only comparison.
- **Possible REM-amount cue.** In the MLP, a neuron with < 10 ISIs in a state gets an all-zero
  histogram for that state. Old AD mice have much less REM, so REM all-zero rows may
  partly encode "little REM" rather than ISI shape. This is worth checking before interpreting the REM
  advantage.
- **Age cutoff changed.** It is 200 d in the current notebooks and 212 d in the older ones. `caf42`
  (207 d) flips between "old" and "young" depending on which cutoff is used.
- **Repeated neurons.** Neuron rows can repeat across recording epochs and ages. Splits are always by
  animal, so this does not leak between train and test.
- **Provenance of the clean metric tables.** Liam's saved outputs in the original `compute_clean_spikemetrics` print
  `*_v2.pkl` file names. The tracked pickles also carry `attrs` keys (`isi_support`, `raw_size`,
  `raw_mtime_ns`) that the current code doesn't set. So they came from a slightly earlier revision of
  that notebook. Their `metric_schema` / `lvr_R_ms` tags match.
- **GLM path mismatch in the originals.** The original GLM notebook saved to `tests/glm_results/`, but
  its consumers read from `tests/graphical_analysis/glm_results/`. The copies all use
  `data/glm_results/`, which is not in git (two ~45 MB pickles); regenerate it with
  `graphical_analysis/01_point_process_glm_connectivity`.

## Environment

Run on the shared server **marvin**. Keep the preamble at the top of every notebook: thread limits of
4 per process, `pdf.fonttype = 42`, DejaVu Sans. The main notebooks use `N_JOBS = 20`.

Liam ran these notebooks in the conda envs `james_test` / `james_test_copy` (Python 3.8). Both envs
have everything needed: numpy 1.23, pandas 1.4, scikit-learn 1.1, xgboost 1.5, hyperopt 0.2.7,
imbalanced-learn, elephant, joblib, networkx, statsmodels, torch and torch_geometric.

**Run order:**
1. `non_behavioral_spike_analysis/01_…` (only needed to rebuild `data/spike_metrics/`).
2. `non_behavioral_spike_analysis/03_…` and `behavior_isi_analysis/01_…`, in either order. The MLP
   notebook reads the raw 13 GB table directly.
3. `graphical_analysis/01_…` before the other graphical notebooks and `behavior_isi_analysis/03_…`.
