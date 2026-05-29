# Glossary & Definitions: Texas Real Estate Investment Analysis

This document provides a comprehensive technical guide to the datasets, column definitions, and scoring methodologies used in this project.

---

## 1. Project Logic: Is "High" Good or Bad?

In this system, **Higher is always Better**. 

All raw metrics are processed so that the final scores (0 to 100) reflect investment desirability:
- **Score = 100:** Top-tier performance (e.g., highest rent growth, lowest vacancy).
- **Score = 0:** Bottom-tier performance (e.g., highest poverty, stagnant income).
- **Rankings:** When a column is labeled as a `_rank`, it refers to a percentile rank (0-100%). A rank of 95 means the tract performed better than 95% of other tracts in that specific category.

---

## 2. File Glossary

### 2.1 Processed Scores (`data/processed/`)
| File Name | Description |
| :--- | :--- |
| `TX_score_tract.csv` | Primary scoring file for all ~6,800 Texas Census Tracts. |
| `TX_score_county.csv` | Household-weighted averages at the County level. |
| `TX_score_zcta.csv` | Weighted averages for ZIP Codes (ZCTAs). |

### 2.2 Raw Metrics (`data/processed/`)
| File Name | Description |
| :--- | :--- |
| `TX_metrics_tract_long.csv` | Raw ACS metrics used for scoring. Each row is one metric for one tract. |

### 2.3 Dimension Files (`data/cache/`)
| File Name | Description |
| :--- | :--- |
| `TX_geo_tract_dim.csv` | Names, labels, and centroids (Lat/Lon) for every tract. |
| `TX_geo_county_dim.csv` | Mapping of County FIPS codes to Names. |
| `tx_tracts.geojson` | Geometric boundaries for high-resolution mapping. |

---

## 3. Field Definitions: Processed Scores (0-100)

| Column | Definition | Directional Logic |
| :--- | :--- | :--- |
| `cashflow_score` | Composite focus on current yield. | High = High rents + High income + Low vacancy. |
| `appreciation_score` | Composite focus on future value growth. | High = High income growth + New stock + Low poverty. |
| `trend_score` | Average of all growth/momentum ranks. | High = The neighborhood is "heating up". |
| `balanced_score` | Primary Investment Signal. | Weighted blend (60% Cashflow / 40% Appreciation). |
| `deal_score` | Notebook-specific screening score. | Blends Balanced Score, Trend, and Risk Penalties. |
| `median_income_growth_rank` | Percentile rank of income growth. | High = Rapidly rising wealth. |
| `median_rent_growth_rank` | Percentile rank of rent growth. | High = Strong rental demand. |
| `vacancy_rate_growth_rank` | Percentile rank of vacancy change. | **Inverted:** High = Vacancy is dropping fast. |
| `poverty_rate_growth_rank` | Percentile rank of poverty change. | **Inverted:** High = Poverty is dropping fast. |

---

## 4. Field Definitions: Raw Metrics (Building Blocks)

Found in `TX_metrics_tract_long.csv` and used to derive the scores above.

### 4.1 Economic Indicators
| Metric | Definition | Significance |
| :--- | :--- | :--- |
| `median_income` | Total household income (Inflation adj). | Measures neighborhood wealth and tenant quality. |
| `poverty_rate` | % pop below federal poverty line. | Measures economic risk; used as a penalty. |
| `mobility_rate` | % pop moved recently. | Measures churn. High = dynamic, Low = stagnant. |

### 4.2 Housing Market Indicators
| Metric | Definition | Significance |
| :--- | :--- | :--- |
| `median_rent` | Median "Gross Rent" (Rent + utilities). | Primary driver for rental yield. |
| `median_home_value` | Owner-estimated property value. | Baseline for market pricing and entry cost. |
| `rent_burden_30_plus` | % tenants paying >30% income on rent. | Measures default risk. High = unstable. |
| `vacancy_rate` | % housing units that are empty. | High = oversupply or low demand. |

### 4.3 Stock & Demographic Indicators
| Metric | Definition | Significance |
| :--- | :--- | :--- |
| `new_stock_share` | % homes built since 2010. | High = modern stock, lower maintenance. |
| `single_family_share` | % units that are detached houses. | Measures the dominant asset type. |
| `households` | Total occupied units. | Used to weight averages for rollups. |
| `population` | Total residents. | Baseline for growth and density. |

---

## 5. Data Interpretation Samples

### Success Scenario (Top Performance)
| geoid | balanced_score | trend_score | Signals |
| :--- | :--- | :--- | :--- |
| 48113014126 | **88.4** | **92.1** | Strong current yield AND massive momentum. |

### Mature Market (Stable Performance)
| geoid | balanced_score | trend_score | Signals |
| :--- | :--- | :--- | :--- |
| 48113000100 | **72.0** | **15.4** | Good yield but stagnant growth. Safe, but not explosive. |

---

## 6. Mapping & Tableau Instructions

### 6.1 Why is the map empty?
1.  **GEOID Format:** IDs must be **Strings** with leading zeros (County: 5 digits, Tract: 11 digits).
2.  **Internet Access:** Required to fetch County GeoJSON in the Notebook.
3.  **Coordinate Join:** Join `df_tract` to `TX_geo_tract_dim.csv` to enable `lat`/`lon` scatter maps.

### 6.2 Using Polygon Data in Tableau
For high-resolution neighborhood borders:
1.  **Connect:** Click **Add** -> **Spatial File** and select `real_estate/data/cache/tx_tracts.geojson`.
2.  **Join:** Join to `TX_score_tract.csv` on **`GEOID`** (JSON) = **`geoid`** (CSV).
3.  **Map:** Double-click the **Geometry** field.
4.  **Quick Role:** Alternatively, right-click `geoid` -> **Geographic Role** -> **Census Tract** to use Tableau's built-in boundaries.

Here is links to check tract profile 
https://censusreporter.org/profiles/14000US48113014126-census-tract-14126-dallas-tx/


