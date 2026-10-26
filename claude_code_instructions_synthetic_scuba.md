# Synthetic SCUBA and Turf Rebate Data: Instructions for Claude Code

## 1. Goal and deliverables

Build a reproducible generator for fully synthetic water utility data that mirrors the schema of the California Data Collaborative SCUBA metadata catalogue and the structure of the California Turf Rebate Dataset (CTRD) described in Schmitt, Tull and Atwater (2018), "Extending Bayesian structural time-series estimates of causal impact to many-household conservation initiatives," Annals of Applied Statistics 12(4), 2517-2539. The paper is available at https://projecteuclid.org/journalArticle/Download?urlId=10.1214%2F18-AOAS1166 and the catalogue is at https://github.com/California-Data-Collaborative/metadata_catalogue. Both documents are the source of the bounds in this specification. The data must carry known ground truth, including the true effect of turf removal for every treated household, so that analysts can test the matching, Bayesian structural time series and meta-regression pipeline (called M123 in the paper) against a known answer.

The deliverables are a config file holding every parameter, a numbered set of generation scripts, the output tables, a truth folder, a validation report, an environment record, and a README that documents the data-generating process. Use R as the default language because the CaDC tools (RateParser, OWRS) are written in R, with `cmdstanr` or `brms` for the refit checks. If R is not available in the environment, use Python with numpy, pandas and CmdStanPy, and say so in the README.

The first build is small: 3,000 accounts spread across 5 fictional retailers, described in section 3. The code must treat the account count, retailer list and treated share as configuration, so a later run can scale up without code changes.

Everything produced is synthetic. Do not request, read or embed any real customer data. Do not use real utility names, real street addresses or real assessor parcel numbers. Use only the five fictional retailer names given in section 3, and add a column `is_synthetic = TRUE` to every table, plus a banner at the top of the README stating that the data is simulated and must not be mistaken for real records.

## 2. How to treat the source documents as bounds

Every parameter in `config/params.yaml` must carry a `source` tag with one of three values: `catalogue` (taken from the SCUBA metadata catalogue), `paper` (taken from the AOAS paper), or `assumed` (chosen by you because the sources are silent). Keep a short `rationale` string beside each assumed value. Where a value is assumed, prefer widely cited physical or demographic reasoning and mark it so the user can overrule it. The table below lists the bounds the generated data must respect.

| Quantity | Bound or target | Source |
|---|---|---|
| Billing panel window | Monthly bills from January 2010 to April 2016, 72 months | paper (figures) |
| Treated analysis cohort in the paper | 545 households that pass the filters below | paper |
| Treated analysis cohort in this 3,000-account build | About 150 to 200 households that pass the filters, set through a configurable treated share so a larger run can reach about 545 | assumed scaling of paper |
| Filters for the analysis cohort | At least 24 pre-rebate months and at least 12 post-rebate months | paper |
| Control candidates | At least 100 non-rebate accounts in the same zip code, 6 matched controls per treated account | paper |
| Rebate payment | About $2 per square foot of turf removed | paper |
| Unit conversion | 1 CCF equals 748.052 gallons, 1 acre-foot equals 435.6 CCF | paper, standard |
| Mean savings | About 20 to 25 gallons per square foot of turf removed per year | paper |
| Seasonal savings | About 2.7 gal/sqft/month in July to September, about 1.5 in January to March | paper |
| Spread of annual household savings | Medians near 12 to 22 gal/sqft/yr, interquartile range roughly 8 to 32, rare extreme savers near 100, and a small share of households with no savings or dissavings | paper (Figure 3) |
| Customer classes | Seven DWR classes: single family residential, multi-family residential, commercial, industrial, irrigation, institutional, other | catalogue |
| Water type | Potable, raw, or recycled | catalogue |
| Billing frequency | Monthly for some utilities, bimonthly for others | catalogue |
| Usage units | CCF per account, acre-feet per supplier-month, ET in inches, area in square feet | catalogue |
| Block group covariates | Population density, total population, total households, average household size, population 25 and over, education counts, median household income, median year structure built | catalogue |
| Drought shock | Mandatory restrictions begin in April 2015, producing a common behavioral drop for all accounts | paper |

