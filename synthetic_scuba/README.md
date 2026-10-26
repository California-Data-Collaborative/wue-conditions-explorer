# Synthetic SCUBA and turf rebate data

> **THIS DATA IS SIMULATED.** Every table in this folder is produced by a random generator. It must not be mistaken
> for real utility records. The retailers, customers, addresses, parcel numbers, rates and coordinates are fictional,
> and every table carries `is_synthetic = TRUE`.

This folder holds a reproducible generator for synthetic water utility data shaped like the California Data
Collaborative SCUBA metadata catalogue and the California Turf Rebate Dataset in Schmitt, Tull and Atwater (2018),
together with a matched difference-in-differences (DiD) estimator of rebate savings and a validation suite.

Nobody knows the true savings of real households, because their counterfactual usage is never observed. The generator
knows each synthetic household's savings only because it planted them, so every stored effect is a **planted**
assumption and not a fact about the world. The validation answers a narrower question than "does the method recover
the truth". It asks whether matched DiD recovers an effect we planted, in a world with turnover, vacancy, missing rows,
drought shocks and selection on conservation attitude, and how robust that recovery is when the planted world changes.
The paper's savings figures are estimates, so they anchor the estimated side of the results, not the planted effect.

## Running it

R 4.3 is installed from the Ubuntu archive and packages come from CRAN (see `renv.lock`). One command runs generation,
matched DiD estimation and the descriptive validation on the default profile (3,000 accounts):

```
Rscript R/run_all.R --cores 4
```

