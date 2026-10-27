# Pipeline for Time-Series Data - Household Electricity Consumption

Formative 1 (Machine Learning Pipeline, Group 2). The group forecasts a household's **average active power one hour ahead**
(`Global_active_power`, kW) from the UCI *Individual household electric power consumption* data.

| Task | What | Where | Owner |
|---|---|---|---|
| 1 | EDA, preprocessing, lag / moving-average features, model + experiment table | `notebooks/`, `src/` | teammates |
| **2** | **Relational (MySQL) and MongoDB design, ERD, sample documents, queries + results** | `database/`, `docs/database_design.md` | Honette |
| 3 | CRUD + latest-record + date-range endpoints for both databases | _to be added_ | teammates |
| 4 | Prediction script: fetch from API → preprocess → load model → predict | _to be added_ | teammates |

> This README covers Task 2. Teammates: add your own sections for Tasks 1, 3 and 4 below.

## Repository layout

```
database/
  sql/            01_schema.sql  02_seed_reference_data.sql  03_queries.sql
  erd/            erd.mmd (source)  erd.png  erd.svg
  mongo/          queries.py (8 queries)  sample_documents.json
  scripts/        load_sql.py  load_mongo.py  run_sql_queries.py  run_mongo_queries.py  export_sample_documents.py
  results/        sql_query_results.md  mongo_query_results.md      <- queries + their results (for the report)
  config.py       connection settings (env vars / .env)
data/processed/   hourly_observed.csv   (hourly data produced by Task 1)
docs/             database_design.md    (design rationale, table dictionary, results summary)
docker-compose.yml  requirements.txt  .env.example
```

## Reproduce

```bash
python -m venv .venv && source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env                                     # edit passwords if you use your own servers
docker compose up -d                                     # MySQL 8 + MongoDB 7; or `podman compose up -d` (skip if you already run them)

python -m database.scripts.load_sql                      # creates power_db, loads 34,589 hours + 136,604 circuit rows
python -m database.scripts.load_mongo                    # creates collections/indexes/validators, loads 34,589 hourly documents
python -m database.scripts.run_sql_queries               # -> database/results/sql_query_results.md
python -m database.scripts.run_mongo_queries             # -> database/results/mongo_query_results.md
```
Both loaders are re-runnable (they drop and recreate the data).

## What the other tasks can rely on

* **Task 3 (API):** connection settings in `database/config.py` (reads `.env`). Read/write the `hourly_readings` table / collection.
  In SQL, sub-metering lives in `circuit_energy` (one row per circuit); the view `v_hourly_wide` returns it in the wide CSV layout.
  Uniqueness on (household, hour) means a duplicate POST raises a duplicate-key error → return 409.
* **Task 3 "latest record":** the final hour (2010-11-26 21:00) has only 3 observed minutes, so its values are NULL.
  Filter on `is_valid_hour = 1` (SQL) / `quality.is_valid_hour: true` (Mongo) when you need the latest usable hour.
* **Task 4 (prediction):** store predictions in `forecasts` (SQL) / `forecasts` collection (Mongo). The model is registered as
  experiment `E4` in `ml_models` / `models`; `actual_kw` can be filled in later to track error.

## Design summary
* **MySQL** (6 tables): `households` → `circuits` / `hourly_readings` → `circuit_energy`, plus `ml_models` → `forecasts`. 3NF, FKs, CHECKs, unique (household, hour).
* **MongoDB** (5 collections): one document per hour with embedded `measurements`, `sub_metering_wh`, `quality`, `calendar`; pre-aggregated `daily_summaries`; `$jsonSchema` validators and unique compound indexes.
* Full rationale, table dictionary and a summary of all 16 queries (8 per database): [`docs/database_design.md`](docs/database_design.md).

## Team contributions

| Member | Main components |
|---|---|
| Honette Marie Ihozo | Task 2: ERD, SQL schema, MongoDB collection design, loaders, sample documents, queries and results |
| Mildred Ewenrim Ebomah | _fill in_ |
| Ketia Ineza Gakwaya | _fill in_ |