The analysis cohort in the paper is restricted to single family residential accounts, so generate the treated cohort and its matched controls as single family accounts, but also generate a minority of multi-family, commercial, institutional and irrigation accounts so the class field is exercised.

## 3. The five retailers and their climates

Generate 3,000 accounts across the five retailers below. Zip codes nest within retailers and census block groups nest within zip codes, giving about 12 zip codes (roughly 250 accounts each, which keeps the control pool above 100 per zip) and about 40 block groups. All values in this section are `assumed`, based on approximate regional climate normals and demographics, and the user should replace them with CIMIS station averages and census data when real figures are wanted. If network access is available, the monthly ET0 values may be calibrated to published CIMIS monthly averages for representative stations, with the source recorded in the config. Otherwise use the values given here.

| Retailer | Setting | Zips | Accounts | Billing | Median household income | Median irrigable area (sq ft) | Median year built |
|---|---|---|---|---|---|---|---|
| Surf City | Coastal, small lots, marine layer, mild summers | 2 | 500 | monthly | $95,000 | 2,800 | 1968 |
| So Cal Suburb | Inland coastal plain, master-planned tracts | 3 | 800 | monthly | $110,000 | 4,200 | 1989 |
| Anytown California | Moderate, mixed older and newer housing, near the coast | 3 | 700 | bimonthly | $80,000 | 4,000 | 1975 |
| Inland Enclave | Hot interior foothill community, large lots | 2 | 600 | monthly | $120,000 | 7,500 | 1996 |
| Valley Small Town | Central Valley, hot dry summers, cool foggy winters | 2 | 400 | bimonthly | $55,000 | 5,000 | 1972 |

Use monthly reference evapotranspiration (ET0, inches per month) with the following long-run monthly means as the climate baseline for each retailer. Coastal retailers have a flatter annual cycle and inland and valley retailers have a pronounced summer peak.

| Month | Surf City | So Cal Suburb | Anytown California | Inland Enclave | Valley Small Town |
|---|---|---|---|---|---|
| Jan | 2.0 | 2.4 | 1.7 | 2.1 | 0.9 |
| Feb | 2.3 | 2.8 | 2.2 | 2.7 | 1.7 |
| Mar | 3.5 | 4.1 | 3.5 | 4.2 | 3.3 |
| Apr | 4.5 | 5.2 | 4.6 | 5.5 | 5.0 |
| May | 5.2 | 6.0 | 5.6 | 6.7 | 7.2 |
| Jun | 5.6 | 6.5 | 6.2 | 7.6 | 8.5 |
| Jul | 6.0 | 7.1 | 6.7 | 8.3 | 9.1 |
| Aug | 5.8 | 6.8 | 6.2 | 7.7 | 8.0 |
| Sep | 4.6 | 5.3 | 4.9 | 5.9 | 5.9 |
| Oct | 3.4 | 3.9 | 3.4 | 4.1 | 3.7 |
| Nov | 2.4 | 2.8 | 2.1 | 2.6 | 1.6 |
| Dec | 1.9 | 2.2 | 1.5 | 1.9 | 0.8 |
| Annual | about 47 | about 55 | about 49 | about 59 | about 56 |

Add a smaller monthly precipitation series for each retailer, concentrated from November through March, with annual totals of roughly 12 inches for Surf City, 13 for So Cal Suburb, 17 for Anytown California, 11 for Inland Enclave and 11 for Valley Small Town. Let effective rainfall offset outdoor irrigation demand in winter and spring. Add year-level anomalies to both series that are shared across retailers with local noise, and make water years 2012 to 2015 drier and slightly higher in ET0 than the baseline, to match the drought context of the paper. Each retailer reports its own ET value (`usage_et_amount`) with small measurement differences, and a smoothed nearby-station value (`usage_et_amount_default`) as in the catalogue. Climate must also shape the other fields: inland and valley retailers get larger irrigable areas, stronger summer outdoor peaks, and higher outdoor shares of annual use, while coastal retailers get smaller outdoor shares.

