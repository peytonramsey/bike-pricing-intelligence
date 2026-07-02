# Bike Pricing Intelligence: Hedonic Pricing & Causal Analysis

A self-built data pipeline and econometric analysis of road and gravel bike pricing across Cannondale, Giant, and Trek, from live scraping through hedonic regression to a difference-in-differences causal test.

**Stack:** Python, requests, BeautifulSoup, Playwright, pandas, statsmodels, Wayback Machine CDX API

**Headline result:** OLS log-price regression, adj-R2 = 0.824 (n = 755). TWFE difference-in-differences test on electronic drivetrain adoption, n = 100 series-year observations, adj-R2 = 0.869.

---

## Overview

Bike manufacturers publish MSRP with no systematic explanation of what drives price differences across models. This project builds a full pipeline, live scrapers, historical backfill via the Wayback Machine, spec normalization, and two rounds of econometric modeling, to answer two questions:

1. How much of a bike's price is explained by groupset, frame material, and platform positioning versus brand premium (hedonic regression)?
2. Does introducing electronic drivetrain variants causally raise a model series average price, or does it simply add a price point within an existing tier structure (difference-in-differences)?

No public dataset captures cross-brand, multi-year MSRP alongside structured spec attributes, so the data pipeline was a prerequisite for the analysis itself.

---

## Data

Two scrapers collect road and gravel bikes from Giant and Cannondale using requests and BeautifulSoup for standard pages, with Playwright reserved for JavaScript-rendered product pages. Historical pricing for Cannondale and Trek was backfilled through the Wayback Machine CDX API, querying the Internet Archive for all product page snapshots and parsing the relevant HTML with no manual data entry.

| Dataset | Rows | Giant | Cannondale | Years |
|---|---|---|---|---|
| Full (V1) | 1,436 | 255 | 1,181 | 2017-2026 |
| Groupset-known (V2/V3) | 755 | 219 | 536 | 2017-2026 |
| Trek (DiD control) | 3,039 priced records | - | - | 2017-2026 |

Raw groupset strings were normalized into a `groupset_tier` categorical variable and a `groupset_rank` ordinal (0-10). Frame material was collapsed into `frame_tier` (carbon, carbon_hi_mod, aluminum). Model platform was extracted into a `model_series` variable (SuperSix EVO, Synapse, TCR Advanced, and others) via keyword lookup.

---

## Hedonic Pricing Regression

Three OLS models were estimated with HC3 heteroskedasticity-robust standard errors, using log(price) as the dependent variable.

| Model | n | R2 | Adj-R2 | Features |
|---|---|---|---|---|
| V1 - Full sample | 1,436 | 0.429 | 0.426 | groupset_rank + unknown_flag + frame_tier + category + brand |
| V2 - Complete-case baseline | 755 | 0.701 | 0.699 | Same, groupset-known only |
| V3 - Enhanced | 755 | 0.832 | 0.824 | groupset_tier (categorical) + model_series + scrape_year |

### Key findings

- **SRAM commands a large premium over Shimano at equivalent rank tiers.** The gap ranges from +101 percent at low tiers to roughly parity at the top, motivating a shift from a scalar rank to categorical groupset dummies.
- **Model platform is the primary price driver, not specs.** Adding model_series contributed more to R2 than any other change (+12.5 pp), with platforms like SystemSix and SuperSix EVO carrying premiums over 100 percent versus the Synapse baseline.
- **The apparent gravel discount in the full sample was a data artifact.** It disappears once groupset-unknown bikes are excluded, confirming no genuine road/gravel price differential at equivalent specs.
- **Electronic drivetrain premiums are absorbed by platform**, not priced independently, once model_series is controlled for.
- **List prices declined roughly 4 percent per year** after controlling for spec, consistent with brands improving spec-for-price over time rather than raising MSRP with inflation.

Full model diagnostics, VIF checks, and coefficient tables are in `WRITEUP.md`.

---

## Causal Inference: Difference-in-Differences

