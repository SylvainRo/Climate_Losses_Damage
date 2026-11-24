# Paper ↔ code mapping

Where each table and figure of *Damage Functions Underestimate Extreme Climate
Losses* (Rossi, Cirillo, Battiston) comes from in this repository: which run of
`notebooks/Bootstrap Consolidated.ipynb` produces it, and which output file holds
it. Run invocations are in [`reproducibility.md`](reproducibility.md); file
schemas are in [`data.md`](data.md).

The paper's results need **v1.1.0 or later** of this code (see the README's
"Current Release Status"). v1.0.0 lacks the `GUMBEL_THETA` setting, the
zero-truncated quadratic, the 10-bin sensitivity sweep and the lognormal margin.

## Terminology

| Paper | Code / output files |
| --- | --- |
| Monte Carlo subsampling, $S$ subsamples of size $n$ drawn without replacement | "bootstrap" (`run_bootstrap_study`, `iterations` = $S$, `n` = $n$) |
| Mean, Median (SI tables) | `Bootstrap mean`, `Bootstrap median` |
| Mean % Err. | `% error` = $100\,(\bar{\hat\theta}-\theta)/\theta$ |
| Median % Err. | `Median % error` |
| MAE in brackets in Table 2 (% of target) | `MAE / Theory × 100`; the CSV's `MAE` is in loss units |
| Accepted fits (Table 2 footnote) | $S$ − `NaN count` of the Quadratic/Sigmoid rows |
| Oracle loss $\tilde L=d(H)$: $\mathrm{VaR}_q(\tilde L)$, $\mathrm{ES}_q(\tilde L)$ | `Theory` of the pointwise table, `Theory` of the tail-conditional-expectation table |
| Binned quadratic / sigmoid / isotonic, raw isotonic, IDR, empirical subsample | `Quadratic`, `Sigmoid`, `Isotonic`, `Isotonic (raw)`, `IDR`, `Empirical subsample` |

Every simulation run uses the full-scale design of Part 10b: $N=500{,}000$,
$n=500$, $S=100{,}000$, 10 equal-count bins, seed 123, Weibull hazard
$c=2/3$ or GPD hazard $\xi=0.3$, Dagum losses $\alpha=2,\ p=1,\ b=1$ (or
lognormal $\mu=0,\ \sigma=1$), quantile levels $q\in\{0.95,0.99,0.999,0.9999\}$.

## Tables

`<tag>` below is `{copula}_{hazard}_{loss}_N500000_n500_iter100000_bins10[_theta<θ>]`
(no `_theta` suffix for θ = 2 and for Lorenz). Each `results/tables_<tag>/`
folder holds `hazard_var.csv`, `losses_var.csv`,
`losses_pointwise_conditional_expectation.csv`, `losses_es.csv` and
`losses_tail_conditional_expectation.csv`.

| Paper item | Scenario | Run (Part 10b) | Output |
| --- | --- | --- | --- |
| Table 1, rows θ = 1.1, 2, 5 | Gumbel, Weibull, Dagum | `GUMBEL_THETA` = 1.1, 2, 5 | `Theory` columns: true VaR/ES of $L$ from `losses_var.csv`/`losses_es.csv`; $\mathrm{VaR}_q(\tilde L)$ from the pointwise table; $\mathrm{ES}_q(\tilde L)$ from the tail table |
| Table 2 | Gumbel, Weibull, Dagum, θ ∈ {1.1, 2, 5} | as above | `Median % error`, `MAE`/`Theory` and `NaN count` of `losses_var.csv`, `losses_es.csv` and `losses_tail_conditional_expectation.csv` |
| SI Table S2, rows θ = 1.1, 2, 5 | Gumbel, Weibull, lognormal | `FULL_SWEEP_LOSS_NAMES=lognorm`, θ = 1.1, 2, 5 | `Theory` columns, as for Table 1 |
| SI Table S3 | Weibull hazard | any Weibull run | `hazard_var.csv` |
| SI Table S4 | GPD hazard | any GPD run | `hazard_var.csv` |
| SI Tables S5–S8 | Gumbel θ = 1.1, Weibull, Dagum | `GUMBEL_THETA=1.1` | `tables_gumbel_weibull_min_dagum_…_bins10_theta1.1/` |
| SI Tables S9–S12 | Gumbel θ = 2, Weibull, Dagum | default | `tables_gumbel_weibull_min_dagum_…_bins10/` |
| SI Tables S13–S16 | Gumbel θ = 5, Weibull, Dagum | `GUMBEL_THETA=5` | `tables_gumbel_weibull_min_dagum_…_bins10_theta5/` |
| SI Tables S17–S20 | Gumbel θ = 2, GPD, Dagum | default | `tables_gumbel_gpd_dagum_…_bins10/` |
| SI Tables S21–S24 | Lorenz β = 3, Weibull, Dagum | default | `tables_lorenz_weibull_min_dagum_…_bins10/` |
| SI Tables S25–S28 | Lorenz β = 3, GPD, Dagum | default | `tables_lorenz_gpd_dagum_…_bins10/` |
| SI Tables S29–S32 | Gumbel θ = 1.1, Weibull, lognormal | `GUMBEL_THETA=1.1`, `lognorm` | `tables_gumbel_weibull_min_lognorm_…_bins10_theta1.1/` |
| SI Tables S33–S36 | Gumbel θ = 5, Weibull, lognormal | `GUMBEL_THETA=5`, `lognorm` | `tables_gumbel_weibull_min_lognorm_…_bins10_theta5/` |
| SI Table S37 | HANZE, Spain | Part 12 dashboard: Country = Spain, other controls at their defaults (see the Part 12d markdown cell) | the two "Damage-model results" tables |