## 4. The generative model

State the model in the README before writing code, as Gelman and Hill recommend, and keep the code a literal translation of that statement. The model is multilevel and generates potential outcomes, so that both the untreated and treated usage exist for every account-month in the truth folder.

**Geography and covariates.** Create the five retailers, their zip codes and their block groups from section 3. Draw block group median household income from a lognormal distribution centered on the retailer median and bounded between $25,000 and $250,000, median year built between 1940 and 2010, and average household size between 2 and 4. Let income, year built and household size covary through a shared block group effect. Generate fake latitude and longitude from a synthetic grid for each block group and jitter accounts within it. Generate synthetic parcel identifiers that cannot resolve to real parcels.

**Accounts.** Nest accounts in block groups. Draw household size from a truncated Poisson shifted to start at 1 and centered on the block group average, irrigable area in square feet from a lognormal distribution centered on the retailer median (bounded between 500 and 40,000, with a positive link to income and an inverse link to year built), meter size, and a pressure zone. Include a latent, unobserved conservation attitude `a_i` drawn from a normal distribution with a block group mean. This latent attitude lowers baseline usage and raises the probability of taking a rebate, which creates the selection bias that the paper's time series matching is designed to handle.

**Untreated monthly usage.** Model usage in CCF as indoor plus outdoor demand. Indoor demand equals household size times a household-specific gallons per capita day (assumed lognormal near 55) times days in the period, divided by 748.052. Outdoor demand equals 0.623 gallons per square foot per inch of net ET (ET0 minus effective rainfall, floored at zero), times the turf and landscape area, times a plant factor, divided by an irrigation efficiency, then divided by 748.052. Multiply the sum by a lognormal error that combines four pieces: a household random intercept that includes `a_i`, a household-level AR(1) deviation, a retailer-by-month shock, and a common drought effect that begins in April 2015. Add a slow local level and trend term that mirrors the local linear trend in the paper's BSTS model. Use varying intercepts and slopes by retailer and block group, with partial pooling and explicit hyperparameters, as in chapters 12 and 13 of Gelman and Hill.

**Treatment.** Choose treated accounts with a logistic selection model in income, irrigable area, pre-period efficiency, and the latent attitude, targeting roughly 6 to 8 percent of single family accounts so that about 150 to 200 pass the analysis filters. Draw rebate dates so that post-inspection dates fall mainly between April 2012 and April 2015, and add a smaller number of later rebates that fail the 12-month post filter. The true household-level savings per square foot of turf removed follows a regression whose coefficient signs follow Table 3 of the paper: more savings for households that were inefficient before removal, a diminishing return in the log of rebate quantity and area ratio, less savings per square foot in higher income areas, more savings when ET is high, plus a seasonal cosine pair. Scale the intercept so the annual mean lands near 20 to 25 gal/sqft/yr and the monthly means land inside the 1.5 to 2.7 range above. Do not copy the Table 3 intercept literally, because it must be read together with covariates such as the log of median income. Add a household random effect and a small share of extreme savers. Enforce physical feasibility: treated usage can never fall below the household's indoor demand, and savings can never exceed the untreated outdoor demand.

**Billing.** Compute `usage_total_bill` deterministically from usage with a synthetic tiered or budget-based rate structure, including indoor and outdoor budgets (indoor budget from household size at 55 gallons per person per day, outdoor budget from ET and irrigable area), so that the paper's pre-removal efficiency variable, the mean of log((use + 1) / (budget + 1)) over pre-period months, can be computed exactly. Write the rate structure in an OWRS-compatible form if practical, so RateParser can reproduce the bills. Do not copy any real utility's rates.

