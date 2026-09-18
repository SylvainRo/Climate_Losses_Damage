# Data and output schemas

## `data/HANZE_events.csv`

The HANZE v2.1.1 European flood-event database (Paprotny et al.), event-level
table, one row per aggregate flood event. Download it separately (Zenodo,
https://zenodo.org/records/11259233) and place it at `data/HANZE_events.csv`.
Columns, as delivered upstream (renamed
internally by `HANZE_RENAME_MAP` in the notebook's Part 12a):

| Column | Renamed to | Notes |
| --- | --- | --- |
| `ID` | `event_id` | unique per event |
| `Country code` | `country_code` | ISO-2 |
| `Year` | `year` | |
| `Country name` | `country` | |
| `Start date` / `End date` | `start_date` / `end_date` | parsed as dates |
| `Type` | `flood_type` | |
| `Flood source` | `source` | |
| `Regions affected (v2010)` / `(v2021)` | `region2010` / `region2021` | `;`-separated NUTS codes; `region2021` is the one used for filtering |
| `Area flooded` | `area_flooded_total` | |
| `Fatalities` / `Persons affected` | `fatalities` / `persons_affected` | |
| `Losses (nominal value)` / `(original currency)` / `(2020 euro)` | `losses_nominal` / `currency` / `losses_euro_total` | `losses_euro_total` is the loss variable used throughout Part 12 |
| `Cause` | `cause_raw` | free text; parsed into `hazard_main_type`/`rain_amount_mm`/`intensity_mm_per_h`/etc. by Part 12b |
| `Notes` / `References` / `Changes` | `notes_raw` / `references_raw` / `changes_raw` | |

Each event remains one observation; `losses_euro_total` is that event's
aggregate loss, not split or allocated across the regions it lists.

## Generated population caches: `data/<tag>.h5`

Written by `generate_population_and_subsamples` (Part 8) whenever a
`StudyConfig.run.data_backend == "hdf5"` — by default, every Part 10 run
except 10c's p-sweep (10a, 10b, 10c's alpha-sweep, and 10e all use it).
Three datasets, gzip-compressed:

| Dataset | Shape | Contents |
| --- | --- | --- |
| `hazard` | `(N,)` | the full simulated hazard population |
| `losses` | `(N,)` | the full simulated loss population, jointly dependent with `hazard` via the scenario's copula |
| `sub_indices` | `(iterations, n)` | row indices into `hazard`/`losses` selected for each bootstrap subsample |

If a cache file for a given `<tag>` already exists, it's loaded instead of
regenerated (unless `RunConfig.rebuild_cache=True`).

## Results: `results/pointwise_<tag>.csv`, `results/tail_<tag>.csv`

Written by `save_results` (Part 9), called from 10a, 10b, 10b-bis, and 10e
whenever `load_or_run_bootstrap_study` (Part 9) doesn't already have a
cache hit for that scenario. If both files for a given `<tag>` (plus
`cache_tag_suffix`, where a call site passes one -- e.g. 10e's GPD-hazard
demo uses `"_gpdhazard"`) already exist, `load_or_run_bootstrap_study`
loads them instead of re-fitting every method on every bootstrap subsample
-- the dominant cost of a full-scale run -- unless `RunConfig.rebuild_cache
=True`, or the cached files don't cover the current
`run.methods`/`run.var_levels` (`<tag>` doesn't encode either, so this is
checked explicitly before trusting the cache).

One row per `(subsample_id, method, var_level)`. Both files share the same
columns (`target` distinguishes them: `"pointwise"` or `"tail"`):

| Column | Meaning |
| --- | --- |
| `method` | estimator name (`"Quadratic"`, `"Sigmoid"`, `"Isotonic"`, `"Isotonic (raw)"`, or `"IDR"` as of the current method roster; see the caveat below) |
| `subsample_id` | which bootstrap subsample (0-indexed) |
| `var_level` | the VaR level $q$ this row is evaluated at |
| `h_var_sub` | this subsample's own empirical hazard VaR at $q$ |
| `loss_var_sub` / `loss_es_sub` | this subsample's empirical loss VaR/ES at $q$ (the "Empirical subsample" baseline) |
| `damage_var_sub` / `damage_es_sub` | this method's estimated marginal loss VaR/ES at $q$ |
| `damage_variance_sub` | variance of this method's damage-sample estimate |
| `damage_stat_source` | which code path produced the marginal estimate (`"predicted_damage_sample"`, `"conditional_distribution"`, `"tail_distribution"`, `"mixture_var_es"`, ...) |
| `h_theory_true` | the theoretical hazard VaR at $q$ (not the empirical `h_var_sub`) |
| `prediction` | this method's pointwise (`E[L\|H=h_q]`) or tail (`E[L\|H>=h_q]`) estimate, depending on `target` |
| `true_full` | the theoretical target `prediction` is compared against |
| `target` | `"pointwise"` or `"tail"` |

**Caveat on pre-existing cache files:** `load_or_run_bootstrap_study`'s
cache-hit check only verifies that the cached file's methods are a
*superset* of what's currently requested (`expected_methods.issubset(
cached_methods)`) -- it doesn't check for an exact match. So a
`pointwise_<tag>.csv`/`tail_<tag>.csv` pair written before the `"GPD
regression"`/`"POT ..."` estimators were removed from the notebook still
loads as a valid cache hit today, and still physically contains rows for
those retired methods. This doesn't corrupt anything displayed --
`build_summary_tables` filters rows to the currently-registered method
list, so retired-method rows are silently ignored -- but the file itself
stays larger than a from-scratch run would produce until it's regenerated
(delete it, or set `RunConfig.rebuild_cache=True`, to force that).