Within each group of four SI tables the order is: VaR (`losses_var.csv`),
pointwise conditional expectation, ES (`losses_es.csv`), hazard-tail conditional
expectation. Ratios quoted in the main text (e.g. the binned curves' mean ES as a
fraction of the oracle ES, or the Lorenz oracle ES gap of −31.9% to −40.0%) are
computed from these CSVs.

## Figures

With `SAVE_FIGURES=1`, the figures are written to `plots/`:

| Paper figure | Drawn by | File |
| --- | --- | --- |
| Figure 1 | Part 10e, `plot_hazard_loss_risk_diagram_curves(mode="full")`, Weibull, θ = 1.1 | `hazard_loss_risk_diagram_curves_full_gumbel_weibull_min_dagum_N500000_n500_iter100000_bins10_theta1.1.png` |
| Figure 2 | Part 10e, `plot_upper_tail_comparison` (Sigmoid, subsample seed 32), Weibull, θ = 1.1 | `upper_tail_comparison_gumbel_weibull_min_dagum_N500000_n500_iter100000_bins10_theta1.1.png` (`…_lossvar.png` adds the $\mathrm{VaR}_q(L)$ line) |
| SI Figure S1 | Part 10e, `plot_idr_risk_diagram`, Weibull, θ = 2 | `idr_risk_diagram_gumbel_weibull_min_dagum_N500000_n500_iter100000_bins10.png` |
| SI Figure S2 | Part 10c, Dagum α-sweep, Weibull hazard, Gumbel θ = 2, 10 bins, $S=20{,}000$ | `sweep_grid_dagum_alpha_weibull_bins10.png` (data: `results/sweep_dagum_alpha_weibull_iter20000_bins10.csv`) |

Not shown in the paper but produced by the same runs: the GPD-hazard α-sweep
(`sweep_grid_dagum_alpha_gpd_bins10.png`) and the body-parameter p-sweep
(`sweep_grid_dagum_p_bins10.png`; the SI's "auxiliary sweep over the body
parameter p").

## Not computed in this repository

These quantities in the paper are analytical or were computed outside this code:

- Table 1: the rows θ = 1.05 and θ = 1.5, and the "Asymptotic gap" column
  (the closed-form $\kappa_{\alpha,\theta}$, Eq. 2).
- SI Table S1 (asymptotic gap expressions).
- SI Table S2: the rows θ = 1.05 and θ = 1.5, and the asymptotic column.
- SI §2 numerical values: the finite-level τ table of §2.6 (θ = 1.2, 1.25),
  the limiting gaps for other α and θ, $\kappa^{\mathrm{Lor}}_{2,3}\approx0.5965$,
  and the 61-level VaR-ratio grid check.

The finite-level multipliers of SI §2.3 and §2.8 (1.172922, 1.177799 and
1.1485, 1.1400) are ratios of true to oracle VaR/ES and follow from the θ = 2
`Theory` columns listed above. The oracle machinery needed for other θ
(`ConditionalTargets`, Part 6) is in the notebook: for any θ, the oracle VaR and
ES at level `q` are `targets.d(F_H^{-1}(q))` and
`targets.tail_mean(F_H^{-1}(q))`.