**Occupant turnover, new construction and closures.** Treat these as three distinct processes and never conflate them with missing data. Occupant turnover means a household moves out and another moves in at the same physical property. It occurs at an assumed rate of about 7 percent per year for single family homes. On turnover, keep `location_id_from_utility` and the property attributes (including irrigable area and any removed turf) fixed, assign a new `cust_id_from_utility` and a new `cust_loc_id`, insert a vacancy gap of zero to three months with near-zero usage but present billing rows, and redraw household size and the latent attitude `a_i` (partially, to keep some correlation with the block group). New construction means a new property appears mid-panel with a new `location_id_from_utility`, at an assumed rate near 1 percent of accounts per year, concentrated in the So Cal Suburb and Inland Enclave retailers. Closure means a property demolished or merged, at a very low rate, after which its rows stop. Store the event type, date and affected IDs in the truth folder. Define pre-rebate filters and the pre-removal efficiency variable at the location level, keep a flag `turnover_in_pre_period` and a flag `turnover_in_post_period` on treated locations, and report in the validation suite how turnover affects the analysis cohort and the efficiency measure.

**Observation layer.** Apply the data quality features of real utility data as a separate step after generating the complete, clean panel, as in chapter 25 of Gelman and Hill on missing data. Keep the clean panel in the truth folder. The observation layer should include bimonthly billing for the two bimonthly retailers (with billing period start and end dates), missing billing rows that represent gaps in the data feed while the account stays active (these are distinct from vacancy rows, which are real and near zero), missing household size or irrigable area for some retailers, default irrigable area values that cause bunching at one value for one or two retailers (the paper drops such accounts), duplicate rebates on one account (the paper drops these), and a small share of extreme or erroneous values.

**Rebate and supplier tables.** Write the rebate table with the catalogue's fields, including turf, high efficiency clothes washer and weather based irrigation controller types, with received date before post-inspection date before check date. Write the supplier_report table by aggregating account-level usage, scaling the sample up to a plausible service population for each retailer, and computing gallons per capita day using the catalogue's formula (current production times percent residential, divided by days in month times population served). Report a percent reduction relative to 2013 and drought-era complaints, warnings and penalties as Poisson counts that rise after mid-2014. Make the account-level and supplier-level tables consistent so that totals reconcile within a stated tolerance.

## 5. Build order with validation gates

Work in the following order and do not move to the next stage until the gate for the current stage passes. Start with a simple model, check it, and then add complexity, which is the "start simple and meet in the middle" advice from the Bayesian Workflow paper.

1. Set up the repository, the environment lockfile, the config file with source tags, and the RNG scheme in section 6.
2. Generate geography, block group covariates and accounts. Gate: plot the marginal and joint distributions by retailer and confirm they fall inside the bounds tables.
3. Generate ET, precipitation and the untreated usage panel with no treatment and no observation noise. Gate: run a prior predictive check by drawing the parameters from their priors many times and confirming that the implied usage is plausible (single family usage mostly between about 0 and 100 CCF per month, annual use mostly between roughly 100 and 600 CCF, clear summer peaks that are stronger inland than on the coast, a visible 2015 drop).
4. Add selection, treatment and turnover with the true effects. Gate: confirm selection depends on covariates and the latent attitude, confirm the savings distribution matches the Figure 3 and Figure 4 targets, and confirm the physical feasibility constraints hold.
5. Add billing, the rate structure and the rebate and supplier tables. Gate: reproduce the bills and the reconciliation totals.
6. Add the observation layer. Gate: confirm the filters leave about 150 to 200 treated households and at least 100 control candidates per zip code.
7. Run the full validation suite in the next section and write the report.

## 6. Reproducibility and environment

**Tiers of reproducibility.** Data generation must be byte-for-byte reproducible on any machine that uses the same lockfile. Model refits in Stan are reproducible only to numerical tolerance, because compiler versions and floating-point behavior can differ across machines, so the validation report should state tolerances and not promise identical draws. Keep the generation scripts free of Stan and of any step that depends on multithreaded floating-point reduction order.

