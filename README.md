# Climate Losses Damage

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22830785.svg)](https://doi.org/10.5281/zenodo.22830785)

Climate Losses Damage is a research-code repository for studying how damage
functions recover climate-related losses in the upper tail. It combines a
synthetic hazard-loss simulation study based on bivariate extreme-value
copulas with a real-data case study on the HANZE European flood-event
database — both live in **one self-contained notebook**,
`notebooks/Bootstrap Consolidated.ipynb`.

The simulation study compares five estimators:

- a quadratic regression with a significance screen;
- a scaled sigmoid regression;
- monotone isotonic regression, trained on binned data or on the raw
  subsample (a control arm);
- isotonic distributional regression (IDR).

The main quantities of interest are the pointwise conditional mean
`E[L | H = h]`, the exceedance-conditioned mean `E[L | H >= h]`, and loss-risk
summaries such as Value-at-Risk (VaR) and Expected Shortfall (ES).

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

The IDR estimator requires the external `isodistrreg` Python bindings
(`isodistrreg==0.5.1` in the environment that produced this repository's
results). They are not installed by `requirements.txt`, since they are not
published on PyPI. The upstream project currently instructs users to
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
| 10a — Light-scale run | Weibull hazard, Dagum losses, `N=100,000`, 100 bootstrap iterations; cached (HDF5 + CSV) like every other run below | seconds once cached, longer on first run |
| 10b — Full-scale sweep | Every combination of 1 loss margin (Dagum) × 2 hazard margins × 2 copulas (4 scenarios), each at `N=500,000`, 100,000 iterations | very expensive; `RUN_FULL_SCALE = True` by default |
| 10c — Sensitivity sweep | Dagum tail-index/body-shape sweep | ~minutes |
| 10e — Extended diagnostic figures | Upper-tail comparison, hazard-loss risk diagram, IDR conditional law | seconds once cached, longer on first run |

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

For IDR, marginal VaR and ES are computed from the mixture of fitted
conditional distributions over the empirical hazard distribution — never as
an average of conditional quantiles. Empirical/sample-based VaR and ES throughout the notebook use the
coherent superquantile convention, matching the theoretical target they are
benchmarked against.

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

## Documentation

- [Reproducibility guide](docs/reproducibility.md) — setup, how to run,
  what `RUN_FULL_SCALE`/`SAVE_FIGURES` do, where outputs land.
- [Data and output schemas](docs/data.md) — HDF5 population cache, results
  CSVs, `HANZE_events.csv`.

The notebook's own markdown cells are the primary documentation of the
methodology (every formula, convention, and design decision is explained
inline, next to the code that implements it) — these two docs cover only
what the notebook's prose doesn't: environment setup and output formats.

## Current Release Status

This is research code, not a finished software release. In the current
tree:

- correctness is checked by running the notebook and inspecting its output,
  rather than by an automated test suite;
- `requirements.txt` pins the exact versions from the environment that
  produced the committed results/data-cache outputs;
- the official IDR wheel must be installed separately;
- full-scale sweep result CSVs/HDF5 caches/figures are generated locally and
  kept out of the repository via `.gitignore` (potentially large);
- `CITATION.cff` has author, repository, and Zenodo DOI information for the
  software itself; the associated paper's own DOI is still pending, since
  it's not yet published.

## Citation and License

The source data, external IDR implementation, and associated scientific
papers must be cited independently of this repository. `CITATION.cff`
credits this repository's code to its author, Sylvain Rossi — the
associated paper has three authors (Sylvain Rossi, Pasquale Cirillo,
Stefano Battiston); cite the paper itself for that authorship.

This software (v1.0.0) is archived on Zenodo:
[10.5281/zenodo.22830785](https://doi.org/10.5281/zenodo.22830785) — cite
that DOI for the code itself. The associated paper's own DOI will be added
here once it's published.

Licensed under the MIT License — see [`LICENSE`](LICENSE).
