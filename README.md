# 3M Technical Case — NYC TLC Yellow Taxi Analytics

[![Status](https://img.shields.io/badge/status-complete-brightgreen)]()
[![Stack](https://img.shields.io/badge/stack-Microsoft_Fabric-blue)]()
[![Power BI](https://img.shields.io/badge/Power_BI-Desktop-F2C811)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()

> Technical case for a **Data / Analytics / BI** position on 3M's global **Enterprise Supply Chain Analytics** team.

## Overview

End-to-end analytics-engineering solution on the public **NYC TLC Yellow Taxi (2022–2024)** dataset, reframing taxi operations as an **urban supply chain** (demand, lead time, O–D lanes, operational efficiency, cost-to-serve and exception management).

The solution covers the full flow:

1. **Data Intake** — programmatic ingestion of 36 Parquet files (3 years × 12 months) from the TLC public CDN.
2. **Harmonization & Cleansing** — cleaning, typing, field derivation and quality flags in Spark SQL.
3. **Enrichment** — joins with external sources: **NYC ACS Demographics** (income, population, density by borough) and **NOAA Weather** (historical climate).
4. **Data Model** — star schema in Delta Lake (1 fact + 6 dimensions) materialized in the Lakehouse.
5. **Front-end** — Power BI report connected via **DirectLake**, using Time Intelligence, Field Parameters, Calculation Groups, Conditional Formatting, RLS and DAX UDFs.

## Stack

| Layer | Tool |
|---|---|
| Storage & Compute | Microsoft Fabric Lakehouse (OneLake + Delta) |
| Ingestion | Fabric Notebook (PySpark) |
| Transformation | Fabric Notebook (Spark SQL) |
| Semantic Model | Power BI semantic model on DirectLake |
| Front-end | Power BI Desktop |
| Advanced DAX | Tabular Editor 2 |
| Versioning | GitHub |
| Local fallback | Python + Polars |

## Repository structure

```
3m-techcase-nyc-tlc/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md         # technical architecture
│   ├── CONVENTIONS.md          # naming and code conventions
│   ├── DATA_SOURCES.md         # data sources and how to access them
│   ├── DATA_DICTIONARY.md      # Gold tables data dictionary
│   ├── PRESENTATION_NOTES.md   # technical decisions + storytelling for the talk
│   ├── CODE_EXPLAINED.md       # didactic walkthrough of the code
│   ├── CHECKPOINT_REVIEW.md    # mid-project review
│   └── WIREFRAMES.md           # Power BI dashboard page layouts
├── notebooks/                  # exported from Fabric
│   ├── 01_bronze_ingest.ipynb     # ingestion + Delta persistence
│   ├── 02_silver_clean.ipynb      # quality filters + derivations
│   ├── 03_gold_model.ipynb        # star schema + ACS + NOAA enrichment
│   └── 04_gold_explore.ipynb      # Gold exploratory analysis
├── powerbi/
│   ├── 3m_techcase.pbix           # composite report (DirectLake + local model)
│   └── theme.json                 # corporate navy theme (#1F3864)
├── presentation/
│   └── 3M_TechCase_Presentation.pptx   # final deck (15 slides, English)
├── images/                     # diagrams (dashboard home-menu SVGs)
├── sql/                        # auxiliary SQL scaffolding
├── python_fallback/            # local plan-B ETL scaffolding (Polars)
├── .env.example                # environment variables template
├── .gitignore
├── requirements.txt
└── LICENSE
```

## Prerequisites

- Microsoft 365 Developer Program tenant
- Active Microsoft Fabric Trial
- Power BI Desktop ≥ 2.140 (with preview features: UDFs, Calculation Groups, DirectLake)
- Tabular Editor 2
- Python 3.10+

## Environment setup

```bash
git clone https://github.com/Nicolas-arch/3m-techcase-nyc-tlc.git
cd 3m-techcase-nyc-tlc

# venv for the local plan-B
python -m venv .venv
.venv\Scripts\activate         # Windows
pip install -r requirements.txt

# environment variables
copy .env.example .env         # fill in tokens
```

Required tokens (free signup):

- **NOAA Climate Data Online**: https://www.ncdc.noaa.gov/cdo-web/token
- **NYC Open Data App Token**: https://data.cityofnewyork.us/profile/app_tokens

## Project status

| Stage | Status |
|---|---|
| Infrastructure setup (Fabric Workspace, Lakehouse, GitHub, venv) | ✅ done |
| Bronze ingestion (36 parquets + zone lookup) | ✅ done |
| Silver cleansing (filters, derivations, anomaly flag) | ✅ done |
| Gold modeling + Enrichment (ACS + NOAA) | ✅ done |
| Power BI semantic model (DirectLake) | ✅ done |
| DAX features (Calculation Groups + Field Parameters + UDFs) | ✅ done |
| RLS (6 roles) + corporate theme + Conditional Formatting | ✅ done |
| Dashboard — 6 pages (Executive · Demand · Operations · Demographics · Data Quality/RLS · Weather) | ✅ done |
| Consistency check (1920×1080 canvas, theme, 6 mandatory features) | ✅ done |
| Presentation deck (.pptx — 15 slides, English) | ✅ done |

## Development highlights

### Bronze
- **119,136,044 trips** ingested into Delta from 36 Parquet files (~1.85 GB).
- **Explicit canonical schema** to resolve schema drift across TLC files (INT vs BIGINT, `airport_fee` vs `Airport_fee`).
- **Idempotent ingestion**: re-running does not duplicate files or rows.
- **663 records found with out-of-scope dates** (2001–2026) — a data-quality discovery.

### Silver
- **115,756,175 trips** after cleansing (97.16% of the Bronze volume).
- **3,379,869 rows discarded (2.84%)** with a documented breakdown:
  - 2,123,821 with distance ≤ 0 (cancellations / meter errors)
  - 1,365,574 with negative fare (refunds / corrections)
  - 61,114 with invalid duration (dropoff ≤ pickup)
  - 663 with temporal leakage (2001–2026)
- **180,145 anomalies flagged** (0.16%) — kept in the dataset for analysis on the Data Quality page.
- **Top anomaly detected**: a **US$401,092** trip lasting 10 minutes (obvious meter error) — becomes an exception-management case in the presentation.
- **YoY pattern**: −3.4% in 2023, +7.5% in 2024 — a post-pandemic demand-recovery narrative.

### Gold (Star Schema)
- **1 fact** (`fact_trips` — 115.7M trips partitioned by year/month) + **6 dimensions**.
- **dim_date enriched with NOAA weather** (1,096 days × TMAX, TMIN, PRCP, SNOW, weather_category, is_extreme_weather).
- **dim_zone enriched with ACS Borough demographics** (population, median income, density).
- **Referential integrity validated**: zero orphan foreign keys in the fact (COALESCE → Unknown pattern).
- **Hybrid external-source pattern**: NOAA via REST API with yearly pagination (operational data), ACS via Borough-level snapshot (reference data).
- **Documented gotcha**: the NOAA CDO API limits the range to 1 year per request — fixed with a per-year loop.

### DAX Advanced Features
- **Calculation Group "Time Analytics"** built with Tabular Editor 2 using 7 calculation items (Current, PY, YoY, YoY%, YTD, PYTD, MAT). Instead of duplicating 10 measures × 7 transformations = 70 measures, it keeps 10 measures + 7 items = 17 objects.
- **2 Field Parameters** (FP_Metric with 5 measures + FP_Dimension with 5 dimensions) — 25 analytical combinations in a single visual.
- **3 DAX UDFs** (User-Defined Functions, 2025 preview feature): `ClassifyTripDuration`, `ClassifyFareTier`, `ComputeRevenuePerMile` — encapsulating reusable logic.
- **YoY pattern validated by the Calc Group**: +27.4% in 2023 (post-pandemic recovery) and +5.7% in 2024 (stabilization).
- **Field Parameters insight**: Afternoon + Evening = 70% of daily volume — a classic urban-demand pattern.
- **UDF insight**: EWR (Newark Airport) shows Revenue per Mile of $24.05 — 5× the Manhattan average, a premium-niche pattern.
- **Documented gotchas**: DAX UDF syntax with no quotes in DEFINE, the preview feature requires a Power BI Desktop restart, BLANK→0 coercion in SWITCH.

### Power BI Semantic Model (DirectLake / Composite)
- **`sm_yellow_taxi_3m`** built in DirectLake over the `lh_yellow_taxi` Lakehouse.
- **Power BI Desktop connected via composite model** (DirectQuery + remote semantic model) — allows adding local measures, hierarchies and RLS without losing zero-copy.
- **7 relationships** configured in the semantic model (1 fact + 6 dimensions + 1 role-playing Dropoff Zone).
- **4 hierarchies** created: Date (Year → Quarter → Month → Day), Time (Day Part → Hour), Pickup Zone (Borough → Service Zone → Zone Name), Dropoff Zone.
- **`dim_date` marked as Date Table** — enables DAX Time Intelligence.
- **10 base measures** created in `_Measures` (Total Trips, Trips per Day, Total Revenue, Avg Fare, Total Tip, Tip Rate %, Total Distance, Avg Distance, Avg Duration, Anomaly Rate %).
- **17 technical columns hidden** (surrogate keys + business IDs) — Kimball pattern.
- **Cross-validation**: the Power BI matrix matches 100% with the business sanity query run on Gold (Manhattan 2024: 35.27M trips, US$844M; total 2024: 39.7M trips, US$1.14B).

### RLS, Theme & Conditional Formatting
- **6 RLS roles** configured in the Fabric semantic model: 5 Borough Managers (Manhattan, Brooklyn, Queens, Bronx, Staten Island) + 1 Global Manager.
- **Corporate `powerbi/theme.json`** applied (navy palette `#1F3864`).
- **"Conditional Format Lab" page** (internal) with 4 validated techniques: dynamic color via DAX measure, heat map, data bars, traffic-light icons.
- **Documented gotcha**: a Composite Model does NOT show remote roles under "View as" in Power BI Desktop — demonstrated instead via equivalent DAX measures (`Manhattan Manager View = CALCULATE([Total Revenue], dim_zone[borough]="Manhattan")`).

### Page 1 — Executive Overview
- **1920×1080 canvas** (standard across all 6 pages) with a navy header + 5 KPI cards + trendline + borough chart + Top 10 O–D lanes table.
- **2+3 chromatic hierarchy**: 2 signal cards (Revenue + Anomaly Rate, dynamic color via DAX) + 3 neutral informational cards.
- **EN internationalization**: all dynamic subtitles in English (global 3M audience).
- **KPI hero swapped**: removed YoY Growth (redundant with the Revenue card subtitle) → added **AVG LEAD TIME (min) = 17.56** as a universal supply-chain KPI.
- **8 new DAX measures**: `Anomalous Trips`, `Anomaly Rate`, `Anomaly Color`, `Anomaly Subtitle`, `Avg Lead Time (min)`, `Lead Time Subtitle`, `Trips Subtitle`, `Fare Subtitle`, `Revenue Subtitle` — all using the MAX(year) pattern for robust YoY in any context.
- **3 critical DAX gotchas discovered and documented** (interview-worthy):
  - DAX does not coerce INT ↔ BOOL (the `is_anomalous` flag needs `= 1`, not `= TRUE()`).
  - `FORMAT()` respects the client locale — always force `"en-US"` as the 3rd argument in global models.
  - The `MAX(year)` pattern is more robust than `DATEADD()` for YoY in a multi-year context.
- **3 new business insights**:
  - Fare boom 2022→2023 (+33.1%) — analogous to a peak-season surcharge in supply chain.
  - Fare plateau 2024 (−1.0%) — revenue growth through volume, not price (healthy).
  - **Lead-time variability < 1% over 3 years** (17.46→17.59→17.63 min) — a gold KPI for a global S&OP, with its own slide.

### Dashboard — Pages 2 to 6
- **Page 2 — Demand Deep Dive**: hour×day heat map (Matrix + color scale), seasonality by year, top zones, and a Field-Parameters explorer. `day_order_mon` column in Gold to order Mon→Sun. Insight: two demand regimes (late-afternoon weekday peak + weekend late-night); 31% of demand in 4h/day.
- **Page 3 — Operations & Cost**: distance×fare scatter, lead-time histogram (`duration_class`, mirroring the `ClassifyTripDuration` UDF), fare composition (stacked bar), and a cost-to-serve trend. `ComputeRevenuePerMile` UDF live. Insight: 78% of trips in 5–30 min (consistent lead time); +33% fare shock in 2023.
- **Page 4 — Demand vs Demographics** (mandatory external source): over/under-served diverging bar, demand×income combo, ABC concentration treemap, and a scorecard. Pearson correlation in DAX. Insight: Manhattan over-served ~4× (75% revenue / 19% population); Brooklyn under-served.
- **Page 5 — Data Quality & RLS/Governance**: discard breakdown, anomaly rate by borough, a **Row-Level Security showcase** (6 roles), and top anomalies by zone. Closes the 6th mandatory feature.
- **Page 6 — Weather Impact & Operational Risk** (NOAA external source in use): season combo chart (throughput × lead time by season), 3 Normal-vs-Extreme mini-charts, a season matrix, and insight boxes. Key insight: Fall = bimodal stress (2nd-highest volume + worst lead time); Spring = efficient peak volume; 4 extreme days in 3 years — demand self-selection.
- **Final check**: 6 pages on a 1920×1080 canvas, consistent navy theme, all 6 mandatory features covered.

Technical decisions and standard interview answers in [`docs/PRESENTATION_NOTES.md`](docs/PRESENTATION_NOTES.md).
Didactic code walkthrough in [`docs/CODE_EXPLAINED.md`](docs/CODE_EXPLAINED.md).
Dashboard layout in [`docs/WIREFRAMES.md`](docs/WIREFRAMES.md).

## Test criteria

- ✅ Scope: Yellow Taxi 2022–2024
- ✅ Correlated external source (ACS Demographics + NOAA Weather)
- ✅ Mandatory features: Time Intelligence, Field Parameters, Calculation Groups, Conditional Formatting, RLS, UDFs

## Author

Nicolas Augusto — Campinas/SP, Brazil

## License

[MIT](LICENSE)
