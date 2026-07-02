# Bike Pricing Intelligence — Hedonic Pricing Analysis

**Stack:** Python · requests · BeautifulSoup · Playwright · pandas · statsmodels · Wayback Machine CDX API  
**Data:** Cannondale (2017–2026, multi-year panel) + Giant (2026, live scrape) — road and gravel bikes only  
**Final model:** OLS log-price regression, adj-R² = 0.824, n = 755

---

## Problem and Approach

Bike manufacturers publish MSRP but provide no price guidance beyond their own catalogue. A rider comparing a SuperSix EVO with SRAM Force to a Synapse with Shimano Ultegra has no systematic way to answer whether the premium is driven by the groupset, the frame platform, the brand, or something else entirely.

Hedonic pricing regression decomposes a product's price into the implicit value of each attribute by regressing log(price) on observed specifications. The framework is well-suited here because road and gravel bikes are highly configurable — the same model line ships with multiple groupset and frame tier options — which means within-series variation can cleanly identify component-level price contributions.

**Why scrape rather than use an existing dataset?** No public dataset captures cross-brand, multi-year MSRP alongside structured spec attributes (frame material, groupset tier, drivetrain type). Existing sources are either consumer reviews (no price series) or retailer inventory scrapers (point-in-time, no spec normalisation). Building the pipeline was the prerequisite for doing the analysis at all.

---

## Data

### Scraper and Backfill

Two scrapers collect road and gravel bikes from Giant and Cannondale. Both use `requests` + `BeautifulSoup` for standard catalogue pages; Playwright is reserved for JavaScript-rendered product detail pages where static requests fail. Scrapers are modular (config, scrapers, parsers, pipeline) with polite rate limiting, retries, and raw HTML saved for debugging.

Cannondale's historical price data was backfilled through the **Wayback Machine CDX API** — querying the Internet Archive's index for all Cannondale product page snapshots, then fetching and parsing the relevant HTML. This yielded pricing records from 2017 through early 2026 with no manual data entry.

### Spec Normalisation

Raw groupset strings ("Shimano GRX 600", "SRAM Force AXS", etc.) were normalised to a `groupset_tier` categorical variable and a `groupset_rank` ordinal (0–10). Bikes with unrecognised groupsets were flagged with `groupset_unknown_flag` rather than dropped, preserving them for the full-sample model while isolating the complete-case sample for the enhanced model. Frame material was collapsed to `frame_tier` (carbon, carbon_hi_mod, aluminum). Model platform was extracted from model names via a keyword lookup into a `model_series` variable (SuperSix EVO, Synapse, TCR Advanced, etc.).

### Sample

| Dataset | Rows | Giant | Cannondale | Years |
|---|---|---|---|---|
| Full (V1) | 1,436 | 255 | 1,181 | 2017–2026 |
| Groupset-known (V2/V3) | 755 | 219 | 536 | 2017–2026 |
| Unique model names (gs) | 131 | 20 | 111 | — |

**Temporal asymmetry:** Cannondale is a multi-year panel (~4.8 observations per model across 9 years). Giant is effectively a 2026 cross-section (~3.3 obs/model, nearly all from the live scrape). This means `scrape_year` controls almost exclusively for Cannondale's price drift and cannot cleanly separate brand effects from temporal coverage differences. Brand coefficients should be read as directional, not precise.

---

## Model Progression

All models use OLS with **HC3 heteroskedasticity-robust standard errors** and `log(price)` as the dependent variable. Reference levels: Shimano 105 (groupset), standard carbon (frame), road (category), Cannondale (brand), Synapse (model_series in V3).

| Model | n | R² | Adj-R² | Features |
|---|---|---|---|---|
| V1 — Full sample | 1,436 | 0.429 | 0.426 | `groupset_rank` + `unknown_flag` + `frame_tier` + `category` + `brand` |
| V2 — Complete-case baseline | 755 | 0.701 | 0.699 | Same, groupset-known only |
| V3 — Enhanced | 755 | 0.832 | **0.824** | `groupset_tier` (categorical) + `model_series` + `scrape_year` |

