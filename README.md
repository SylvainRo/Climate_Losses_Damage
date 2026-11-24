# Climate Losses Damage

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22830784.svg)](https://doi.org/10.5281/zenodo.22830784)

Climate Losses Damage is a research-code repository for studying how damage
functions recover climate-related losses in the upper tail. It combines a
synthetic hazard-loss simulation study based on bivariate extreme-value
copulas with a real-data case study on the HANZE European flood-event
database — both live in **one self-contained notebook**,
`notebooks/Bootstrap Consolidated.ipynb`.

The simulation study compares five estimators, the first three fitted to 10
equal-count bin means:

- a quadratic regression, kept only if its quadratic coefficient is
  significant at the 5% level, and truncated at zero;
- a sigmoid regression on min–max-scaled bin means, kept only if its
  steepness is significant at the 5% level, and therefore capped at the
  largest bin mean;
- monotone isotonic regression, trained on the binned data or on the raw
  subsample (a control arm that isolates the cost of binning);
- isotonic distributional regression (IDR) on the raw subsample.

The main quantities of interest are the pointwise conditional mean
`E[L | H = h]`, the exceedance-conditioned mean `E[L | H >= h]`, and loss-risk
summaries such as Value-at-Risk (VaR) and Expected Shortfall (ES).

Throughout the code, "bootstrap" (`run_bootstrap_study`, the "Bootstrap mean"
table columns, ...) means what the paper calls **Monte Carlo subsampling**:
`iterations` subsamples of size `n`, each drawn without replacement from a
size-`N` population. [`docs/paper_mapping.md`](docs/paper_mapping.md) maps
every table and figure of the paper to the run and output file that produce it.

## Repository Structure

```text
Climate_Losses_Damage/
  README.md                    Project entry point
  LICENSE                      MIT License
  CITATION.cff                 Software citation metadata
  requirements.txt             Python dependencies
  notebooks/
    Bootstrap Consolidated.ipynb   Everything: synthetic study + HANZE case study
  data/
    README.md                  Only tracked file here -- everything else is generated/downloaded
    HANZE_events.csv           Real flood-event table (download separately, see below; gitignored)
    HANZE_events_parsed.csv    Cached cause-text parse of it, written by Part 12b (gitignored)
    *.h5                        Generated population/subsample caches (gitignored)
  results/
    README.md                  Only tracked file here
    ...                        Generated CSV result tables (gitignored)
  plots/
    README.md                  Only tracked file here
    ...                        Generated figures, if SAVE_FIGURES is turned on (gitignored)
  docs/
    reproducibility.md         Setup, how to run, what each output is
    data.md                    Schemas: HDF5 cache, results CSVs, HANZE_events.csv
    paper_mapping.md           Which run and output file produce each table/figure of the paper
```

`data/`, `results/`, and `plots/` each ship with only their own `README.md`
tracked, so the folder itself exists on a fresh clone even though its
generated contents don't — see each folder's `README.md` for exactly
what populates it and where that's documented.

Every distribution, copula, estimator, evaluation routine, and plotting
function the notebook uses is defined inline, in the notebook itself,
drawing only on the standard scientific-Python stack. The only files that
need to exist on disk before running it are `data/HANZE_events.csv` (see
below) and whatever `data/`/`results/`/`plots/` outputs a previous run has
already generated (all optional, all regenerable).

## Installation

Python 3.13 is required — the official IDR backend's wheel is built
specifically for CPython 3.13 (`cp313-abi3`) and will not install under
3.12. Create a fresh environment rather than reusing any local environment
directories checked into this tree (they may contain stale paths from an
earlier repository layout).

```bash
python3.13 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### Official IDR Backend

The IDR estimator requires the external `isodistrreg` Python bindings.
They are not installed by `requirements.txt`, since they are not
published on PyPI.

**Exact version used for this repository's results.** The wheel reports
version `0.5.1`, but it is *not* the tagged `v0.5.1` release (`ab7d844`): it
is a development build produced by the upstream CI from `master` at commit
[`a95ba86b03bdac83e1dbd1a0de276d41b562b588`](https://github.com/AlexanderHenzi/isodistrreg/commit/a95ba86b03bdac83e1dbd1a0de276d41b562b588)
(2026-07-10), 18 commits after `v0.5.1` and before `v0.5.2`, at a point where
the version number had not yet been bumped. Wheel:
`isodistrreg-0.5.1-cp313-abi3-manylinux_2_34_x86_64.whl`, sha256
`d571e7c59f87fa76e691a3f90a7bdec79a51e0264907443ce1a727cf0af5a783`. The
commit was identified from the build timestamp in the wheel's embedded SBOM,
which falls within exactly one upstream CI run (on that commit). CI wheels
expire, so to reproduce this exact version, build it from source at that
commit (requires a Rust toolchain):

```bash
python -m pip install "git+https://github.com/AlexanderHenzi/isodistrreg@a95ba86b03bdac83e1dbd1a0de276d41b562b588#subdirectory=bindings/python"
```

Otherwise, the upstream project currently instructs users to
download a platform-specific wheel from a recent GitHub Actions run and
install it locally — note that the current wheel targets CPython 3.13
specifically (`cp313-abi3`), consistent with this project's Python 3.13
requirement above:

```bash
python -m pip install /path/to/isodistrreg_wheel.whl
```

See the [official isodistrreg Python installation
guide](https://github.com/AlexanderHenzi/isodistrreg/blob/master/bindings/python/README.md).
Confirm the backend before running an IDR workflow:

```bash
python -c "from isodistrreg import IDR; print(IDR)"
```

`ipywidgets` (needed for the interactive HANZE dashboard, Part 12 of the
notebook) is included in `requirements.txt`.

### The HANZE dataset

`data/HANZE_events.csv` is large and Zenodo-licensed, so it is downloaded
separately rather than committed — get it and place it at
`data/HANZE_events.csv` before running Part 12. See `docs/data.md` for its
source and schema. Every other Part of the notebook runs on the synthetic
data it generates itself.

## Start Here

```bash
cd notebooks
jupyter lab "Bootstrap Consolidated.ipynb"
```

Run the notebook top to bottom. Parts 1–9 define everything (distributions,
copula, estimators, the bootstrap evaluation engine, plotting); Part 10 runs
it:

| Section | What it does | Cost |
| --- | --- | ---: |
| 10a — Light-scale run | Weibull hazard, Dagum losses, `N=100,000`, 100 iterations, 50 bins; cached (HDF5 + CSV) like every other run below. A quick check, not one of the paper's runs | seconds once cached, longer on first run |
| 10b — Full-scale sweep | Every combination of 1 loss margin (Dagum) × 2 hazard margins × 2 copulas (4 scenarios), each at `N=500,000`, 100,000 iterations, 10 bins, Gumbel θ = `GUMBEL_THETA`. The paper's θ = 1.1, θ = 5 and lognormal runs are re-invocations of this cell (see `docs/reproducibility.md`) | very expensive; `RUN_FULL_SCALE = True` by default |
| 10c — Sensitivity sweep | Dagum tail-index (`N=500,000`, 20,000 iterations, Weibull and GPD hazards; SI Fig. S2) and body-shape sweeps, 10 equal-count bins as in 10b | ~1.5 h uncached (alpha-sweep), minutes once cached |
| 10e — Extended diagnostic figures | Upper-tail zoom, both hazard-loss risk diagrams, IDR risk diagram and IDR conditional law, for the Weibull and GPD hazards at Gumbel θ = 1.1, 2 and 5 (includes the paper's Figs. 1, 2 and S1) | seconds once cached; with `RUN_FULL_SCALE`, a missing full-scale Weibull scenario at one of those θ is computed (a full 10b run) |

Part 12, at the end, is the HANZE real-data case study — an interactive
`ipywidgets` dashboard, independent of everything in Parts 1–11 except the
estimator machinery it reuses from Part 7/8.

See `docs/reproducibility.md` for a fuller walkthrough, including what
`RUN_FULL_SCALE` and `SAVE_FIGURES` actually do and where each output lands.

## Interpreting the Evaluation

At each probability level `q`, a subsample supplies an empirical hazard
threshold `h_q = VaR_q(H_sub)`. The evaluation records either:

- `pointwise`: the fitted conditional mean at `h_q`;
- `tail`: the fitted mean over hazards satisfying `H >= h_q`.

The theoretical comparison values are evaluated at the theoretical hazard
VaR, not at the random subsample VaR — a reported error can combine
estimator error and threshold-estimation error.

Empirical VaR is the lower quantile `inf{x : F_n(x) >= q}`
(`np.quantile(..., method="inverted_cdf")`), which is the subsample maximum for
every `q > 1 - 1/n`. Empirical ES is the superquantile of that quantile
function, `(1-q)^-1 ∫_q^1 Q_n(u) du`, the same coherent definition as the
theoretical ES it is benchmarked against. For the mean curves, marginal VaR/ES
are those of the fitted values `d_hat(H_i)` at the subsample's hazards, and the
tail target averages the fitted values over the observations with `H_i >= h_q`
(`n(1-q)+1` of them when `nq` is an integer).

For IDR, marginal VaR and ES are computed from the mixture of fitted
conditional distributions over the empirical hazard distribution — never as
an average of conditional quantiles. In-sample this mixture equals the
empirical loss distribution, so IDR's marginal VaR and ES coincide with the
empirical ones in every subsample.

## HANZE Real-Data Case Study

Part 12 of the notebook fits the same estimator families to the real HANZE
v2.1.1 European flood-event database (Paprotny et al.) instead of synthetic
data: it normalizes the raw event table, parses hazard type and rainfall
amount/duration from the free-text cause field, and exposes an interactive
dashboard for filtering by country, NUTS region, year range, and hazard
type, then fitting and comparing the estimators on the selection. Each
event's loss stays a single aggregate figure, not split or allocated across
the regions it lists. Real data has no theoretical target to benchmark
against, so this Part reports fitted values instead of error against theory.
The paper's Spanish example (SI Table S37) is the dashboard with Country =
Spain and every other control at its default (rainfall amount, positive
losses, 10 bins): 53 complete hazard–loss pairs, 1940–2020.

## Documentation

- [Reproducibility guide](docs/reproducibility.md) — setup, how to run,
  what `RUN_FULL_SCALE`/`SAVE_FIGURES` do, where outputs land.
- [Data and output schemas](docs/data.md) — HDF5 population cache, results
  CSVs, `HANZE_events.csv`.
- [Paper ↔ code mapping](docs/paper_mapping.md) — which run and output file
  produce each table and figure of the paper, and which values in the paper
  are not computed here.

The notebook's own markdown cells are the primary documentation of the
methodology (every formula, convention, and design decision is explained
inline, next to the code that implements it) — these docs cover only
what the notebook's prose doesn't: environment setup, output formats, and
the correspondence with the paper.

## Current Release Status

This is research code, not a finished software release. In the current
tree:

- this tree is **v1.1.0**, the code version behind the paper's results;
  **v1.0.0 does not reproduce them** (see "Changes since v1.0.0" below);
- correctness is checked by running the notebook and inspecting its output,
  rather than by an automated test suite;
- `requirements.txt` pins the exact versions from the environment that
  produced the paper's results and data caches;
- the official IDR wheel must be installed separately;
- full-scale sweep result CSVs/HDF5 caches/figures are generated locally and
  kept out of the repository via `.gitignore` (potentially large);
- `CITATION.cff` has author, repository, and Zenodo DOI information for the
  software itself (the concept DOI, which always resolves to the latest
  version); the associated paper's own DOI is added once the paper is
  published.

### Changes since v1.0.0

- `GUMBEL_THETA` setting (overridable by environment variable), with a
  `_theta<θ>` tag on every cache, results and figure filename for θ ≠ 2. The
  paper's θ = 1.1 and θ = 5 results need it (Table 2, SI Tables S5–S8,
  S13–S16, S29–S36).
- The quadratic damage function is truncated at zero, `max(m(h), 0)`, as
  described in the paper's Methods.
- The sensitivity sweep uses the main study's 10 equal-count bins (SI
  Fig. S2) instead of 50.
- A lognormal loss margin (μ = 0, σ = 1) selectable for the full-scale sweep
  (`FULL_SWEEP_LOSS_NAMES=lognorm`; SI Tables S2 and S29–S36).
- Scenario selection for the full-scale sweep (`FULL_SWEEP_LOSS_NAMES`,
  `FULL_SWEEP_HAZARD_NAMES`, `FULL_SWEEP_COPULA_NAMES`), `RUN_ALPHA_SWEEP`, and
  a `SAVE_FIGURES` environment variable for headless runs.
- Part 10e draws its figures at θ = 1.1, 2 and 5; refined risk-diagram and
  upper-tail-zoom figures (Figs. 1 and 2).
- Cached HANZE cause-text parsing (`data/HANZE_events_parsed.csv`); the
  unconditional backfill cells were removed.
- Documentation: the paper ↔ code mapping, the exact `isodistrreg` build,
  and corrected notebook markdown.

## Citation and License

The source data, external IDR implementation, and associated scientific
papers must be cited independently of this repository. `CITATION.cff`
credits this repository's code to its author, Sylvain Rossi — the
associated paper has three authors (Sylvain Rossi, Pasquale Cirillo,
Stefano Battiston); cite the paper itself for that authorship.

This software is archived on Zenodo. The concept DOI
[10.5281/zenodo.22830784](https://doi.org/10.5281/zenodo.22830784) covers all
versions and always resolves to the latest one; each version also has its own
DOI, listed on the Zenodo page. **v1.1.0** is the version behind the paper's
results: to cite the exact code, use the v1.1.0 version DOI shown there. The
earlier v1.0.0 archive,
[10.5281/zenodo.22830785](https://doi.org/10.5281/zenodo.22830785), predates
the changes listed above and does not reproduce the paper. The associated
paper's own DOI will be added here once it's published.

Licensed under the MIT License — see [`LICENSE`](LICENSE).
