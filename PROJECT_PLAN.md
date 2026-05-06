# Texas On‑Demand Area Research & Investment Scoring (Tract / County / ZIP)

## Purpose
Build an **on-demand**, data-driven system that ranks **areas** in Texas for rental-property investing using objective fundamentals and trends. Users select one or more **counties** (or all), and the pipeline produces **Tableau-ready long CSVs** and **scores computed only within the selected counties** across **all available ACS 5‑year end-years** (roughly the last 10 years).

This is an **area selection** tool (where to buy), not an individual deal underwriting tool (which house to buy).

---

## 1) Scope and key behaviors
### 1.1 State and county selection
- State: **Texas (pilot)**
- Input: county list (multi-select) or “all counties”
- Output scores are **normalized within the selected counties** (Option A). This means the same tract’s score can change if the county set changes.

### 1.2 Geographies supported
- **Census tract** (primary scoring unit)
- **County** (rollup of tract scores)
- **ZIP/ZCTA** (rollup of tract scores via crosswalk weights)

### 1.3 Time window
- **All available ACS 5-year end-years** in a requested range (e.g., 2015 → latest released)
- All outputs include all years (no “latest only” mode in v1).

---

## 2) Why tract-first (design rationale)
### Tracts
- Stable, neighborhood-sized statistical units
- Strong coverage in **ACS 5‑year**
- Reveals “pockets” inside counties that county or ZIP averages can hide

### ZIP/ZCTA
- Included as a secondary lens because investors and agents often search by ZIP
- Produced by rolling up tract scores (keeps ZIP scores consistent with tract scoring logic)

---

## 3) Data sources
### 3.1 Core: US Census ACS 5‑year
Used for tract-level fundamentals:
- population & households
- income & poverty
- rent & rent burden
- vacancy
- tenure (rent/own)
- mobility (% moved in last 12 months)
- housing stock (structure type, year built)

Store **MOE (Margin of Error)** for metrics where available.

### 3.2 Geography support: TIGER/Line
- tract/county IDs and names
- centroids (lat/lon) for Tableau mapping

### 3.3 Crosswalks
- tract ↔ ZCTA crosswalk with weights (based on **households**)

### 3.4 Optional extensions (future)
- FHFA HPI (county/metro) for repeat-sales price index backbone
- BLS/BEA for county-level labor and income series
- Local open-data (permits, code violations, crime) via spatial joins

---

## 4) Output deliverables (CSV)
All generated for the selected county set and include all years.

### 4.1 Dimension files
1. `TX_geo_county_dim.csv`
   - `state_abbr, state_fips, county_name, county_fips, county_geoid`

2. `TX_geo_tract_dim.csv`
   - `geoid, tract_name, county_geoid, county_name, state_abbr, lat, lon`

3. `TX_geo_zcta_dim.csv` *(optional but recommended)*
   - `zcta, lat, lon`

4. `TX_xwalk_tract_zcta.csv`
   - `geoid, zcta, weight_households`

### 4.2 Metrics (long format)
5. `TX_metrics_tract_long.csv`
   - `geoid, year, metric, value, moe`

### 4.3 Scores (all years)
6. `TX_score_tract.csv`
   - `geoid, year, cashflow_score, appreciation_score, trend_score, balanced_score`
   - optional driver columns/sub-scores for explainability

7. `TX_score_county.csv` *(household-weighted rollup)*
   - `county_geoid, year, cashflow_score, appreciation_score, trend_score, balanced_score`

8. `TX_score_zcta.csv` *(crosswalk-weighted rollup)*
   - `zcta, year, cashflow_score, appreciation_score, trend_score, balanced_score`

---

## 5) Metrics (deep but investor-relevant)
Exact ACS variable codes will be specified during implementation, but the v1 metric set covers the highest-signal drivers for rental investing.

### A) Demand & tightness
- households (level + trend)
- population (context)
- vacancy rate (level + trend)

### B) Ability to pay / distress
- median household income (level + trend)
- poverty rate (level + trend)

### C) Rental fundamentals
- median gross rent (level + trend)
- rent burden: % paying ≥30% of income on rent (lower is better)

### D) Mobility / churn (moderate is best)
- % moved in last 12 months

### E) Housing stock / capex risk proxies
- structure type mix (1-unit, 2–4, 5+)
- year built mix (newer stock share)

### F) Value context (fundamentals proxy)
- median home value (level + trend)

---

## 6) Scoring design (computed within selected counties)
Scores are computed for each year independently using the tract set inside the selected counties.

### 6.1 Normalization
For each year, for each metric:
- compute percentile rank across included tracts (0–100)
- invert “bad” metrics so higher is always better:
  - vacancy, poverty, rent burden

### 6.2 Mobility scoring (moderate is best)
Mobility is treated as a non-linear feature:
- define an “ideal band” (e.g., 35th–65th percentile within the county-set)
- mobility score is highest inside the band and decreases toward extremes

### 6.3 Sub-scores
#### Cashflow score (0–100)
Goal: stable occupancy + tenants can pay + rents supported without severe affordability stress.
Typical inputs:
- income level and trend
- vacancy (inverted) and vacancy trend
- rent burden (inverted)
- rent level and trend
- mobility moderation score
- stock quality proxy

#### Appreciation fundamentals score (0–100)
Goal: fundamentals supportive of future price/rent strength.
Typical inputs:
- income trend
- household growth trend
- improving vacancy
- lower/improving poverty
- stock desirability proxy

*(Optional later: blend in FHFA HPI baseline trend.)*

#### Trend score (0–100)
Goal: “is it improving?”
- income growth
- rent growth
- vacancy improvement
- poverty improvement
- (optional) household growth

#### Balanced score (0–100)
Default weighting (configurable):
- `balanced = 0.6 * cashflow + 0.4 * appreciation`

### 6.4 Rollups
Weights: **households**.
- County score: household-weighted average of tract scores
- ZCTA score: crosswalk household-weighted average of tract scores

---

## 7) Tableau usage patterns
Recommended dashboards:
1. **Map view**: tracts colored by balanced score or cashflow score
2. **Ranking table**: top/bottom tracts with driver metrics
3. **Trend view**: multi-year score trends for selected tract/county/ZCTA
4. **ZIP lens**: browse ZIP/ZCTA summary then drill into high-performing tracts
5. **Confidence/risk view**: optional MOE-based flags and filters

---

## 8) Execution plan (build order)
### Phase 1 — Foundation
- Generate/cached Texas geography dims (counties, tracts, centroids)
- Define v1 metric list + ACS variable codes

### Phase 2 — Data extraction
- Pull ACS 5‑year tract metrics for all Texas tracts and all years
- Write `TX_metrics_tract_long.csv`

### Phase 3 — Scoring engine
- Inputs: selected county list, year range
- Compute normalized components per year within county-set
- Compute sub-scores and final scores per year
- Write `TX_score_tract.csv`

### Phase 4 — Rollups
- County rollup (household weights) → `TX_score_county.csv`
- ZCTA rollup (crosswalk weights) → `TX_score_zcta.csv`

### Phase 5 — Tableau packaging
- Provide a recommended Tableau data model (relationships keyed by `geoid`, `county_geoid`, `zcta`, and `year`)
- Provide dashboard templates and metric definitions

---

## 9) Open items to finalize during implementation
- Final ACS variable list and codes (with MOE fields)
- Mobility “ideal band” selection and penalty curve
- Weighting scheme for sub-scores (start with defaults; keep config-driven)
- MOE-based confidence flagging rules
- Optional integration of FHFA HPI and BLS/BEA series