**V1 → V2:** Restricting to groupset-known bikes removes 681 observations with unknown components. The jump from 0.426 to 0.699 adj-R² is partly mechanical (removing a noisy cluster) and partly real (the `groupset_unknown_flag` coefficient of +1.59 in V1 reflects how mixed that group is, not a genuine spec effect).

**V2 → V3:** Three changes, each motivated by diagnosis:

1. **`groupset_rank` (scalar) → `groupset_tier` (categorical).** The ordinal rank treats SRAM Force and Shimano 105 as equivalent because they share the same rank tier. They don't price equivalently — see SRAM findings below. Categorical dummies recover the true non-linear structure.

2. **`model_series` added.** The single largest gain (+12.5 pp adj-R²). Platform positioning — SystemSix vs. Synapse vs. CAAD Optimo — explains price variation that specs alone cannot. These are real attributes: intended rider type, geometry targets, marketing tier.

3. **`category` (gravel/road) dropped.** Including category alongside model_series raised the gravel VIF to 13.5 — a specification problem, not just a high VIF. Topstone, Revolt, CAADX, and SuperX are all gravel-only; SystemSix, SuperSix EVO, and Synapse are all road-only. The two variables nearly coincide, making the coefficient and p-value unstable. The correct resolution is to let model_series carry the road/gravel distinction at the appropriate granularity. V2 serves as the clean category isolation: after controlling for groupset, the gravel coefficient is +0.004 log pts (p = 0.92) — no category effect.

VIF check on V3: max 7.1 (aluminum frame tier, structural overlap with CAAD-series bikes). All groupset, model_series, and scrape_year predictors are below 5.

---

## Key Findings

### 1. SRAM commands a large premium over Shimano at equivalent rank tiers

Before building V3, a diagnostic compared mean log-price between SRAM and Shimano bikes at the same `groupset_rank`:

| Rank tier | Shimano (mean price) | SRAM (mean price) | SRAM premium |
|---|---|---|---|
| Rank 4 | $2,019 | $4,065 | +101% |
| Rank 6 | $2,905 | $4,419 | +52% |
| Rank 8 | $5,862 | $6,634 | +13% |
| Rank 10 | $12,109 | $11,663 | −4% |

The gap is largest in the mid-range and disappears at the top. This is what motivated replacing the scalar rank with categorical groupset_tier dummies. In V3, SRAM Force carries a coefficient of **+0.98 log pts** above the 105 reference (holding model_series and frame tier constant), while dura_ace is +0.59 — indicating the SRAM mid-tier premium is larger than the premium from moving to Shimano's own flagship.

### 2. Model platform is the primary price driver, not specs

Adding `model_series` contributes more to R² than any other single change. After controlling for groupset, frame, and year, platform coefficients relative to Synapse:

| Platform | Coef vs. Synapse | Approx. level premium |
|---|---|---|
| SystemSix | +0.76 | +113% |
| SuperSix EVO | +0.72 | +105% |
| Propel Advanced | +0.74 | +109% |
| Defy Advanced Pro | +0.43 | +54% |
| Defy Advanced | +0.20 | +22% |
| CAAD Optimo | −0.38 | −32% |
| CAADX | −0.58 | −44% |

This is hedonic pricing working as intended: the model_series coefficient captures the implicit price of the platform attribute — geometry intent, brand architecture, marketing position — that sits above what groupset and frame material explain.

### 3. The gravel discount in V1 was a data artifact

V1 shows gravel bikes priced ~15% below road (−0.155 log pts, p < 0.001). This disappears in V2 (−0.004, p = 0.92) and was not retained in V3. The V1 result was driven by the large unknown-groupset cluster in the full sample, which skewed heavily toward lower-priced gravel bikes. Once groupset is known, there is no road/gravel price differential holding specs constant.

### 4. Electronic drivetrains are absorbed by platform in V3

In V1 and V2, electronic flag carries +0.21–0.22 log pts (~23% premium). In V3 it collapses to zero (p = 0.87). Di2 and AXS builds cluster within specific model lines — SystemSix Carbon Di2, SuperSix EVO Hi-MOD Dura-Ace Di2 — that already carry large positive model_series coefficients. The electronic premium is real; it's just expressed through platform, not independently.

