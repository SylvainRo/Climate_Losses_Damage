# Reproducibility guide

Everything in this repository runs from one notebook,
`notebooks/Bootstrap Consolidated.ipynb`. Every distribution, copula,
estimator, and plotting routine is defined inline in the notebook — nothing
beyond `requirements.txt` needs installing.

## Environment

Python 3.13 is required — the official IDR backend's wheel is built
specifically for CPython 3.13 (`cp313-abi3`) and will not install under
3.12.

```bash
python3.13 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

`requirements.txt` pins the exact versions from the environment that
produced this repository's committed results/data-cache outputs. One
dependency is not in `requirements.txt` and needs a separate step:

- **`isodistrreg`** (the official IDR backend, needed only when the `"IDR"`
  method is actually fit): not published on PyPI. The reproducing
  environment used a wheel reporting version `0.5.1` that is **not** the
  tagged `v0.5.1` release, but an upstream CI build of `master` at commit
  `a95ba86b03bdac83e1dbd1a0de276d41b562b588` (2026-07-10, 18 commits after
  `v0.5.1`, before `v0.5.2`); wheel
  `isodistrreg-0.5.1-cp313-abi3-manylinux_2_34_x86_64.whl`, sha256
  `d571e7c59f87fa76e691a3f90a7bdec79a51e0264907443ce1a727cf0af5a783`. To
  reproduce that exact version, build it from source at the commit (needs a
  Rust toolchain): `python -m pip install
  "git+https://github.com/AlexanderHenzi/isodistrreg@a95ba86b03bdac83e1dbd1a0de276d41b562b588#subdirectory=bindings/python"`.
  Alternatively, install the platform-specific wheel from a recent GitHub
  Actions run of the upstream project — see the [official installation
  guide](https://github.com/AlexanderHenzi/isodistrreg/blob/master/bindings/python/README.md).
  Without it, the notebook still imports cleanly, but any run that includes
  `"IDR"` in its method list will fail when it tries to fit.

`ipywidgets` *is* in `requirements.txt`, but is only exercised by Part 12
(the HANZE dashboard) — every earlier Part runs without it.

`h5py` and `joblib` are also optional (wrapped in `try/except ImportError`
in Part 1): `h5py` is needed for `data_backend="hdf5"` runs, which by
default is every Part 10 run except 10c's p-sweep (see "Where outputs land"
below); `joblib` only speeds up the bootstrap loop across subsamples —
without it, the evaluation engine falls back to a plain sequential loop,
just slower.

## Running the notebook

```bash
cd notebooks
jupyter lab "Bootstrap Consolidated.ipynb"
```

Run top to bottom. Parts 1–9 only define things (no expensive computation);
Part 10 is where runs happen. **10a (light-scale)** always runs — a small
scenario (`N=100,000`, 100 iterations) that's cheap on a fresh run and
effectively free on any subsequent run, since it goes through the same
HDF5 + CSV caching described below. **10e** (the extended diagnostic
figures) is the same story. **10c** (the sensitivity sweep) takes minutes
to tens of minutes depending on how much of it is already cached. **10b**
is gated behind an explicit flag because it's expensive even cached:

### `RUN_FULL_SCALE` (Part 10b)

**On (`True`) by default.** When true, the notebook runs a production-scale
bootstrap study — `N=500,000` population, `iterations=100,000` bootstrap
subsamples — for **every** combination of:

- loss margin: Dagum (1),
- hazard margin: Weibull, or a Generalised Pareto hazard with shape
  $\xi=0.3$ (2),
- copula: Gumbel or Lorenz (2),

i.e. 4 scenarios in total, each independently as expensive as a single
full-scale run. This is meant to be started and left running unattended
(hours on an uncached first run, not something to babysit in a live
session) — set it to `False` if you just want to explore the notebook
interactively without paying that cost. Each scenario is cached
independently (see `docs/data.md`), so an interrupted sweep — or a rerun
after the first one — picks up where it left off instead of redoing
already-cached scenarios.

### `SAVE_FIGURES` (Part 3)

Off (`False`) by default. When set to `True`, every plotting function in
the notebook additionally writes its figure to `plots/` as a PNG, named
after what it plots and (where relevant) which scenario produced it — e.g.
`joint_sample_gumbel_weibull_min_dagum_N100000_n500_iter100_bins50.png`.
With it off (the default), figures only render inline in the notebook, as
before.

## Where outputs land

All three are sibling folders to `notebooks/`, located automatically
(`_find_project_root`, Part 1) by walking up from the notebook's working
directory until a `data/` folder is found — this works whether Jupyter is
started from the repository root or from inside `notebooks/`.

| Folder | What goes there | Written by |
| --- | --- | --- |
| `data/` | `HANZE_events.csv` (you provide this, see `docs/data.md`); generated `<tag>.h5` population/subsample caches | Part 8's `generate_population_and_subsamples`, whenever `data_backend="hdf5"` — every Part 10 run (10a, 10b, 10c's alpha-sweep, 10e) except 10c's p-sweep, which stays in-memory |
| `results/` | `pointwise_<tag>.csv`, `tail_<tag>.csv`, `tables_<tag>/*.csv`; also `sweep_dagum_<param>_<hazard>[_iter<n>].csv` from 10c's alpha-sweep (see `docs/data.md`) | `save_results`, called from 10a, 10b, 10b-bis, and 10e whenever their `load_or_run_bootstrap_study` call misses its cache |
| `plots/` | `<name>.png` per figure | `save_figure`, called from every plotting function, only when `SAVE_FIGURES=True` |

`<tag>` is `StudyConfig.cache_tag()` — copula, hazard, loss margin names and
the population/subsample/bin sizes, e.g.
`gumbel_weibull_min_dagum_N500000_n500_iter100000_bins10`. Two different
scenarios never collide on the same cache/results/plot filename, *unless*
they share every one of those fields but differ in a distribution parameter
value, `seed`, `var_levels`, or `methods` — `cache_tag()` doesn't encode
those, so such cases need a manually-chosen `cache_tag_suffix` to stay
disambiguated (10e's GPD-hazard demo scenario does this, via `"_gpdhazard"`).

## Interpreting results

See the README's "Interpreting the Evaluation" section for the pointwise/
tail/VaR/ES conventions, and the notebook's own Part 6/8 markdown cells for
the full mathematical detail (theoretical targets, the coherent
superquantile ES convention, IDR's mixture-distribution VaR/ES). This guide
only covers setup and where things end up on disk — the methodology is
documented inline, next to the code that implements it, to avoid two copies
of the same explanation drifting out of sync.
