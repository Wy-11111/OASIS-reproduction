# OASIS: source code and results

Supplementary code of the manuscript *"OASIS: An Efficient and Training-Free Search-Plan Selector for Spatial
Queries on Resource-Constrained Devices"*.

The package contains (1) the code of the seven search plans, of OASIS, and of all compared selectors, (2) the scripts
that run every experiment of Section 5, and (3) the measured results behind every figure and table of the paper, so
that the figures can be redrawn without rerunning the experiments.

## 1. Contents

```
README.md
requirements.txt
run_all.sh            runs the whole pipeline (Section 3 below)
raw/                  put the downloaded datasets here (Section 2 below)
results_paper/        measured results used in the paper, one folder per dataset
src/
  prep.py             raw files -> data/<dataset>.bin (insertion order, scaled into the unit box)
  gen.py              samples the points and generates all query streams
  bench.cpp           the seven search plans; measures the search time of every plan on every query
  export_est.py       expected number of answers of every query (from cell counts and prefix sums)
  replay_core.py      shared code: loading, features, the compared selectors
  replay.py           replays the compared selectors on the measured times
  choose.py           picks the parameters of the earlier OASIS variant that replay.py still replays (see step 4)
  selbench.cpp        measures the selection time of the compared selectors
  oasis2.py           OASIS: two-level feature extraction and time-limited trials
  replay2.py          replays OASIS; ablation, parameter study, scalability
  choose2.py          picks the parameters of OASIS on the validation streams
  selbench2.cpp       measures the selection time of OASIS
  merge2.py           merges the results of OASIS and of the compared selectors
  plot_paper.py, plot_paper2.py, plot_paper3.py   figures of Section 5 (plot_paper3.py is the entry point)
  plot_intro.py       Figure 1
  why.py              mean search time of each plan per query type and size (the numbers of Figure 1a)
```

## 2. Requirements and data

* Linux, GCC with C++17 (the C++ files include `<bits/stdc++.h>`), Python 3 with the packages in
  `requirements.txt`. `replay2.py` shares data with its worker processes by `fork`, so it does not run on Windows.
* The four datasets are public and are not redistributed here. Download them and place them in `raw/` under the
  file names that `prep.py` expects:

| Dataset | Source | Files in `raw/` |
|---|---|---|
| Uber | https://github.com/fivethirtyeight/uber-tlc-foil-response | `uber-apr.csv`, `uber-may.csv`, `uber-jun.csv`, `uber-jul.csv`, `uber-aug.csv`, `uber-sep.csv` (the six 2014 files, renamed) |
| GeoNames | https://www.geonames.org | `US.zip` (contains `US.txt`) |
| Porto | https://www.kaggle.com/c/pkdd-15-predict-taxi-service-trajectory-i | `porto.zip` (contains `train.csv` or `train.csv.zip`) |
| Dragon | https://graphics.stanford.edu/data/3Dscanrep/ | `dragon.ply.gz` (XYZ RGB Dragon, binary PLY) |

## 3. Reproducing the experiments

`bash run_all.sh` runs all steps in order. The steps are:

1. **Build.** `bench.cpp`, `selbench.cpp`, and `selbench2.cpp` are compiled with `g++ -O3 -march=native -std=c++17`.
2. **Data.** `prep.py` converts the raw files; `gen.py` samples 1M points per dataset (and 0.25M, 0.5M, and 2M points
   for the scalability study) and writes the streams: `mixed`, the five distribution streams, `insert`, `drift`, a
   validation stream, and the training streams of the offline selectors.
3. **Search times.** `bench7` (built from `bench.cpp`) runs every query with all seven plans, records the minimum of
   three runs per plan, and checks that all plans return the same answer.
4. **Compared selectors.** `replay.py` trains the offline selectors, replays every selector on the measured search
   times, and measures their selection times with `selbench`. It also replays an earlier variant of OASIS, whose
   parameters `choose.py` picks; step 6 replaces these rows by the results of `oasis2.py`, which is the OASIS of
   the paper.
5. **OASIS.** `replay2.py --tune` and `choose2.py` choose the parameters on the validation streams;
   `replay2.py --cost` measures the selection time with `selbench2`; `replay2.py` replays OASIS on all streams,
   including the ablation and the parameter study; `replay2.py --scal` runs the scalability study.
