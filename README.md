# NYC Taxi Medallion Lakehouse

An end-to-end medallion lakehouse over real NYC Taxi & Limousine Commission (TLC) Yellow Taxi trip data — full calendar year 2024, ~41.2 million trips — built as a portfolio project and DP-700 preparation vehicle.

## Why this was built on Databricks, not Fabric

This project was originally designed entirely around Microsoft Fabric. Over the course of setup, it hit five consecutive administrative walls on a training-provided tenant: GitHub sync disabled at the tenant level, Azure self-service signup blocked by organizational policy, Fabric trial capacity creation disabled, a personal-account workaround whose trial eligibility was then exhausted, and finally free-tier item creation that worked briefly and then stopped. None of these were engineering problems.

Rather than lose the momentum, the entire build was rewritten for Databricks Free Edition — genuinely free, no tenant admin dependency, no trial to expire. Everywhere the two platforms diverge (Direct Lake semantic models, Eventstream/Eventhouse/KQL, Fabric-specific security and administration), the equivalent Fabric approach is documented separately, to be built out once real Fabric access exists through an employer. See `docs/decisions.md` for the full reasoning trail.

## Tech stack

Databricks Free Edition (serverless compute) · Unity Catalog · Delta Lake · PySpark · Databricks SQL · Databricks Workflows (Lakeflow Jobs) · Databricks Asset Bundles · GitHub Actions (CI/CD)

## Architecture

    NYC TLC public data (monthly Parquet files)
            |
            v
       BRONZE  (nyc_taxi.bronze)      raw, append-only, ingestion lineage metadata, idempotency-guarded
            |
            v
       SILVER  (nyc_taxi.silver)      cleaned + quarantined, partitioned by pickup_date, idempotent MERGE
            |
            v
       GOLD    (nyc_taxi.gold)        star schema: fact_trips + dimensions, row/column security
            |
            v
       BI layer (Databricks SQL dashboards)

Orchestrated end to end by a Databricks Workflow (`bronze_ingest` -> `silver_transform` -> `gold_build`), deployed via Databricks Asset Bundles and a GitHub Actions pipeline that redeploys on every push to `main`. Full diagram above.

## Data model (Gold layer)

Grain: one row in `fact_trips` represents one completed taxi trip.

| Table | Type | Notes |
|---|---|---|
| `fact_trips` | Fact | Foreign keys to every dimension below, plus distance/fare/duration measures |
| `dim_date` | Dimension | Generated independently via date sequence, not derived from fact data |
| `dim_taxi_zone` | Dimension | Base table, 265 TLC zones + a -1 Unknown row for referential integrity |
| `dim_pickup_zone` / `dim_dropoff_zone` | Views | Role-named views over `dim_taxi_zone`, avoiding the single-active-relationship limitation in Power BI/Direct Lake |
| `dim_rate_code` | Dimension | TLC rate code lookup |
| `dim_payment_type` | Dimension | TLC payment type lookup |

`dim_taxi_zone` is a role-playing dimension - joined twice (pickup, dropoff). Rather than one shared table with two foreign keys (which breaks Power BI's single-active-relationship rule), two thin views expose it under two names with zero data duplication.

## Design decisions

Pulled from `docs/decisions.md`:

1. **Bronze idempotency guard, not a MERGE-based upsert.** Re-running ingestion for an already-loaded month was silently duplicating rows. Considered switching Bronze to MERGE semantics; rejected - raw taxi trip records have no natural unique key, so a merge would need an invented composite/hash key, and it blurs Bronze's raw/replayable purpose into Silver's job. Went with an explicit guard that raises on a duplicate partition instead.
2. **Explicit, logged `overwrite` override - not a silent bypass.** Legitimate re-ingestion (a source correction, or recovering from a bad run) still needs a path around the guard. Rejected quietly removing or disabling it; added a default-`false` override flag that deletes-and-reinserts only when explicitly set, so every override is visible and auditable.
3. **Cross-layer row-count validation as a standard check, not run-status alone.** A green checkmark on all three orchestration tasks turned out not to guarantee the data was correct or complete. Added a repeatable Bronze-vs-Gold count comparison against prior months' drop rate, rather than trusting "Succeeded" at face value.
4. **CI/CD auth via personal access token, not a service principal.** OAuth machine-to-machine auth via a service principal is the standard enterprise pattern; rejected for now since this is a solo, single-workspace Free Edition project - PAT is the correctly-scoped choice here, with service principal noted as the upgrade path if this ever runs multi-user.
5. **Unity Catalog row/column security policies attached to the table, not separate views per audience.** Considered maintaining separate admin/public views; rejected - views multiply per audience and have to be kept in sync with every schema change, while a policy function attached once to the canonical table scales better as access rules get more nuanced.
6. **A logic bug in the fare/total validation** - a payment-type exemption meant for legitimately low no-charge/dispute fares was accidentally also protecting negative fares sharing the same payment type, letting 418,882 rows with impossible negative values into clean. Fixed by separating unconditional negative checks from the exempted below-minimum checks.
7. **A temporal plausibility gap** - every original rule validated a value in isolation; nothing checked whether a row's date belonged in the batch at all. 55 rows with pickup dates outside 2024 passed every check and reached `fact_trips` undetected. Fixed with a new rule and a full rebuild of Silver and Gold from Bronze, since MERGE-based tables can only insert new rows, not retroactively correct history already merged in.

## Data quality

Every Bronze row is tagged with a single `_reject_reason` (or NULL) in one pass, then split into `trips_clean` and `trips_quarantine` - invalid rows are quarantined with a reason attached, never silently dropped.

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
| **Total quarantined** | **5,542,350** | **13.46%** |

Bronze total: 41,169,720. Clean: 35,627,370 (86.54%).

> **Open item, flagged rather than smoothed over:** a later cross-layer check (see decision 3, and `docs/decisions.md`) found January/February/March 2024 individually running at 17.7% / 18.3% / 24.3% Bronze->Gold drop - all above this 13.46% full-year average. Working hypothesis, not yet confirmed: 13.46% is a blended annual figure, and a rough back-of-envelope split suggests Q1 running high-teens/low-20s against the other nine months running closer to 11% would still average out correctly - plausible, but unverified against the actual month-by-month numbers from the original build. March's 24.3% specifically is still open and not yet root-caused.

## Security & governance

Unity Catalog row-level security and column masking are applied to `fact_trips`:

- **Row filter** - non-admin users only see rows where `pickup_zone_key` resolves to a Manhattan zone; members of `taxi_admins` see everything.
- **Column mask** - `total_amount` returns the real value for admins, `NULL` for everyone else. The column still exists and type-checks for non-admins; it just never carries a value.

Both are attached to the table object in Unity Catalog, not to any individual query - so the policy holds regardless of which tool or notebook touches `fact_trips`. Known limit: this doesn't cover the case of something reading the same underlying files by bypassing the catalog entirely (relevant to the Fabric equivalent, where SQL-endpoint security doesn't automatically extend to a direct OneLake read).

