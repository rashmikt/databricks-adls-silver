![DataKern Watermark](https://img.shields.io/badge/DATAKERN-Assignment-F26522?style=for-the-badge)
_www.data-kern.com_

# Observation Report: ADLS → Bronze → Silver

> **How to fill this in:** copy every number from **your own run**. Do not round and do not guess. The source files are identical for every student, so row counts should match across the class; a different number is a signal to investigate your pipeline, not to adjust the report.
> Short answers are better than long ones. One or two sentences in your own words, every time you see **Why?**

**Name:** Rashmi        
**Date:** 18-11-2026

---

## Scoreboard (fill this in last)

| What | Your value |
|---|---|
| Storage account | stnycweatherdev001 |
| Catalog used | dbw_nyc_weather_dev |
| Bronze rows: Aug / Sep / Oct / total | 3,574,091 / 4,251,015 / 4,428,699 / 12,253,805 |
| Silver rows (`silver.taxi_trips`) | 10,959,322 |
| Rejected rows (`silver.taxi_trips_rejected`) | 1,294,483 |
| Exact duplicates removed | 0 |
| Weather rows (Bronze / Silver) | 92 / 92 |
| Check cells: notebook 1 passed / total | 10 / 10 |
| Check cells: notebook 2 passed / total | 10 / 10 |
| Trips per day, rainy vs dry (§9) | 2691271 / 8268051 |
| Total time spent (hours) | |

---

## §1 Raw landing (Notebook 1, Step 1)

| Path in ADLS | File size (MB) |
|---|---|
| `raw/taxi/yellow_tripdata_2025-08.parquet` | 75.07 MiB |
| `raw/taxi/yellow_tripdata_2025-09.parquet` | 88.44 MiB |
| `raw/taxi/yellow_tripdata_2025-10.parquet` | 92.36 MiB |
| `raw/weather/nyc_daily_2025-08-01_2025-10-31.json` | 3.85 KiB |

**Why** keep an untouched copy in `raw/` instead of writing Delta straight from the download?

> It acts as a single source of truth. We can re-process the data from raw without needing to re-download files from the source. 

**How** did you move the file from the internet into ADLS on serverless? (the 2–3 calls you used, and what each does)

> requests.get(url): Fetches the raw file bytes from the internet
> sdf.coalesce(1).write.parquet(...) (and dbutils.fs.put for the weather JSON): Writes the data into a temporary folder in ADLS.
> dbutils.fs.cp() and dbutils.fs.rm(): Copies that file into its exact final name (e.g., yellow_tripdata_2025-08.parquet) and cleans up the temporary folder.

---

## §2 Bronze (Notebook 1, Steps 2–3)

| Month | Rows in raw Parquet | Rows in `bronze.taxi_trips` |
|---|---|---|
| 2025-08 | 3,574,091 | 3,574,091 |
| 2025-09 | 4,251,015 | 4,251,015 |
| 2025-10 | 4,428,699 | 4,428,699 |

**Three schema surprises** in the Bronze taxi table (column, type you saw, why it surprised you):

| Column | Type in Bronze | Why odd or surprising |
|---|---|---|
| passenger_count | double | Logically, a passenger count must be a whole integer |
| | | |
| | | |

**Weather:** rows in Bronze = 92. Why is "one JSON object with parallel arrays" an awkward shape for analytics, and what does one row per date make easy?

> Parallel arrays makes the analysis work difficult. Flattening the data into one row per date makes it much easier to filter, analyze over time, and join directly with the taxi trips.

**Partitioning:** list the folders you see under `bronze/taxi_trips/` (paste `dbutils.fs.ls` output). What is each folder, and why did we partition by `source_month`?

> - `source_month=2025-08/`
> - `source_month=2025-09/`
> - `source_month=2025-10/`

---

## §3 Re-run test (Notebook 1, Step 5)

| | Bronze rows (all months) |
|---|---|
| Before re-running Step 2 | 12,253,805 |
| After re-running Step 2 |  12,253,805 |

Number of Delta versions in `DESCRIBE HISTORY` of `bronze.taxi_trips` after the re-run: ______

**Why** did the row count stay the same? What would have happened with plain `append`? Explain `replaceWhere` in your own words.

> The row count stayed the same because our write operation was idempotent. Instead of adding new records, it updated and replaced the existing data cleanly. 
> 
> If we had used plain `append` instead, running the ingestion twice would have inserted the exact same rows a second time, completely doubling our row count to over 24.5 million and creating duplicate data.
> 
> In plain words, `replaceWhere` tells Delta: Find only the rows matching this specific month (like 2025-08), delete them, and swap in the new ones, leaving the other months untouched. This lets us safely re-run the pipeline whenever we want without creating duplicate rows.

---

## §4 Registration evidence (Notebook 1 Step 4 and Notebook 2 Step 5)

Fill one row per table (from `DESCRIBE EXTENDED`):

| Table | Type | Provider | Location (path after `.dfs.core.windows.net/`) |
|---|---|---|---|
| `bronze.taxi_trips` | `EXTERNAL` | `delta` | `bronze/taxi_trips` |
| `bronze.weather_daily` | `EXTERNAL` | `delta` | `bronze/weather_daily` |
| `silver.taxi_trips` | `EXTERNAL` | `delta` | `silver/taxi_trips` |
| `silver.taxi_trips_rejected` | `EXTERNAL` | `delta` | `silver/taxi_trips_rejected` |
| `silver.weather_daily` | `EXTERNAL` | `delta` | `silver/weather_daily` |

List the files you find in `bronze/taxi_trips/_delta_log/` (first 3 names) and say what the `.json` files in that folder are for:

> - `00000000000000000000.json`
> - `00000000000000000001.json`
> - `00000000000000000002.json`

**Screenshots** (save in an `evidence/` folder next to this report and link them):

1. ADLS container `lakehouse` showing the folders `raw`, `bronze`, `silver`: `![adls](evidence/adls.png)`
2. Catalog Explorer showing the `bronze` and `silver` schemas with all 5 tables: `![catalog](evidence/catalog.png)`
3. Output of the check cells in both notebooks: `![checks](evidence/checks.png)`

**Why** is it safe to register these as *external* tables over the same container? What happens to the files in ADLS if someone runs `DROP TABLE` on an external table? (Do not test it.)

> Registering them as external tables separates storage from the metadata catalog. Each table points to its own distinct subfolder (`bronze/...` or `silver/...`), so they don't overwrite or interfere with one another inside the shared container.
>
> If someone runs `DROP TABLE` on an external table, **only the table definition in Unity Catalog is removed**. The actual Parquet and Delta log files sitting in ADLS remain completely untouched and safe, meaning you can easily re-register the table later without losing any data.

---

## §5 What I found when I profiled Bronze (Notebook 2, Step 1)

| Question | Your number |
|---|---|
| Total Bronze rows | 12,253,805 |
| Earliest pickup timestamp | 2009-01-01T12:52:15.000+00:00 |
| Latest pickup timestamp | 2025-11-01T00:32:12.000+00:00 |
| Pickups outside the month of their own file | 37 |
| Rows with dropoff ≤ pickup | 172980 |
| Rows with distance ≤ 0 | 355266 |
| Rows with fare ≤ 0 | 840442 |
| Null count of `passenger_count` | 2944316 |

Which columns share exactly the same large null count? `passenger_count`, `RatecodeID`, `store_and_fwd_flag`, `congestion_surcharge`, `Airport_fee`
Which `payment_type` value has that same row count? 0 What does that payment type mean (see Step 3 table)? `0` stands for Flex Fare

> **Why** do those columns all go null together? What does it tell you about those rows?
> _your answer_

---

## §6 Rejected rows and my decisions (Notebook 2, Step 2)

| `reject_reason` | Rows rejected | One example row (pickup, dropoff, distance, fare) | Why this row is not a real, usable trip |
|---|---|---|---|
| `pickup_outside_source_month` | 37 | 2009-01-01 12:52:15, 2009-01-01 13:12:15, 2.13, 18.4 | The trip's pickup timestamp falls outside the month |
| `dropoff_not_after_pickup` | 172980 | 2025-08-01 00:01:09, 2025-08-01 00:01:09, 0.0, 3.0 | The dropoff time is before or equal to pickup |
| `duration_over_6h` | 3490 | 2025-08-01 00:05:51, 2025-08-01 23:48:57, 1.71, 10.7 | Lasting over 6 hours |
| `distance_invalid` | 351499 | 2025-08-01 00:00:56, 2025-08-01 00:01:01, 0.0, 3.0 | A trip distance of zero or negative cannot represent as valid |
| `fare_invalid` | 766477 | 2025-08-01 00:00:03, 2025-08-01 00:13:11, 1.76, -4.75 | Negative or zero fares are invalid |
| **Total rejected** | 1294483 | | |

**Decision 1:** Millions of rows have a null `passenger_count`. We did **not** reject them. Why not? What would the business lose if we had?

> _your answer_

**Decision 2:** A row can break several rules at once, but gets one `reject_reason`. Take a row that has both `distance ≤ 0` and `fare ≤ 0`: which reason does it get, and why does the **order** of the rules change your rejected counts per reason but not the **total**?

> _your answer_

**Decision 3:** Why does the rejected table keep the *original* (uncleaned) values and Bronze column names?

> _your answer_

**Your own judgment:** pick **one** of the five rules and say if you would loosen or tighten it (for example the 6-hour limit) and what evidence you would want first.

> _your answer_

---

## §7 Silver design (Notebook 2, Steps 3–4)

Fill the before/after table for **seven** columns of your choice from `silver.taxi_trips` (must include `passenger_count`, a money column and `store_and_fwd_flag`):

| Bronze column and type | Silver column and type | What changed and why |
|---|---|---|
| `passenger_count` (`double`) | `integer` | Converted to a whole number since passenger counts can't be fractions; nulls are kept for app rides. |
| `fare_amount` (`double`) | `decimal(10,2)` | Changed to a decimal with 2 decimal places for accurate money math and to avoid rounding errors. |
| `store_and_fwd_flag` (`string`) | `string` | Cleaned and standardized to ensure consistent flags (like `Y` or `N`). |
| `trip_distance` (`double`) | `decimal(10,2)` | Converted to decimal for precise mileage and distance tracking. |
| `tpep_pickup_datetime` (`timestamp`) | `timestamp` | Kept as a timestamp so we can easily track when the trip started and group by date. |
| `RatecodeID` (`double`) | `integer` | Cast to an integer because rate codes represent whole-number categories. |
| `total_amount` (`double`) | `decimal(10,2)` | Converted to a decimal with 2 decimal places for accurate financial totals. |

Silver rows per month: Aug 3179661  Sep 3837040  Oct 3942621  
Silver rows with a NULL `passenger_count`: ______ (compare with the Bronze null count in §5 and explain the difference)

**Why** decimal for money and not double? Give one concrete example of what can go wrong with double.

> _your answer_

**Why** `passenger_count` outside 1–6 becomes NULL instead of rejecting the trip, while a fare ≤ 0 rejects it?

> _your answer_

**Why** units in the weather column names (`temp_max_c`, `precip_mm`, …)? What is `is_rainy` and which threshold did we use?

> _your answer_

Weather in Silver: rows ______, rainy days ______ (`is_rainy = true`), dry days ______.

---

## §8 Reconciliation: nothing disappears silently

| Bronze rows | Silver rows | Rejected rows | Duplicates removed | Bronze − (Silver + Rejected + Duplicates) |
|---|---|---|---|---|
| 12,253,805 | 10,959,322 | 1,294,483 | 0 | 0 |

Percentage of Bronze rows that were rejected: **10.56 %**

**Why** is "rows in = rows out + rows rejected" the first thing a data engineer checks after building a layer? Describe one real situation where it would have caught a bug.

> _your answer_

---

## §9 Does Silver join? (Notebook 2, Step 5)

Paste the SQL you wrote (rainy vs dry days, joined on date):

```sql
SELECT 
    w.is_rainy,
    COUNT(DISTINCT w.date) AS days,
    COUNT(t.trip_id) AS trips,
    ROUND(COUNT(t.trip_id) / COUNT(DISTINCT w.date), 0) AS trips_per_day
FROM silver.weather_daily w
LEFT JOIN silver.taxi_trips t 
    ON DATE(t.pickup_datetime) = w.date
GROUP BY w.is_rainy
ORDER BY w.is_rainy;
```

| is_rainy | Days | Trips | Trips per day |
|---|---|---|---|
| false | | 8268051 | 118115.01 |
| true | | 2691271 | 122330.5 |

In one sentence: what does this number *not* prove? (Hint: how many rainy days are there, and what else changes on those days?)

> _your answer_

---

## §10 Problems I hit

At least **two** real errors or surprises from your own run (not the ones in the hints).

| What I saw (paste the error line) | Why it happened | How I fixed it | Time lost |
|---|---|---|---|
| `LocalFilesystemAccessDeniedException: Cannot access non /Workspace local filesystem path: file:/tmp/...` | Databricks serverless blocks `dbutils.fs.cp` from reading temporary files written to the driver's local `/tmp/` disk, and `dbutils.fs.put` only accepts text, not binary Parquet bytes. | Wrote the DataFrame to a temporary folder in ADLS using `coalesce(1)`, copied the single generated `part-*.parquet` file to the final target path with `dbutils.fs.cp`, and deleted the temp folder. |  |
| `❌ bronze.taxi_trips is an EXTERNAL table` (8 of 10 checks passed) | Calling `.saveAsTable("bronze.taxi_trips")` without specifying a storage path created a managed table in Unity Catalog instead of an external table pointing to ADLS. | Added `.option("path", f"{BASE}/bronze/taxi_trips")` to the Delta write command and recreated the table, bringing the validation score to 10/10. | 15 mins |

---

## §11 Explain it back (no AI, from memory, one or two sentences each)

1. Bronze, Silver, Gold: what is the single job of **Silver**, in your words?
2. Why did Bronze keep the original column names and types, when it would have been easy to clean them there?
3. What is the difference between **rejecting** a row, **fixing** a value, and **keeping** a null? Give one example of each from this assignment.
4. If your Silver notebook crashes halfway through the write, what state is the Delta table in, and why is re-running it safe?
5. Your colleague drops the table `silver.taxi_trips`. Is the data gone? How do you bring the table back?
6. A new month (2025-11) arrives. List exactly what you would change or run to add it, in order.

> **1.** Silver's job is to clean up raw Bronze data, fix data types, and filter out bad rows so the data is trustworthy and easy to query or join.
> **2.** Bronze keeps a original copy of what arrived from the source because it easier to check what went wrong later or re-process the data if rules change.
> **3.** Rejecting throws out a bad row, like a trip with a zero or negative fare. Fixing changes how a value is stored without losing the row, like changing fare_amount from a double to a decimal for money. Keeping a null leaves an empty value alone when it makes sense, like keeping NULL in passenger_count for app-booked rides.
> **4.** Re-running it is safe because it cleanly overwrites the target data instead of duplicating it.
> **5.** The actual files in ADLS are still safe because it is an external table. The table can be bring back just by registering it again at that same ADLS path.
> **6.** Add 2025-11 to MONTHS, Run Notebook 1 to download the new file and load it into Bronze. Run Notebook 2 to filter out bad rows, update Silver, and check the counts.

---

## §12 How I used AI

| Question | Answer |
|---|---|
| Which AI tool(s) did you use? | |
| For which steps (list the notebook and step numbers)? | |
| One thing the AI got **wrong or that you had to change** (paste the original line and your fix) | |
| One line of AI-written code that you did not understand at first, and what it does | |
| One step you wrote yourself without AI, and why | |

> **Self-check before you submit:** close the AI tool. Pick any cell in your notebook. Can you explain every line and what breaks if you delete it? If not, go back to that cell.

---

## Submit

1. Both notebooks with output visible (run top to bottom).
2. This report, completely filled in, with the `evidence/` screenshots.
3. Commit everything: `git commit -m "assignment: adls bronze silver"` and share the link.

---

*Don’t focus on getting the answer. Focus on learning how to engineer the solution.*
**— Uma Kiran | DataKern | www.data-kern.com**
