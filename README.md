# Neural spectral recovery of hidden molecular components

Code for *"Spectral extraction of hidden molecular components from composite
vibrational measurements"* (Jiaheng Cui, Yanjun Yang, Haoran Lu, Yingchuan Zhang, Yufang Liu, Xianyan Chen, Jackelyn Murray, Les Jones, Hemant Naikare, Ralph A. Tripp, Wenxuan Zhong, Ping Ma and Yiping Zhao).

The method recovers an unknown target spectrum from a mixture in which a known
interfering component dominates, using only that one known reference plus
variation in how much each component contributes across measurements. Here the
target is SARS-CoV-2 RNA, the known component is the immobilized DNA probe, and
the measurements are surface-enhanced Raman (SERS) spectra.

## Layout

Each folder is one source of mixing diversity, in the order the paper presents
them. Every notebook opens with a cell describing its inputs, its outputs and
which figure it feeds.

```
1_all_concentration/     recovery from a full concentration series (the benchmark)
2_sequential_dilution/   recovery from a single 10-fold dilution step
  01_build_subgroups.ipynb
  02_extract_combinations.ipynb
  03_merge_extracted.ipynb
3_spatial/               recovery from one sample, no dilution at all
4_time_dependent/        recovery along the hybridization time axis
5_downstream_cnn/        does recovery improve classification and regression?
  01_build_unknown_test_samples.ipynb
  02_recover_unknown_test_samples.ipynb
  03_train_eval_recovered.ipynb
  04_train_eval_hybridized.ipynb
```

Folders 1, 3 and 4 hold one self-contained notebook each. Folders 2 and 5 run in
the numbered order.

## Data

The spectra are deposited separately on figshare (DOI: *to be added*). All are
supplied preprocessed: interpolated to a 1 cm⁻¹ grid over 400–1800 cm⁻¹
(1,401 points), cosmic-ray corrected, baseline-subtracted by airPLS, min–max
scaled to [0,1], then mean-normalized.

Preprocessing itself was performed in **SpectraGuru**
(https://spectraguru.org/), a published open platform, and is therefore not
reproduced here.

## The ×400 scale factor

Spectra are multiplied by a constant factor of 400 before the network sees them,
and the factor is divided out of every saved spectrum. This is a
numerical-conditioning choice: mean-normalized SERS intensities are small enough
that training is poorly behaved without it. Because the measured spectrum, the
DNA reference and the ground-truth reference are all scaled by the same factor,
it is an exact rescaling of both sides of the mixture equation, so the recovered
mixing coefficients and all shape-based metrics (cosine similarity, Pearson
correlation, R²) are unchanged. The value 400 was chosen empirically.

**All spectra written by this code, and all spectra in the deposited dataset,
are on the mean-normalized scale.**

## Analyses not included as code

Some steps were done in general-purpose software rather than in scripts. Their
definitions are given here so the results remain reproducible.

**Curve fitting and mixture-model selection** were performed in OriginPro. The
Hill fits (Eq. 2), the saturation fits (Eq. 3), the local Gaussian band fitting
and the Gaussian-mixture fits all live there; fitted parameters and BIC values
are tabulated in Supplementary Tables S1, S2, S4, S5 and S7–S10.

**The 1/e criterion.** Recovery quality `Q` is judged against a threshold set at
`Q_min + (Q_ideal − Q_min)(1 − 1/e)`. The two cases use different endpoints:

| | `Q` | `Q_min` | `Q_ideal` | threshold |
|---|---|---|---|---|
| spectral shape | cosine similarity to the measured RNA reference | 0.923, the similarity between the measured RNA and DNA references | the all-concentration benchmark, 0.995 | **0.970** |
| composition | NNLS RNA fraction | 0.256 | 1 exactly | **0.726** |

A concentration must satisfy both to count as recoverable.

**Spike identification.** For the recovered coefficients at one concentration, a
robust standard deviation is taken as `1.4826 × median absolute deviation from
the median`, and a position or timepoint counts as a spike when it exceeds the
median by more than three of those units.

**Mixture-model selection.** Gaussian mixtures with one, two and three
components are fitted by maximum likelihood and the component count is chosen by
BIC. Components are labelled after selection from their fitted means: a mean
below 0.01 is *negligible*; of the rest, the largest-mean component is
*dominant* and the smaller nonzero one *minor*. A single selected component is
labelled dominant.

## Environments

The recovery network and the downstream CNN were run on different machines.

| | recovery network | multi-task CNN |
|---|---|---|
| framework | PyTorch 2.3.1 (CUDA 12.1) | TensorFlow 2.10.0 |
| Python | 3.11.5 | 3.8.12 |
| hardware | Intel Core i7-13700KF, 64 GB RAM, NVIDIA RTX 4080 | Georgia Advanced Computing Resource Center: AMD EPYC Milan (64 cores, 1 TB RAM), 4× NVIDIA A100 |

Supporting libraries: NumPy 1.26.4, SciPy 1.12.0, scikit-learn 1.4.1.

## Notes on reproducing

- Paths at the top of each notebook point at the machines the work was done on
  and must be repointed at your own data.
- Notebooks are stored without outputs. Run them top to bottom.
- Seeds are set where sampling decides the result: `RANDOM_SEED = 42` in
  `01_build_subgroups.ipynb` and `02_recover_unknown_test_samples.ipynb`, and
  `SEED = 42` for the train/validation/test split in both CNN notebooks. So
  subgroup membership and the data split reproduce exactly.
- The recovery notebooks do **not** seed network initialization. Weights are
  drawn by `xavier_uniform_` each run, so repeated recovery of the same input
  gives slightly different spectra. Metrics in the paper are averaged over many
  subgroups or windows, which is what makes them stable.
