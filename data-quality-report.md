# Data Quality Report

## Initial exploration (January 2024 sample — section 2.5)

Profiled before the full year was ingested, as a first look at the data's shape.

| Check | Row count | % of January rows |
|---|---|---|
| Rows where dropoff is earlier than pickup | 56 | 0.0019% |
| Rows with negative fare_amount | 37,448 | 1.2632% |
| Rows with zero or negative trip_distance | 60,371 | 2.0364% |
| Rows with NULL passenger_count | 140,162 | 4.7278% |
| Rows with 0 passenger_count | 31,465 | 1.0613% |
| Rows with missing, NULL, or 0 passenger_count | 171,627 | 5.7892% |

January 2024 total rows: 2,964,624

## Full dataset (Bronze, all 12 months of 2024)

| Check | Row count | % of total rows |
|---|---|---|
| Rows where dropoff is earlier than pickup | 1,575 | 0.0038% |
| Rows with negative fare_amount | 731,024 | 1.7756% |
| Rows with zero or negative trip_distance | 776,305 | 1.8856% |
| Rows with NULL passenger_count | 4,091,232 | 9.9375% |
| Rows with 0 passenger_count | 401,354 | 0.9749% |
| Rows with missing, NULL, or 0 passenger_count | 4,492,586 | 10.9124% |

Total Bronze rows (all 12 months): 41,169,720

## Observations

- Missing passenger_count is nearly double across the full year (10.91%) versus the January-only sample (5.79%) — the January profile was not fully representative on this dimension. Worth checking in Week 2 whether this concentrates in specific months.
- Negative-fare and zero-distance rates are broadly consistent between the sample and the full year, suggesting January was representative on those two checks specifically, just not on passenger_count.

## Week 2 — Silver quarantine breakdown

Total Bronze rows processed: 41,169,720
Clean: 35,627,454 (86.54%)
Quarantined: 5,542,266 (13.46%)

| Reject reason | Row count | % of total |
|---|---|---|
| null_passenger_count | 3,817,047 | 9.2715% |
| zero_or_negative_distance | 776,303 | 1.8856% |
| negative_fare | 554,479 | 1.3465% |
| zero_passenger_count | 383,912 | 0.9325% |
| fare_below_minimum | 6,942 | 0.0169% |
| implausible_distance | 1,613 | 0.0039% |
| dropoff_before_pickup | 1,575 | 0.0038% |
| negative_total | 168 | 0.0004% |
| implausible_passenger_count | 152 | 0.0004% |
| total_below_minimum | 75 | 0.0002% |

**Note:** rules are evaluated in priority order, first match wins — a row failing multiple checks is only counted once, under whichever rule appears first. This is why `null_passenger_count` here (3,817,047) differs from the raw full-year NULL count profiled in Week 1 (4,091,232): roughly 274,000 of those rows also failed an earlier check (missing timestamp, bad distance) and were claimed by that rule instead.