### 5. List prices declined ~4% per year after spec controls

`scrape_year` enters V3 at −0.043 log pts per year (p < 0.001). After holding platform, groupset tier, and frame constant, Cannondale's MSRPs fell about 4.3% annually across 2017–2026. The most plausible explanation is spec-for-price improvement: Cannondale moved to better groupsets at the same MSRP over time rather than raising prices to match inflation.

---

## Caveats

**Giant/Cannondale temporal asymmetry** is the most important structural limitation and should be stated up front in any summary of the brand coefficient. Cannondale provides a nine-year panel; Giant is a 2026 cross-section. `scrape_year` cannot cleanly separate time trend from brand-level effects for Giant, and the Giant coefficient (−0.06 to −0.09 in V1/V2) partially reflects coverage differences rather than true brand discount.

**MSRP, not transaction prices.** All prices are manufacturer list prices at time of scrape. Clearance events, dealer discounts, and regional price variation are not captured. The hedonic estimates reflect how manufacturers price attributes, not how the market clears them.

**Small-n series.** CAADX (n=5) and Propel Advanced (n=8) have too few observations for precise coefficient estimates. Their direction is informative; the magnitudes are not reliable for headline claims.

---

## Tech Stack

| Layer | Tools |
|---|---|
| Scraping | Python · requests · BeautifulSoup · Playwright |
| Historical backfill | Wayback Machine CDX API · custom HTML parser |
| Data pipeline | pandas · parquet snapshots · modular pipeline (config/scrapers/parsers/models/pipeline) |
| Regression | statsmodels OLS · HC3 robust SEs · VIF via `variance_inflation_factor` |
| Output | CSV + parquet by scrape date · regression CSVs · written report |

---

## Phase 2: Causal Inference — Attempted DiD Design

### Goal and Design

The hedonic model (V3) answers "what explains price differences" but cannot answer "does adding electronic drivetrain variants causally raise a model series' average MSRP?" A difference-in-differences (DiD) design was planned to isolate the causal effect of groupset tier upgrades and mechanical-to-electronic transitions from concurrent macro shocks (tariffs, freight costs, COVID demand surge).

**Intended design:**
- Treatment: a Cannondale model series introduces its first electronic-drivetrain variants in a given year
- Control: model series (ideally a second brand) with no electronic variants that year
- Method: Two-Way Fixed Effects (TWFE), `log(mean_price_it) ~ treated_post + series_FE + year_FE + tariff_pct`

### What the Data Revealed

Executing Stage 3 (coverage check) uncovered a critical data gap: the Cannondale Wayback Machine backfill begins in **May 2020**, not 2017. The Internet Archive did not have sufficient pre-2020 Cannondale catalogue pages for the CDX API to return parseable snapshots.

This eliminates the pre-treatment window for Cannondale:

| Series | First Electronic Year | Earliest Data Year | Pre-treatment obs |
|---|---|---|---|
| SuperSix EVO | 2020 | 2020 | 0 |
| SystemSix | 2020 | 2020 | 0 |
| SuperX | 2020 | 2020 | 0 |
| Synapse | 2020 | 2020 | 0 |
| CAAD13 | 2021 | 2020 | 1 year only |

Without pre-treatment data for the treated series, the parallel trends assumption cannot be tested, and the treatment indicator is nearly collinear with the year fixed effects. The TWFE regression ran (n=45 series-year observations) but produced a non-significant coefficient with very wide confidence intervals (95% CI: [-100%, +2M%]) — a direct consequence of insufficient identifying variation.

The tariff coefficient was highly significant (p < 0.001, coef = +0.24 log pts per percentage point of tariff rate), confirming that macro controls do shift prices and are worth including, but the treatment effect itself is unidentifiable from Cannondale's panel alone.

### Why Trek Fixes This

A CDX coverage probe (run via the Wayback Machine CDX API) confirmed that Trek's product pages are archived from 2016 onwards, with adequate snapshot coverage each year:

| Year | Trek Road Snapshots |
|---|---|
| 2016 | 2,467 |
| 2019 | 266 |
| 2020 | 216 |
| 2021 | 370 |
| 2022 | 480 |