6. **Merge and figures.** `merge2.py` writes `results_merged/`; `plot_paper3.py` and `plot_intro.py` draw the figures.

Step 3 is by far the longest: it runs every query of every stream 21 times. We ran it as one batch job per dataset.
All measurements in the paper were taken on one thread of a server with two Intel Xeon Gold 6438Y+ CPUs; absolute times
on other machines differ.

## 4. Redrawing the figures from the shipped results

No dataset or compiler is needed for this step (Python with matplotlib only):

```
cd src
FIGMODE=mdpi python plot_paper3.py ../results_paper ../figures
python plot_intro.py ../results_paper ../figures
```

| Paper | Result file in `results_paper/<dataset>/` | Figure file |
|---|---|---|
| Figure 1 | `GeoNames/stream.csv` (panel b); panel a from `why.py` | `intro_problem.pdf` |
| Query time on the mixed stream | `total_time.csv` | `exp_total.pdf` |
| Selector models | `selectors.csv` | `exp_selectors.pdf` |
| Query distributions | `distributions.csv` | `exp_distributions.pdf` |
| Insertions and query drift | `stream.csv` | `exp_stream.pdf` |
| Breakdown of the query time | `overhead.csv` | `exp_overhead.pdf` |
| Labeling time, training time, and memory (table) | `memory.csv` | - |
| Scalability | `scalability.csv` | `exp_scalability.pdf` |
| Ablation | `ablation2.csv` | `exp_ablation.pdf` |
| Parameter study | `params2.csv` | `exp_params.pdf` |

`summary.csv` holds the mean query time and the prediction accuracy of every method on every stream (the numbers
quoted in the text), `tune2.csv` the validation results of every parameter setting of OASIS, `cost2.json` and
`cost_us.json` the measured selection times, and `info2.csv` the numbers of classes and trials of OASIS.
The files also contain rows of `UnIS-R`, a variant of UnIS that relabels and retrains periodically; it is not
reported in the paper, and `plot_paper3.py` leaves it out.

## 5. Names in the paper and in the code

| Paper | Code |
|---|---|
| Plans KD-DFS (A), KD-BF (B), KD-BFS (C), Grid (D) | `KD-DFS-MBR`, `KD-BF`, `KD-BFS-MBR`, `Grid` |
| Plans KD-DFS-Ball, KD-BFS-Ball, R-BF | `KD-DFS-Ball`, `KD-BFS-Ball`, `R-BF` |
| Class threshold tau = 1 | `tau = 1.0` |
| Candidates per class h = 2 | `r = 2` |
| Initial temperature theta_0 = 0.1 | `T0 = 0.1` |
| Time-limit factor b = 3 | `b = 3.0` |
| Warming factor beta | `gamma = 1.0` |
| Weight N_0 of the class experience | `N0` |
| Without two-level features / without time-limited trials / without both | `w/o Features` / `w/o Trials` / `Base` |

The parameter values used in the paper are in `results_paper/oasis2_config.json`; `choose2.py` selected them among
the settings listed in `tune2.csv`. Values that are not in this file keep the defaults in `oasis2.py`.

Three settings of the implementation are fixed in the code and are not varied in the paper:

* `N0 = 1e9` in the selected configuration: the combined mean of a plan then equals its class mean, and the cell
  experience enters only through the number of observations that sets the temperature. The validation results of
  the finite values `N0 = 8` and `N0 = 32` are in `tune2.csv`.
* `cf = 4`: the cell experience is kept on a grid that is four times coarser per dimension than the grid of the
  cell counts (16 x 16 cells in 2D, 6 x 6 x 6 in 3D).
* `K = 32`: at most 32 classes are kept per query type.

## 6. Differences from the code as it was run

The measurement and replay code is unchanged. For this package, the cluster job scripts were replaced by
`run_all.sh` (same commands, no site-specific paths), two label strings in `plot_paper2.py` were set to the wording
of the paper ("Without time-limited trials", theta_0), and a few outdated comments were corrected (e.g., "six
strategies" to seven).
