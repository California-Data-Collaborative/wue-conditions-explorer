# WUE Conditions Explorer Dashboard: Instructions for Claude Code (Stub)

> **Status: stub.** This file is a skeleton that we will fill in interactively once the synthetic SCUBA data exists. Each section states what is already settled and lists the open questions marked **To decide**. Claude Code should not build the dashboard from this file until every **To decide** item has been resolved or explicitly deferred.

## 1. Goal and audience

The dashboard lets a user explore water use efficiency conditions across the five fictional retailers in the synthetic SCUBA dataset, and it shows how climate, demographics, rates and turf rebates relate to residential water use. It is built entirely on the synthetic data produced by `claude_code_instructions_synthetic_scuba.md`, so it can be shared and demonstrated without any privacy review.

**To decide.** We need to name the primary audience, which could be utility conservation staff, Metropolitan Water District program managers, analysts testing the M123 pipeline, or a general public audience. We also need to state the two or three questions the dashboard must answer in under a minute, since those questions will drive the layout.

## 2. Data inputs

The dashboard reads the observed tables under `synthetic_scuba/data/`, which are `usage.parquet`, `customers.csv`, `rebates.csv`, `acs_block_groups.csv`, `supplier_report.csv`, `climate_et_precip.csv` and the rate structures in `rates_owrs/`. It never edits those files. Any aggregation it needs is computed by a separate preparation step described in section 5.

The `synthetic_scuba/truth/` folder holds the answer key, which includes true savings, latent attitudes and potential outcomes. A realistic dashboard would not have access to these values.

**To decide.** We need to decide whether the dashboard ignores the truth folder entirely, or whether it offers a clearly labeled "answer key" mode that compares estimated savings with true savings for teaching and validation purposes.

## 3. Guardrails

Every page carries a visible banner stating that all data is simulated and must not be mistaken for real utility records. The dashboard uses only the five fictional retailer names, and it shows no real utility names, addresses or parcel numbers. Individual account rows are never displayed alone, even though they are synthetic, because the dashboard should model the privacy habits that a real-data version would need.

**To decide.** We need to set a minimum group size for any aggregate shown (for example, no cell built from fewer than 10 accounts), and to decide whether this rule applies to the synthetic build or only to a future real-data version.

## 4. Candidate views

The list below is a starting menu rather than a commitment. We will choose, cut and order these views once we have looked at the data.

- A retailer overview shows accounts, single family share, median irrigable area, median income and billing frequency for each retailer.
- A climate panel shows monthly ET0 and precipitation by retailer, and it highlights the drought years from 2012 to 2015.
- A usage panel shows seasonal usage profiles by retailer and customer class, and it makes the coastal-versus-inland contrast and the 2015 drought drop visible.
- An efficiency panel shows usage against the indoor and outdoor water budget, using the pre-removal efficiency measure from the paper.
- A supplier panel shows gallons per capita per day, percent reduction relative to 2013, and enforcement counts over time.
- A rebate panel shows uptake over time and across geography, along with turf square footage removed and rebate dollars paid.
- A savings panel shows treated households against matched controls, and, if answer key mode is adopted, it compares estimated with true savings.
- A data quality panel shows missing rows, vacancy gaps, defaulted irrigable area and turnover, so users can see what the observation layer did.

**To decide.** We need to choose the views for the first release, the default filters (retailer, zip code, customer class and date range), and whether the geography uses a synthetic map of the fictional block group grid or stays non-spatial.

## 5. Data preparation layer

A preparation script reads the observed tables and writes small, pre-aggregated files that the dashboard loads directly. Keeping this step separate keeps the dashboard fast and makes every displayed number traceable to a script. The preparation step uses the same language and lockfile as the generator.

**To decide.** We need to settle the aggregation grain (retailer-month, zip-month or block-group-month) and the file format for the prepared data.

## 6. Technology

**To decide.** We need to choose the stack. The main candidates are listed below, and we will pick one after the data exists and the audience is settled.

- A static HTML page with embedded or bundled data is the easiest to share and host, and it needs no server.
- R Shiny or Quarto dashboards fit the R generator and the CaDC tool ecosystem, but Shiny needs a running server.
- Streamlit or Dash fit a Python fallback, and they also need a running server.
- Observable Framework produces a static site with strong charting, at the cost of a JavaScript build step.

## 7. Design

Charts follow accessible color practice, with colorblind-safe palettes, direct labels where possible, and a layout that works on a laptop and a phone. Units are stated on every axis, using CCF for account usage, gallons per capita per day for supplier figures, inches for ET and precipitation, and square feet for area.

**To decide.** We need to decide whether to apply Metropolitan Water District brand styling, California Data Collaborative styling, or a neutral style.

## 8. Validation

Every number on the dashboard must reconcile with the prepared data, and the prepared data must reconcile with the observed tables. A test script recomputes a handful of headline figures directly from the observed tables and compares them with what the dashboard displays.

**To decide.** We need to decide whether to add automated screenshot tests with Playwright, which is available in the cloud environment.

## 9. Output layout

**To decide.** A tentative layout places the preparation script and its outputs under `dashboard/prep/`, the application under `dashboard/app/`, and the tests under `dashboard/tests/`, with a `dashboard/README.md` that repeats the synthetic data banner. We will confirm this once the stack is chosen.

## 10. Open questions log

We will record decisions here as we make them, with the date and a one-sentence rationale for each.

| Date | Decision | Rationale |
|---|---|---|
| | | |

## 11. Definition of done

**To decide.** A draft definition is that the dashboard builds from a clean checkout with one command, reads only the prepared data, shows the synthetic data banner on every page, passes the reconciliation tests, and answers the questions named in section 1.
