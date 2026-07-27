# NYC Taxi Medallion Lakehouse

An end-to-end medallion lakehouse over real NYC Taxi & Limousine Commission (TLC) Yellow Taxi trip data — full calendar year 2024, ~41.2 million trips — built as a portfolio project and DP-700 preparation vehicle.

## Why this was built on Databricks, not Fabric

This project was originally designed entirely around Microsoft Fabric. Over the course of setup, it hit five consecutive administrative walls on a training-provided tenant: GitHub sync disabled at the tenant level, Azure self-service signup blocked by organizational policy, Fabric trial capacity creation disabled, a personal-account workaround whose trial eligibility was then exhausted, and finally free-tier item creation that worked briefly and then stopped. None of these were engineering problems.

Rather than lose the momentum, the entire build was rewritten for Databricks Free Edition — genuinely free, no tenant admin dependency, no trial to expire. Everywhere the two platforms diverge (Direct Lake semantic models, Eventstream/Eventhouse/KQL, Fabric-specific security and administration), the equivalent Fabric approach is documented separately, to be built out once real Fabric access exists through an employer. See docs/decisions.md for the full reasoning trail.

## Architecture

    NYC TLC public data (monthly Parquet files)
            |
            v
       BRONZE  (nyc_taxi.bronze)      raw, append-only, ingestion lineage metadata
            |
            v
       SILVER  (nyc_taxi.silver)      cleaned + quarantined, partitioned by pickup_date, idempotent MERGE
            |
            v
       GOLD    (nyc_taxi.gold)        star schema: fact_trips + dimensions
            |
            v
       BI layer (Databricks SQL dashboards)

## Tech stack

Databricks Free Edition (serverless compute) - Unity Catalog - Delta Lake - PySpark - Databricks SQL - GitHub (direct Repos integration, no intermediate Git provider)

## Data model (Gold layer)

Grain: one row in fact_trips represents one completed taxi trip.

| Table | Type | Notes |
|---|---|---|
| fact_trips | Fact | Foreign keys to every dimension below, plus distance/fare/duration measures |
| dim_date | Dimension | Generated independently via date sequence, not derived from fact data |
| dim_taxi_zone | Dimension | Base table, 265 TLC zones + a -1 Unknown row for referential integrity |
| dim_pickup_zone / dim_dropoff_zone | Views | Role-named views over dim_taxi_zone, avoiding the single-active-relationship limitation in Power BI/Direct Lake |
| dim_rate_code | Dimension | TLC rate code lookup |
| dim_payment_type | Dimension | TLC payment type lookup |

dim_taxi_zone is a role-playing dimension — joined twice (pickup, dropoff). Rather than one shared table with two foreign keys (which breaks Power BI's single-active-relationship rule), two thin views expose it under two names with zero data duplication.

## Data quality

Every Bronze row is tagged with a single _reject_reason (or NULL) in one pass, then split into trips_clean and trips_quarantine — invalid rows are quarantined with a reason attached, never silently dropped.

| Reject reason | Rows | % of total |
|---|---|---|
| null_passenger_count | 3,817,047 | 9.27% |
| zero_or_negative_distance | 776,303 | 1.89% |
| negative_fare | 554,479 | 1.35% |
| zero_passenger_count | 383,912 | 0.93% |
| fare_below_minimum | 6,942 | 0.017% |
| implausible_distance | 1,613 | 0.004% |
| dropoff_before_pickup | 1,575 | 0.004% |
| implausible_pickup_year | 55 | <0.001% |
| negative_total | 168 | <0.001% |
| implausible_passenger_count | 152 | <0.001% |
| total_below_minimum | 75 | <0.001% |
| implausible_tolls | 29 | <0.001% |
| Total quarantined | 5,542,350 | 13.46% |

Bronze total: 41,169,720. Clean: 35,627,370 (86.54%).

Two real bugs were found and fixed during this build, not simulated for the portfolio:

1. A logic bug in the fare/total validation — a payment-type exemption meant for legitimately low no-charge/dispute fares was accidentally also protecting negative fares sharing the same payment type, letting 418,882 rows with impossible negative values into clean. Fixed by separating unconditional negative checks from the exempted below-minimum checks.
2. A temporal plausibility gap — every original rule validated a value in isolation; nothing checked whether a row's date belonged in the batch at all. 55 rows with pickup dates outside 2024 (as far back as 2002, as far forward as 2026) passed every check and reached fact_trips undetected, discovered only while investigating a monthly revenue pattern. Fixed with a new rule and a full rebuild of Silver and Gold from Bronze (MERGE-based tables can only insert new rows, not retroactively correct history already merged in).

Full reasoning for both, plus every partitioning, modeling, and threshold decision, is in docs/decisions.md.

## Key findings so far

- Revenue share by day of week peaks on Thursday (15.9%), not the weekend — a genuine mid-week peak rather than the more obvious weekday/weekend guess.
- Trip volume is lowest in January-February and July-August, highest in autumn — a volume pattern, not a fare pattern (average fare rises steadily through the year regardless).
- Manhattan dominates pickup volume overwhelmingly; Queens' average trip distance is roughly 5x Manhattan's, consistent with JFK/LaGuardia airport traffic.

## Repository structure

    nyc-taxi-fabric-lakehouse/
    |-- README.md
    |-- LICENSE
    |-- .gitignore
    |-- docs/
    |   |-- decisions.md              (every design decision, with reasoning)
    |   |-- data-quality-report.md    (quarantine numbers and investigation notes)
    |   |-- performance-report.md     (Week 5, not yet started)
    |   |-- architecture.png          (Week 6, not yet built)
    |-- fabric/                       (Fabric item definitions, once real access exists)
    |-- notebooks/                    (exported .ipynb copies)
    |-- sql/                          (dashboard queries)
    |-- scripts/

## Status

- DONE - Week 1 - Foundations (Bronze ingestion, GitHub, catalog structure)
- DONE - Week 2 - Silver (quarantine framework, idempotency proven, data quality dashboard)
- DONE - Week 3 - Gold (star schema, SQL Warehouse, business dashboard)
- TODO - Week 4 - Streaming + orchestration
- TODO - Week 5 - Performance optimization + security
- TODO - Week 6 - Final shipping polish (architecture diagram, LinkedIn post)
- TODO - Fabric bridge - once real Fabric access exists (Direct Lake, Eventstream/KQL, Fabric administration)

## Running this

Requires a Databricks workspace with Unity Catalog enabled. Run notebooks in order: 01_bronze_ingest -> 02_silver_transform -> 03_gold_build. See docs/decisions.md for why each step is structured the way it is.
