# Penetrance Bug Review

## Compact Rule

On `/compact`, update this file with all important context (bug
statuses, decisions, key facts, new bugs) before compacting, so this
file serves as the full context restore point for future sessions.

## Style Rule

**The goal of all refactoring and bug fixing is to make the code clearer
and more cohesive. Refactoring must never change public, user-facing
behavior — this includes function signatures, parameter names, defaults,
and the way
[`penetrance()`](https://nicokubi.github.io/penetrance/reference/penetrance.md)
is called.** All fixes must match the existing code structure as closely
as possible. Use the same brace style, error message format
(`"Error: 'param_name' ..."`), and variable naming conventions as the
surrounding code. **Never introduce ambiguous variable names**
(e.g. single letters or meaningless abbreviations) — new variables must
be as descriptive as the ones around them and consistent with
package-wide conventions (e.g. `noSex` not `g`). **Remove pending fix
details from this file once a bug is fixed — keep only the status
table.**

## Key Facts

- `_samples` = per-chain output from
  [`mhChain()`](https://nicokubi.github.io/penetrance/reference/mhChain.md);
  `_results` = combined output from `combine_chains[_noSex]()`
- Weibull uses shift:
  `dweibull(age - threshold, shape=alpha, scale=beta)`. At
  `age==threshold`, arg=0; `dweibull(0, shape<1)=Inf` (not NaN — R
  guards `x<0` → 0, but not `x=0`)
- `length(df)` = ncols, not nrows. Use
  [`nrow()`](https://rdrr.io/r/base/nrow.html) or `df[[1]]` to get
  vector length.
- `sex_specific` determined by presence of
  `combined_chains$median_male_results`
- Beta distribution requires both shape params \> 0; `normalize==1` →
  `beta = at_risk - at_risk = 0` is invalid (Bug 7)
- To extract a data frame column as a numeric vector:
  `as.numeric(df[[1]])` — matches `as.numeric(df[, "Male"])` pattern
  used in sex-specific branch
- `claude.md` is the sole context store (`.claude/memory/` folder
  deleted as redundant)
- `dweibull(0, alpha<1)=Inf` only fires when threshold is exactly an
  integer — confirmed with alpha=0.742407, beta=26.68247, threshold=33.
  Integer thresholds arise from
  [`quantile()`](https://rdrr.io/r/stats/quantile.html) on integer ages
  at MCMC initialization and persist through rejected proposals; can
  survive burn-in.
- Bug 9 fix does two things: (1) multi-column error check (redundant
  with `helpers.R:441` for
  [`penetrance()`](https://nicokubi.github.io/penetrance/reference/penetrance.md)
  path, but guards direct `mhChain` calls); (2)
  `as.numeric(baseline_data[[1]])` conversion (still needed —
  `helpers.R` validates but does not convert before passing to
  `mhChain`, so without it Bug 5 reappears).
- Original unmodified CRAN files are in `penetrance_v0.1.3_ORIGINAL/`
  (renamed from `penetrance_v0.1.3_CRAN_release 2/`). Diff confirms all
  and only the documented changes are present.
- To load local files for testing:
  `detach("package:penetrance", unload=TRUE)` if loaded, then
  [`library(clipp); library(MASS); library(kinship2)`](https://rdrr.io/r/base/library.html),
  then [`source()`](https://rdrr.io/r/base/source.html) all R files.
  Check with `exists("init_one_group")`.
- For `sex_specific=FALSE` testing: use `var=c(0.1,2,5,5)`,
  single-column baseline
  (`data.frame(risk=rowMeans(baseline_data_default[,c("Female","Male")]))`),
  and fix `simulated_families` parent IDs (`MotherID/FatherID == 0` →
  `NA`).

------------------------------------------------------------------------

## Bug Status

| \# | File | Status | One-line summary |
|----|----|----|----|
| 1 | `outputHelpers.R:111` | ✅ FIXED | `generate_density_plots` non-sex branch used `_samples` instead of `_results` |
| 3 | `outputHelpers.R:535,547` | ✅ FIXED | `dweibull(0, alpha<1)=Inf` breaks ylim and plot in [`plot_pdf()`](https://nicokubi.github.io/penetrance/reference/plot_pdf.md) |
| 4 | `outputHelpers.R:~467,~597` | ✅ FIXED | Legend color logic inverted in `plot_penetrance` and `plot_pdf` |
| 5 | `mhChain.R:~270` | ✅ FIXED | `length(single_col_df)==1` not nrows; fixed as side effect of Bug 9 fix |
| 7 | `priorElicitation.R:88–107` | ❌ DISMISSED | `normalize==1` → `beta=0` invalid Beta param in all 3 `compute_parameters_*` fns |
| 8 | `priorElicitation.R:109` | ✅ FIXED | [`makePriors()`](https://nicokubi.github.io/penetrance/reference/makePriors.md) accepts negative/invalid `prior_params` without validation |
| 9 | `mhChain.R:267` | ✅ FIXED | Multi-col `baseline_data` silently uses first col when `sex_specific=FALSE` |
| 10 | `penetranceMain.R:180` | 🔲 PENDING | Default `var` is length-8 (sex-specific), always errors when `sex_specific=FALSE` without explicit `var` |

------------------------------------------------------------------------

## Pending Fix Details

**Bug 10** — `penetranceMain.R` line 180: default
`var = c(0.1, 0.1, 2, 2, 5, 5, 5, 5)` is length-8, always fails
validation when `sex_specific=FALSE` without explicit `var`.

``` r
# Change default in function signature from:
var = c(0.1, 0.1, 2, 2, 5, 5, 5, 5),
# to:
var = NULL,
# Then add before the existing var validation (~line 268):
if (is.null(var)) {
  var <- if (sex_specific) c(0.1, 0.1, 2, 2, 5, 5, 5, 5) else c(0.1, 2, 5, 5)
}
```

------------------------------------------------------------------------

## Refactoring

### Status Table

| \# | File | Status | One-line summary |
|----|----|----|----|
| R1 | `mhChain.R:484–722` | ✅ DONE | Main MH loop written twice under sex_specific branch |
| R2 | `mhChain.R:153–343` | ✅ DONE | Two near-identical `draw_initial_params` closures |
| R3 | `mhLoglikehood.r:158–224,417–467` | ✅ DONE | `lik.fn` and `lik_noSex` share ~60% identical skeleton |
| R4 | `outputHelpers.R:355–602` | 🔲 PENDING | `plot_penetrance` and `plot_pdf` differ only in `pweibull` vs `dweibull` |
| R7 | `penetranceMain.R:201–410` | 🔲 PENDING | Validation block repeats identical patterns for columns and integer params |

------------------------------------------------------------------------

### Refactoring Details

**R4** — `outputHelpers.R`
[`plot_penetrance()`](https://nicokubi.github.io/penetrance/reference/plot_penetrance.md)
and
[`plot_pdf()`](https://nicokubi.github.io/penetrance/reference/plot_pdf.md),
lines 355–602: Two ~120-line functions with identical structure
(parameter extraction, `calculate_ylim` inner helper, sex-dispatch
block, legend call). The only meaningful differences are the
distribution function (`pweibull` vs `dweibull`) and y-axis label. -
**Approach**: A single internal
`.plot_weibull_curve(type = c("cdf", "pdf"), ...)` that selects the
distribution function by `type`; both public functions become thin
wrappers.

**R7** — `penetranceMain.R` validation block, lines 201–410: Two
identical patterns: (a) `CurAge` and `Age` column validation blocks are
character-for-character identical except the column name; (b) the
positive-integer check for `n_chains`, `n_iter_per_chain`, `ncores`,
`thinning_factor` repeats the same 4-part condition 4 times. -
**Approach**: A `validate_age_column(df, col_name)` helper called twice;
an `is_positive_integer(x)` helper used in a loop over the four numeric
params.

------------------------------------------------------------------------

## How mhChain Works

`mhChain` runs one full MCMC chain. The package runs `n_chains` of these
in parallel via `parLapply`, then combines them in `combine_chains`.

**Goal**: estimate the four Weibull parameters (asymptote, threshold,
median, first_quartile) that define the penetrance curve
`P(affected by age t | carrier) = asymptote × pweibull(t - threshold, shape=α, scale=β)`.
With `sex_specific=TRUE` there are 8 parameters (one set per sex).

**Bayesian setup**: the likelihood is `P(pedigree data | θ)` computed by
clipp. The prior is the Weibull parameter distributions from
`makePriors`. The posterior `P(θ | data) ∝ L(data | θ) × prior(θ)` is
what the chain samples from. The likelihood is not ignored in Bayesian
inference — Bayes multiplies it by the prior; the prior is what
distinguishes it from MLE.

**Likelihood structure**: the product is over families (pedigrees), not
individuals. Within each family clipp marginalizes over unknown
genotypes using Mendelian transmission probabilities (`trans`),
accounting for genetic dependence between relatives. Across families,
independence is assumed. A likelihood contribution of 1 for a data point
(unknown age set to 1, proband `aff` set to NA) is multiplicatively
neutral — it cancels in the acceptance ratio and has zero influence on
inference.

**Pre-loop setup**: - Age imputation initialization (or unknown ages set
to 1 to be ignored) - Proband affection set to NA to correct for
ascertainment bias - `draw_initial_params`: initializes chain starting
point from sample quantiles of affected individuals’ ages (10th
percentile → threshold, median → median, 25th percentile →
first_quartile). This is a convenience heuristic, not a prior statement
— it just gets the chain started somewhere sensible. - `C = diag(var)`:
initial proposal covariance, diagonal (parameters proposed
independently). Becomes adaptive during the run. -
`calculate_log_prior`: evaluates log-prior density; parameters are
scaled before being passed to Beta/Uniform distributions. - `geno_freq`,
`trans`: fixed constants for the clipp likelihood.

**Each iteration**: 1. **Build proposal** (stays branched by
`sex_specific`): packs current params into `params_vector`, draws
`proposal_vector = mvrnorm(1, mu=params_vector, Sigma=C)`, reflects
asymptote(s) back into (0,1) if out of range, records raw proposals,
builds `params_proposal`. 2. **Evaluate current state**:
`call_loglikelihood(params_current)` +
`calculate_log_prior(params_current)`. 3. **Validate proposal**:
`check_proposal_valid(proposal_vector)` — hard constraints (asymptote in
(0,1), threshold in prior bounds, first_quartile \> threshold, median \>
first_quartile, median \< max_age). 4. **MH accept/reject**: if valid,
compute
`log α = [log L(proposal) + log π(proposal)] − [log L(current) + log π(current)]`.
Accept with probability `min(1, exp(log α))`. On acceptance
`params_current ← params_proposal`; on rejection `params_current` stays,
`num_rejections` increments. 5. **Adaptive covariance**: after burn-in,
`C` updates to the empirical covariance of all states seen so far
(Haario adaptive MH). The proposal automatically tunes to the posterior
geometry. 6. **Store**: `store_samples(params_current, i)` writes the
current parameters into `out$*_samples[i]` — every iteration is stored,
including burn-in.

**Output**: `out$*_samples` is a full trace of length `n_iter`. Burn-in
is stripped later in `combine_chains`. Each step produces one posterior
sample. With 4 chains × 1000 iterations × 10% burn-in = 3600 posterior
samples pooled together.

**Final penetrance curve**: median of each parameter across all
posterior samples, plugged pointwise into the Weibull CDF. Credible
interval bands come from taking quantiles of the curve across all
posterior samples at each age.

**Multiple chains**: each gets a different seed → different
initialization and proposal sequence. Stochasticity handles within-chain
exploration; multiple chains provide between-chain diversity and
convergence diagnostics (if all chains agree on the posterior shape,
exploration was adequate).

------------------------------------------------------------------------

## Dismissed

- **[`plot_trace()`](https://nicokubi.github.io/penetrance/reference/plot_trace.md)
  empty**: Both branches are placeholder comments, no plots produced.
  Noted, not prioritized.
- **`penetranceMain.R` n_chains**: Warning then stop for same condition
  (~244 vs ~430). Inconsistency noted, not prioritized.
- **Parameter name shadowing**:
  `plot_trace`/`plot_loglikelihood`/`plot_acf` params shadow function
  names — R dispatch handles correctly, not a bug.
- **Bug 2 (`outputHelpers.R` burn_in)**: Investigated; no actual runtime
  error found. Removed from bug list.
- **Bug 7 (`priorElicitation.R` Beta params)**: `ratio = NULL` with real
  `distribution_data` is not a valid combination in practice — `ratio`
  is required to anchor the asymptote prior, and the default
  `distribution_data_default` has all NAs so
  `compute_parameters_asymptote` is never reached in normal usage.
- **Bug 6 (`mhLoglikehood.r:220,464`)**: `geno[i]=="1/1"` crashes on NA
  — dismissed because `helpers.R:172` converts all NA genotypes to `""`
  before data reaches the likelihood function. `"" == "1/1"` returns
  FALSE safely.
- **Bug 10 (`mhChain.R` rejected proposals)**: `log_acceptance_ratio`,
  `loglikelihood_proposal`, `logprior_proposal` store 0 instead of NA on
  rejection. These fields are diagnostic-only and never consumed by the
  package’s inference or plotting functions — no fix needed.