## Key findings so far

- Revenue share by day of week peaks on Thursday (15.9%), not the weekend - a genuine mid-week peak rather than the more obvious weekday/weekend guess.
- Trip volume is lowest in January-February and July-August, highest in autumn - a volume pattern, not a fare pattern (average fare rises steadily through the year regardless).
- Manhattan dominates pickup volume overwhelmingly; Queens' average trip distance is roughly 5x Manhattan's, consistent with JFK/LaGuardia airport traffic.

## Performance

> Deliberately left blank. An initial Z-ORDER attempt reported `numFilesAdded: 0, numFilesRemoved: 0` - it changed nothing, despite an accompanying benchmark that looked like a 94.8% improvement. That number didn't survive scrutiny: it was comparing a cold session against a warm one, not a real before/after. A clean re-test was attempted and hit a separate, unresolved session-state bug (the identical query returned a corrupted single-row result on a warm session). No number goes here until a same-query, fresh-session, one-verified-change comparison actually produces one - an unverified figure is worse than an empty section.

## Known limitations

- March 2024's Bronze->Gold drop rate (24.3%) is flagged against history and not yet root-caused.
- The Bronze->Gold quarantine-rate discrepancy above (13.46% annual vs. 17-24% for Q1 individually) is a working hypothesis, not a confirmed explanation.
- The orchestration job's month parameter is static - nothing currently advances it between scheduled runs, so the daily trigger isn't safe to leave running unattended yet.
- The `overwrite` re-ingestion path deletes then reinserts, non-atomically - a failure between those two steps would leave that month's partition empty rather than duplicated.
- Single Free Edition workspace - no real dev/prod separation; PAT-based CI/CD auth rather than a service principal (see decision 4).

## Repository structure

    nyc-taxi-fabric-lakehouse/
    |-- README.md
    |-- LICENSE
    |-- .gitignore
    |-- databricks.yml                (Asset Bundle definition)
    |-- .github/
    |   |-- workflows/deploy.yml      (CI/CD - deploys the bundle on push to main)
    |-- docs/
    |   |-- decisions.md              (every design decision, with reasoning)
    |   |-- data-quality-report.md    (quarantine numbers and investigation notes)
    |   |-- performance-report.md     (Week 5 - pending a verified before/after number)
    |   |-- architecture.png
    |-- fabric/                       (Fabric item definitions, once real access exists)
    |-- notebooks/                    (exported .ipynb / .py copies)
    |-- sql/                          (dashboard queries)
    |-- scripts/

## Status

- DONE - Week 1 - Foundations (Bronze ingestion, GitHub, catalog structure)
- DONE - Week 2 - Silver (quarantine framework, idempotency proven, data quality dashboard)
- DONE - Week 3 - Gold (star schema, SQL Warehouse, business dashboard)
- DONE - Week 4/5 - Orchestration + CI/CD (Databricks Workflows, Asset Bundles, GitHub Actions) and security (Unity Catalog RLS/CLS) - streaming still open
- TODO - Week 5 - Performance: a verified, defensible before/after number
- TODO - Week 6 - Final shipping polish (LinkedIn post, repo hygiene pass)
- TODO - Fabric bridge - once real Fabric access exists (Direct Lake, Eventstream/KQL, Fabric administration)

## Running this

```bash
# 1. Configure the Databricks CLI (Free Edition workspace)
databricks configure --token

# 2. Validate and deploy the bundle
databricks bundle validate
databricks bundle deploy -t dev

# 3. Trigger a run for a given month via the Workflows UI or CLI
#    Job parameters: year / month / overwrite
databricks bundle run bronze_ingest -t dev
```

See `docs/decisions.md` for why each step is structured the way it is.