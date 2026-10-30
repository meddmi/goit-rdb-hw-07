# goit-rdb-hw-07

*Databases for ML*, homework 7: a reproducible pipeline for time-series features and semi-structured (JSONB) event data in PostgreSQL. Everything lives in one notebook, `hw_7.ipynb`, and runs on an embedded PostgreSQL 16 (`pgserver`), so no database install or credentials are needed.

## What the notebook does

1. **Setup and loading.** Starts PostgreSQL, loads a random 500,000-trip sample of the NYC TLC Yellow Taxi data for March 2024 (cleaned: fare 0-500, distance 0-100, trip duration under 24 h), builds the daily table `t7_hw_taxi_daily` (zone x day) and a typed `TIMESTAMPTZ` trips table.
2. **Time-series ML features** (window functions, every window `PARTITION BY pickup_zone_id`): lag 1/7/30 days, rolling 7-day average fare, rolling 30-day fare volatility, expanding cumulative trips, combined in the view `v_t7_hw_taxi_features`. Includes a caveat on row-based `LAG` for sparse zones.
3. **JSONB event log.** `t7_hw_events` with 50,000 events of 5 types and different payload shapes; queries with `@>`, `?` / `?|` and JSONPath.
4. **GIN and expression indexes.** `EXPLAIN (ANALYZE, BUFFERS)` before and after a `jsonb_path_ops` GIN index and a b-tree on `payload ->> 'device'`, plus index sizes, write cost, and extras on `jsonb_ops` vs `jsonb_path_ops` and the `@?` operator.
5. **JSONB vs normalized.** `t7_hw_checkouts_norm` and the same analytical query on both models, a MongoDB-equivalent pseudocode bonus, and a written Reflection.

## Run it

Requirements: Python 3.10 and internet access (the first run downloads about 50 MB of TLC data into the system temp directory and installs the Python packages listed in the first code cell).

```bash
python3.10 -m venv .venv
.venv/bin/python -m pip install jupyterlab ipykernel nbconvert
.venv/bin/jupyter lab hw_7.ipynb
```

To execute the whole notebook headlessly and refresh the saved outputs:

```bash
.venv/bin/python -m jupyter nbconvert --to notebook --execute --inplace hw_7.ipynb
```

Cells depend on earlier cells, so run them top to bottom. The database lives in `/tmp/hw7_pg` and is rebuilt by re-running the notebook.

## Results at a glance

Measured on one run (Apple silicon, PostgreSQL 16.2); exact numbers vary slightly between runs, the saved outputs in the notebook are the reference.

| Experiment | Before | After |
|---|---|---|
| `payload @> {'payment_method': 'card'}` (GIN `jsonb_path_ops`) | Seq Scan, 6.7 ms (23.8 ms with `jsonb_build_object`, which is re-evaluated per row) | Bitmap Index Scan + Bitmap Heap Scan, 2.9 ms |
| `payload ->> 'device' = 'mobile'` (b-tree on the expression) | Seq Scan, 5.1 ms (GIN does not help) | Bitmap Index Scan, 1.8 ms |
| Bulk insert of 50,000 rows | 41.8 ms without indexes | 113.8 ms with GIN (2.7x), 156.9 ms with GIN + b-tree |
| Checkout aggregate | JSONB: 6.4 ms, 1000 buffers | Normalized: 1.8 ms, 94 buffers |

The GIN index is 1144 kB, 14% of the 8000 kB table.