**Parallel random number streams.** Derive one independent RNG stream per unit of work, and key each stream to the unit's ID (retailer, zip code, block group or account) and not to the worker or core that happens to run it, so that the result does not change with the number of cores. In R, set `RNGkind("L'Ecuyer-CMRG")`, compute a list of streams with `parallel::nextRNGStream()` indexed by unit, and assign each unit its stream before running it, or use `future.apply` with `future.seed = TRUE`; when using a cluster, `parallel::clusterSetRNGStream()` seeds worker streams, but it does not by itself guarantee unit-keyed reproducibility, so prefer the explicit per-unit assignment. In Python, create `numpy.random.SeedSequence(seed)`, spawn one child per unit with a deterministic ordering, and build a `numpy.random.Generator` from each child. Log the root seed and the unit-to-stream mapping. Never draw random numbers inside code whose order depends on scheduling.

**Deterministic outputs.** Sort every table by its keys before writing. Write Parquet with a fixed compression codec and no embedded timestamps or writer-version metadata that varies. Pin BLAS and OpenMP threads to one for the generation scripts. Add a test that runs the generator twice, with different core counts, and compares file hashes.

**Dependency locking.** Mandate a lockfile: `renv.lock` for R (created with `renv::snapshot()`), or `uv.lock` or `poetry.lock` for Python. Pin the CmdStan version, record the C++ compiler and version, and write `reports/environment.txt` containing the R or Python version, `sessionInfo()` or `pip freeze`, the CmdStan version and the compiler. Provide a `Dockerfile` built on a pinned base image tag (for example a versioned `rocker/r-ver` image) so a clean machine can reproduce the environment, and note that the container is optional for developers who already have a matching toolchain.

## 7. Compute and memory budget

The default run has 3,000 accounts over 72 months, which is about 216,000 account-months. That size is easy to generate and store on a standard workstation, so generate and write all tables, the clean panel and the potential outcomes for all accounts. The expensive steps are model fitting, so apply these limits. Fit the multilevel Stan or brms recovery check on a random subsample of 500 accounts, stratified by retailer, and keep the full 3,000-account panel for descriptive comparisons and the replicated-data graphs. Run simulation-based calibration on the small version of the model (about 300 accounts and 36 months). Run the M123 recovery test on all treated households (about 150 to 200) against matched controls, but parallelize it across treated households using the scheme in the paper's appendix, and write intermediate results to disk so a crash does not lose progress. Provide three config profiles: `tiny` (500 accounts, 100 per retailer, for development), `default` (3,000 accounts across the five retailers) and `large` (12,000 accounts, future use, with the subsampling rules above applied automatically). Process the panel in chunks by retailer, avoid loading the full potential-outcomes table into memory at once, and print the peak memory use in the report.

## 8. Validation suite (Gelman and Hill practices)

Write each check as a script that saves a figure and a pass/fail line to the report.

**Fake-data recovery.** Refit the generating model's multilevel structure to the simulated data with Stan or brms (on the subsample from section 7), and confirm that the true hyperparameters (retailer-level and block-group-level variances, the AR(1) coefficient, the drought effect, the treatment regression coefficients) fall inside their posterior intervals at close to the nominal rate. This is the fake-data simulation approach of chapter 8 of Gelman and Hill.

**Simulation-based calibration.** On the small model, repeat the simulate-and-refit cycle at least 100 times, draw parameters from the prior each time, and plot rank histograms for the key parameters. Flag any parameter whose ranks are visibly non-uniform, since that indicates a mismatch between the generator and the fitting code.

**Replicated-data comparison.** Compare the synthetic data to the documented targets using graphs rather than only summary statistics: spaghetti plots of usage by retailer, seasonal profiles that show the coastal versus inland contrast, the distribution of annual savings by year, the distribution of matched-versus-unmatched covariate distances, and residual plots. Treat these as posterior predictive style checks and describe every visible discrepancy in the report.

**Causal recovery.** Run a simplified version of M123 (matching on usage correlation within zip code, CausalImpact or a BSTS fit per treated account, and a weighted mixed-effects meta-regression with `metafor`) on the synthetic data. Compare the estimated savings to the true household-level savings in the truth folder, and report bias, interval coverage, and rank correlation. Because the true potential outcomes are stored, also report how a naive difference-in-differences or mixed model with a post-rebate dummy compares, to illustrate the selection bias the paper discusses, and report results separately for households with and without turnover in the pre-period. This stage is the most expensive, so make it optional behind a flag.