The hedonic model explains price variation but cannot answer whether introducing electronic drivetrain variants causally shifts a model series average price. A two-way fixed effects (TWFE) DiD design was used to test this directly.

An initial Cannondale-only design failed: the Wayback Machine backfill for Cannondale only extends to May 2020, leaving zero pre-treatment years for every series that adopted electronic drivetrains in 2020. The resulting estimate had a 95 percent confidence interval spanning -100 percent to +2,000,000 percent, an uninterpretable result caused by insufficient identifying variation.

Trek was added as a second brand with archived pages dating to 2016 and staggered electronic adoption across its model lines (2017-2022), providing the cross-series and pre-treatment variation the design required.

**Full TWFE result (n = 100 series-year observations, adj-R2 = 0.869):**

| Variable | Coef | p-value |
|---|---|---|
| treated_post (DiD estimate) | -0.019 | 0.923 |
| tariff_pct | -0.009 | 0.780 |
| aluminum_ppi | +0.012 | 0.458 |
| carbon_ppi | -0.015 | 0.174 |
| bikes_ppi | -0.030 | 0.201 |
| freight_ppi | +0.037 | 0.586 |

Pre-treatment event study coefficients were small and non-significant (p > 0.16), supporting the parallel trends assumption.

**Result:** Introducing electronic drivetrain variants is associated with a -1.9 percent change in mean series MSRP, statistically indistinguishable from zero. Brands do not use electronic launches to reprice an entire model line; they add a new price point within an existing platform tier structure. This directly confirms the hedonic regression finding from a different angle: the electronic drivetrain effect collapses to near zero once model_series is controlled for, because drivetrain and price tier are co-determined by platform architecture.

---

## Predictive Validation

The V3 model was refit on an 80 percent training split and evaluated on a 20 percent holdout, stratified by brand.

| Metric | In-sample | Out-of-sample |
|---|---|---|
| R2 / Adj-R2 | 0.824 | 0.825 |
| RMSE (log points) | - | 0.341 |
| MAE (log points) | - | 0.264 |

The near-identical in-sample and out-of-sample fit confirms the model_series dummies capture real structural price drivers rather than overfitting. The remaining error reflects genuine within-platform trim variation (a single platform can span multiple groupset configurations at different price points), not model misspecification.

![Validation plots](analysis/validation_plots.png)

---

## Caveats

- **Giant and Cannondale temporal asymmetry:** Cannondale provides a nine-year panel; Giant is close to a single-year cross-section. Brand-level coefficients should be read as directional, not precise.
- **MSRP, not transaction prices:** all prices are manufacturer list prices at time of scrape. Clearance events, dealer discounts, and regional pricing are not captured.
- **Small-n model series:** a few platforms (CAADX, Propel Advanced) have too few observations for precise coefficient estimates; direction is informative, magnitude is not.

---

## Repository Structure

```
.
├── scrapers/          # Live scrapers per brand (Giant, Cannondale, Trek)
├── parsers/           # HTML parsing logic per brand
├── backfill/           # Wayback Machine CDX API backfill scripts
├── pipeline/           # Data cleaning, spec normalization, model-ready table builder
├── analysis/           # Regression scripts, DiD analysis, validation plots
├── data/               # Raw and cleaned CSV/parquet snapshots
└── WRITEUP.md          # Full methodology, diagnostics, and findings
```

---

## Tech Stack

| Layer | Tools |
|---|---|
| Scraping | Python, requests, BeautifulSoup, Playwright |
| Historical backfill | Wayback Machine CDX API, custom HTML parser |
| Data pipeline | pandas, parquet snapshots, modular pipeline |
| Regression | statsmodels OLS, HC3 robust standard errors, VIF diagnostics |
| Causal inference | Two-way fixed effects DiD, event study design |
| Output | CSV and parquet snapshots, regression outputs, written report |

For full methodology, diagnostics, and detailed findings, see [WRITEUP.md](WRITEUP.md).