Add `--profile tiny` for a 500-account development run or `--profile large` for 12,000 accounts (outputs go to
`build/<profile>/`). The expensive checks are behind flags and off by default: `--scenarios` (the planted-world scenario
loop), `--refit` (multilevel recovery fit with `brms`), `--sbc` (simulation-based calibration) and `--causal` (the
paper's BSTS method through CausalImpact, as a comparison). `bash tests/test_reproducibility.sh` reruns the pipeline on
one and on four cores and compares file hashes. `Rscript R/seed_check.R --set stage4` reruns stages 1 to 4 with 20
seeds, and `Rscript R/calibrate.R` reruns the bounded anchor calibration (do not rerun it to tune; the result is frozen).

Each numbered script can run alone given the previous outputs:

| Script | Stage | What it does |
|---|---|---|
| `R/01_geography.R` | 2 | Retailers, zip codes, block groups and ACS covariates |
| `R/02_accounts.R` | 2 | Locations active in January 2010, customer classes and property attributes |
| `R/03_occupancy.R` | 2 | Occupant turnover with vacancy, new construction and closures |
| `R/04_climate.R` | 3 | Monthly ET0 and precipitation |
| `R/05_usage.R` | 3 | Untreated monthly usage and the prior predictive check |
| `R/06_treatment.R` | 4 | Rebate selection, planted turf savings and the other rebate types |
| `R/07_billing.R` | 5 | Billing periods, meter reads, budgets, OWRS rate files, bills and the supplier report |
| `R/08_observation.R` | 6 | The observation layer and the published tables in `data/` |
| `R/09_estimate_savings.R` | 7 | Matched DiD estimation from the observed tables only, written to `estimates/` |
| `R/10_validate.R` | 8 | Planted versus estimated comparison, scenarios, flagged checks and `reports/validation.html` |

## Outputs

- `data/`: the observed tables with catalogue field names (`usage.parquet`, `customers.csv`, `rebates.csv`,
  `acs_block_groups.csv`, `supplier_report.csv`, `climate_et_precip.csv`, `rates_owrs/`) and `data_dictionary.csv`,
  which marks every field name as read from the catalogue or inferred.
- `estimates/`: matches, household and pooled DiD estimates, the meta-regression, event study, placebo results,
  other-rebate estimates and sensitivity tables, computed from `data/` alone.
- `truth/`: quantities known to the generator (see `truth/README.md`).
- `reports/`: `validation.html`, `environment.txt`, `calibration_log.csv`, the seed robustness tables, the scenario
  table and the gate files.
- `config/params.yaml`: every parameter with a `source` tag (`catalogue`, `paper` or `assumed`) and a rationale for
  each assumed value; `config/profiles/` holds the `tiny`, `default` and `large` profiles.

## The data-generating process

The model is multilevel and generates potential outcomes, so the untreated and treated usage both exist for every
treated account-month. Five fictional retailers (Surf City, So Cal Suburb, Anytown California, Inland Enclave and
Valley Small Town) contain 12 zip codes and 41 block groups in the default profile. Block group income, year built and
household size covary through a shared block group effect. Accounts receive one of the seven DWR customer classes in
exact configured shares, and single family homes receive a year built, an irrigable area (larger inland, smaller for
newer homes and new construction), a meter size and a pressure zone.

Occupancy comes before usage. Single family homes turn over at about 7 percent per year, with a vacancy gap of zero to
three months billed to the incoming account, a new customer ID, and a redrawn household size, indoor use rate,
overwatering factor and latent conservation attitude. New construction (about 1 percent per year, mostly in So Cal
Suburb and Inland Enclave) adds locations with smaller landscapes, and rare closures end them.

Untreated single family usage in CCF per month is indoor plus outdoor demand times a lognormal error:

```
indoor  = household size * gpcd * days / 748.052                  (x 0.03 when vacant)
outdoor = 0.623 * depth * irrigable area * 0.70 / 0.75 * overwater / 748.052   (x a vacancy fraction when vacant)
depth   = max(ET0 - 0.6 * rain, 0.5 * ET0)                          (the winter irrigation floor)
log y   = log(indoor + outdoor) + retailer + block group + household (incl. -0.10 * attitude)
          + AR(1) household deviation + retailer-by-month shock + conservation effects + slow trend and local level
```

The conservation effects are a voluntary drop of about 9 percent from February 2014 and a mandatory drop from April
2015 that brings usage about 22 percent below 2013. Other classes use simpler structural means with the same error
structure.

Rebate uptake follows a logistic model in income, irrigable area, 2010 to 2011 efficiency and the latent attitude. The
intercept is calibrated exactly so that the post-filter analysis cohort hits its target (175 households by default),
using the same filters the estimator applies to the observed tables. Planted turf savings per square foot follow a
regression whose signs follow Table 3 of the paper, plus a split-normal structural household effect, a behavioral
effect that is redrawn on turnover with correlation 0.5, rare extreme savers and a seasonal cosine pair. Savings start
with the first month that begins on or after the post-inspection date and are capped so that treated usage never falls
below indoor demand. Clothes washer and irrigation controller rebates (about 3 percent each, more likely for
conservation-minded households) save 15 gallons per household per day and 10 percent of outdoor demand. They may
overlap with turf rebates, and their holders stay in the control pool.

Bills use synthetic budget-based rates (Surf City, So Cal Suburb and Inland Enclave) or inclining block rates
(Anytown California and Valley Small Town), written as OWRS files, with integer meter reads and a 4 percent increase
each July. The observation layer then removes feed gaps and a retailer outage, corrupts a few rows, withholds some
household sizes and irrigable areas, bunches defaulted irrigable areas at one value for two retailers, and duplicates
some turf rebate records.

## Matched difference-in-differences

`R/09_estimate_savings.R` defines the pre-period as billing periods ending before the rebate received date and the
post-period as periods starting on or after the post-inspection date, and it keeps single family locations with 24 pre-
and 12 post-months, a single turf record and a non-defaulted irrigable area. Each treated household receives 6 controls
from the same zip code and billing cycle with the highest correlation of pre-period usage (with replacement), and the
household DiD regresses the treated-minus-control difference on a harmonic pair and a post indicator with Newey-West
errors over the first 12 post-months. A stacked DiD in `fixest`, `metafor` pooling, a meta-regression, an event study,
a placebo test, comparison estimators and the other rebate types complete the analysis.

RESULTS_SECTION

## Judgment calls

Every parameter tagged `assumed` in `config/params.yaml` should be reviewed by someone with real data experience,
including every climate and demographic value for the five retailers. The following choices matter most.

- The paper defines pre-removal efficiency with a log ratio but calls values below one conservers; the generator uses
  the log definition, so negative values are conservers. The Table 3 intercept is not used as a mean.
- The winter irrigation floor (0.5 times ET0), the turf area distribution, the class shares, the size of the 2014 and
  2015 conservation effects, the 7 percent non-revenue water, the service populations, the other rebate effects, the
  vacancy irrigation fraction, the post-turnover savings correlation and the overlap probabilities are assumed.
- The paper dates mandatory restrictions to April 2015 (the executive order), while the State Water Board's mandatory
  reductions took effect in June 2015.
- The matching details (raw pre-period correlation, ties broken by mean usage, matching within billing cycle, with
  replacement) are inferred, because the paper could not be reread from this environment.
- The 12-month headline window, the 90 percent coverage rule, the planted-effect tolerances and all scenario
  definitions are assumed.
- A level-based DiD is biased by the multiplicative drought effect, because a proportional drop is larger for heavier
  users, so a log-outcome DiD is reported alongside it.
- The structural household savings effect is drawn by stratified sampling within the treated cohort, so the cohort
  reproduces the configured distribution rather than one noisy draw of it.
- Defaulted irrigable areas are detected by bunching (10 or more accounts with one value in a retailer), which also
  flags the rare home whose measured area equals the default value.
- Vacancy months are billed to the incoming account, as when a new owner takes over service before moving in.
- Meter sizes are stored without the inch mark (for example `3/4`), and the OWRS files use the same keys.

## Sources and access

The SCUBA catalogue was read from `California-Data-Collaborative/metadata_catalogue` (commit `623bace`), and the
field names in `data/` follow its partial data dictionary. Fields not in the catalogue are marked `inferred` in
`data/data_dictionary.csv`: the education category names, `block_group_id`, `cust_loc_unit_count`,
`cust_loc_year_built`, `rebate_instance_id`, the climate table's `cimis_eto_in` and `precip_in`, and `report_month`.
The paper on Project Euclid and the CIMIS websites were blocked from this environment, so the paper's figures and the
ET0 tables come from the instructions and could not be rechecked. The Dockerfile has not been built or tested,
because the Docker daemon is not available here.

LIMITATIONS_SECTION