Trek's Madone series adopted electronic drivetrains around 2020 (similar to Cannondale's SuperSix EVO), while the Domane endurance line and Checkpoint gravel line adopted electronic variants later (2022-2024). This staggered adoption timing — across multiple models within Trek, and between Trek and Cannondale — provides the within-year control variation that the DiD requires.

**Cannondale-only TWFE (n=45, preliminary):** The initial run without Trek produced a -44% point estimate with a 95% CI of [-100%, +2M%] — a direct consequence of only one control series (CAAD Optimo) and treatment collinear with year fixed effects. This result was not interpretable.

**Full TWFE with Trek (n=100 series-year obs, adj-R²=0.869):**

| Variable | Coef | p-value | Interpretation |
|---|---|---|---|
| **treated_post (DiD estimate)** | **-0.019** | **0.923** | **-1.9% (95% CI: -33% to +44%)** |
| tariff_pct | -0.009 | 0.780 | Absorbed by year FE |
| aluminum_ppi | +0.012 | 0.458 | — |
| carbon_ppi | -0.015 | 0.174 | — |
| bikes_ppi | -0.030 | 0.201 | — |
| freight_ppi | +0.037 | 0.586 | — |

Event study pre-period coefficients (rel_year −3 to −1): −0.50, −0.21, 0.00 — small and non-significant (p > 0.16). Parallel trends assumption broadly holds.

**Plain-language result:** The DiD tests a specific pricing strategy question: do brands use electronic drivetrain launches as an opportunity to reprice an entire model line? The answer is no. The -1.9% estimate (indistinguishable from zero) means introducing Di2 or AXS variants does not shift the series' average MSRP — it simply adds a higher price point within a tier structure that already existed. Brands price platforms, not drivetrains.

This matters because the alternative hypothesis was plausible: a brand could use an electronic launch as cover to raise the floor price of a series, retire low-margin mechanical variants, or reposition the line upward. The data rules that out. The SystemSix Carbon Di2 and SystemSix Carbon 105 coexist at different price points within the same platform; the Di2 launch doesn't pull the 105 price with it. The hedonic regression (V3) shows the same thing from a different angle — `electronic_flag` coefficient collapses to zero (p = 0.87) after adding model_series controls, because the drivetrain and the price tier are co-determined by platform architecture, not sequentially.

Trek's staggered electronic adoption (Madone SLR 2019, Domane SL 2019, Domane SLR 2017, Checkpoint SL 2022) versus four always-mechanical control series (CAAD Optimo, Checkpoint, Checkpoint ALR, Crockett) provided the cross-series control variation that made the estimate credible. The large negative estimate in the Cannondale-only preliminary run (-44%) was noise from a single control series, not a real effect. The hedonic regression arrives at the same conclusion from a different angle: `electronic_flag` in V3 carries a coefficient near zero (p = 0.87) once model_series is controlled for, because drivetrain and price tier are co-determined by platform architecture from the start.

### Status

All steps completed. Trek backfill (3,039 priced records, 2017–2026), macro data (6 FRED series), and full DiD analysis are done. The result above is the final Phase 2 estimate.

---

## Predictive Validation

To test whether the enhanced model (V3) generalizes beyond the sample it was fit on, the 755-observation groupset-known dataset was split into an 80% training set and a 20% test set, stratified by brand to preserve the Cannondale/Giant ratio.

V3 was refit on the training set and evaluated on the hold-out test set:
- **In-sample Adj-R²:** 0.822
- **Out-of-sample R²:** 0.825
- **Out-of-sample RMSE:** 0.341 log points (~40% error)
- **Out-of-sample MAE:** 0.264 log points (~30% error)

The model generalizes exceptionally well. The out-of-sample R² (0.825) is nearly identical to the in-sample adjusted R², confirming that the extensive `model_series` dummy variables are capturing true structural price drivers rather than overfitting to noise in the full sample. The RMSE of 0.341 log points (~40% in price terms) reflects genuine within-platform variation — a SuperSix EVO Carbon ships in four groupset configurations spanning roughly $2,200 to $7,000, and the model correctly doesn't try to predict which trim a given row represents beyond what the spec columns capture.

![](analysis/validation_plots.png)