**Edge cases.** Test accounts with short histories, rebates near the panel ends, retailers with bimonthly billing, retailers with defaulted irrigable area, vacancy gaps versus missing rows, and turnover near a rebate date, and confirm the observation layer produces them at the configured rates.

## 9. Output layout

Write the following under `synthetic_scuba/`, using Parquet for large tables and CSV for small ones, with a data dictionary that mirrors the catalogue field names.

- `config/params.yaml` and `config/profiles/`: all parameters with `source` and `rationale` fields, and the `tiny`, `default` and `large` profiles.
- `R/01_geography.R` through `R/08_observation.R`, and `R/09_validate.R`: numbered scripts that can each be run alone given the previous outputs.
- `data/usage.parquet`, `data/customers.csv`, `data/rebates.csv`, `data/acs_block_groups.csv`, `data/supplier_report.csv`, `data/climate_et_precip.csv`, `data/rates_owrs/`: the observed tables using catalogue field names.
- `truth/`: the clean panel, the potential outcomes for every account-month, the true household savings, the latent attitude, the turnover, construction and closure events, and a JSON file of all true parameters.
- `reports/validation.html` and `reports/environment.txt`: the validation report with all figures and pass/fail lines, and the environment record.
- `renv.lock` (or the Python lockfile), `Dockerfile`, and `README.md`: the lockfile, the container recipe, the synthetic data banner, the written data-generating process, the bounds tables with sources, and a limitations section.

## 10. Judgment calls to log and surface

Record each of the following in the README so the user can review them. Parameters tagged `assumed` are not supported by the sources and should be tuned by someone with real data experience, including every demographic and climate value in section 3. The paper defines pre-removal efficiency with a log ratio but describes values "below 1" as conservers, so use the log definition and treat negative values as conservers. The Table 3 intercept of about -17.7 is a model intercept rather than a standalone average and must not be used as the mean savings. The catalogue reports that the number of utilities and customer counts change over time, so keep retailer count and account counts configurable instead of hard-coded. The 3,000-account build yields fewer treated households than the paper's 545, so confidence intervals from the recovery tests will be wider than in the paper. If any required quantity is ambiguous, choose a reasonable default, record it in the config, and continue instead of stopping to ask.

## 11. Definition of done

The work is done when the default 3,000-account run completes from a clean checkout with one command, every validation gate passes or has a documented explanation, the truth folder reconciles with the observed tables, the README documents the data-generating process and limitations, and a second run with the same seed, on a different core count, reproduces the same data files with identical hashes.

## References

- Schmitt, E., Tull, C. and Atwater, P. (2018). Extending Bayesian structural time-series estimates of causal impact to many-household conservation initiatives. Annals of Applied Statistics 12(4), 2517-2539. https://projecteuclid.org/journalArticle/Download?urlId=10.1214%2F18-AOAS1166
- California Data Collaborative, SCUBA Metadata Catalogue. https://github.com/California-Data-Collaborative/metadata_catalogue
- Gelman, A. and Hill, J. (2007). Data Analysis Using Regression and Multilevel/Hierarchical Models. Cambridge University Press. Chapters 7 and 8 (simulation and fake-data checking), 12 and 13 (multilevel models), 24 and 25 (model checking and missing data).
- Gelman, A., Hill, J. and Vehtari, A. (2020). Regression and Other Stories. Cambridge University Press.
- Gelman, A., Vehtari, A., Simpson, D., et al. (2020). Bayesian workflow. arXiv:2011.01808.
- Talts, S., Betancourt, M., Simpson, D., Vehtari, A. and Gelman, A. (2020). Validating Bayesian inference algorithms with simulation-based calibration.
- Hill, J. L. (2011). Bayesian nonparametric modeling for causal inference. Journal of Computational and Graphical Statistics 20, 217-240.