## Results: `results/sweep_dagum_<param>_<hazard>[_iter<n>].csv`

Written incrementally by `load_or_run_sensitivity_sweep` (Part 10c), one
row appended per `(parameter_value, method, risk_measure)` combination as
the alpha-sweep computes it -- `<param>` is the swept Dagum parameter
(`alpha` in the current notebook), `<hazard>` is `weibull` or `gpd`, and
`_iter<n>` is the bootstrap iteration count folded into the filename
because the function's own row-level cache key (`parameter_value` alone)
doesn't otherwise distinguish runs at different iteration counts. The same
superset-only cache-hit behavior and stale-row caveat above applies here
too: an already-cached `parameter_value` is never recomputed once present,
even if it was computed under a now-retired method. Columns:

| Column | Meaning |
| --- | --- |
| `distribution` | the swept loss margin (`dagum`) |
| `parameter` | the swept parameter name (e.g. `alpha`) |
| `parameter_value` | this row's grid value |
| `method` | estimator name, or `"Empirical subsample"` |
| `risk_measure` | `"VaR"`, `"ES"`, or `"Tail CE"` |
| `q` | the fixed risk level (`SWEEP_Q`) the sweep is evaluated at |
| `true_value` | theoretical target at this grid point |
| `mean_estimate` / `median_estimate` | across bootstrap iterations |
| `pct_error` / `median_pct_error` | relative to `true_value` |
| `n_total` / `n_valid` | iterations attempted vs. finite |
| `failure_rate_pct` | share of iterations that failed to produce a finite estimate |

The p-sweep (also Part 10c) has no equivalent file -- it isn't cached, so
nothing from it is ever written to `results/`.

## Results: `results/tables_<tag>/*.csv`

One CSV per summary table from `build_summary_tables` (Part 9), each
aggregating the records above across bootstrap subsamples, per
`(method, var_level)`: `hazard_var.csv`, `losses_var.csv`,
`losses_pointwise_conditional_expectation.csv`, `losses_es.csv`,
`losses_tail_conditional_expectation.csv`. Columns:

| Column | Meaning |
| --- | --- |
| `Estimator` | method name (or `"Empirical subsample"`) |
| `VaR level` | $q$ |
| `Theory` | theoretical target value |
| `Bootstrap mean` / `Bootstrap median` | across bootstrap subsamples |
| `Std` | bootstrap standard deviation |
| `Bias` | `Bootstrap mean - Theory` |
| `% error` / `Median % error` | bias as a percentage of `Theory` (mean- and median-based) |
| `RMSE` / `MAE` | against `Theory`, across subsamples |
| `NaN count` | how many subsamples this method failed to produce a finite estimate for |

`<tag>` (used by the `data/<tag>.h5`, `results/pointwise_<tag>.csv`/
`tail_<tag>.csv`, and `results/tables_<tag>/` sections above -- not the
`results/sweep_dagum_...` files, which use their own `<param>`/`<hazard>`/
`<n>` naming) is `StudyConfig.cache_tag()`:
`{copula}_{hazard}_{loss}_N{N}_n{n}_iter{iterations}_bins{bins}`, e.g.
`lorenz_gpd_dagum_N500000_n500_iter100000_bins10` (the full-scale sweep's
loss margin is always `dagum`; see the "10b" row of the README's Part 10
table for the current scenario grid).

## Plots: `plots/<name>.png`

Only written if `SAVE_FIGURES=True` (Part 3, off by default). One PNG per
figure, named after what it shows — most embed the producing scenario's
`cache_tag()` (e.g. `joint_sample_<tag>.png`) so different scenarios never
overwrite each other; a few figures tied to a single interactive tool
(the HANZE dashboard) use a fixed name instead, since only the latest render
is meaningful there (e.g. `hanze_hazard_loss_fits.png`).
