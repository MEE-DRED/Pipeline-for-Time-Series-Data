# Pipeline for Time-Series Data - Household Electricity Consumption

Formative 1 (Machine Learning Pipeline, Group 2). The group forecasts a household's **average active power one hour ahead**
(`Global_active_power`, kW) from the UCI *Individual household electric power consumption* data.

| Task | What | Where | Owner |
|---|---|---|---|
| 1 | EDA, preprocessing, lag / moving-average features, model + experiment table | `notebooks/`, `src/` | teammates |
| **2** | **Relational (MySQL) and MongoDB design, ERD, sample documents, queries + results** | `database/`, `docs/database_design.md` | Honette |
| 3 | CRUD + latest-record + date-range endpoints for both databases (FastAPI) | `api/` | Ketia |
| 4 | Prediction script: fetch from API → preprocess → load model → predict | _to be added_ | Ketia |

> Task 1 is documented in [`Task1README.md`](Task1README.md); Task 2 below; Task 3 in its own section further down.

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
api/              main.py  sql_routes.py  mongo_routes.py  schemas.py  db.py          <- Task 3 (FastAPI)
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

## Task 3 - CRUD and time-series API (`api/`)

FastAPI service over the databases built in Task 2. Both backends expose **the same routes and the same flat JSON record**,
so a client can switch between `/sql` and `/mongo` without changing its code.

```bash
uvicorn api.main:app --reload         # after load_sql / load_mongo; open http://127.0.0.1:8000/docs (Swagger UI)
```

| Method | Route (`{db}` = `sql` or `mongo`) | Purpose |
|---|---|---|
| POST | `/{db}/readings` | create an hourly reading (201; **409** if that hour exists; 422 on invalid values) |
| GET | `/{db}/readings?limit=&offset=` | list readings in time order |
| GET | `/{db}/readings/latest?valid_only=true` | **latest record** (`valid_only` skips incomplete hours such as the final one) |
| GET | `/{db}/readings/range?start=&end=&valid_only=` | **records by date range**, both ends inclusive, oldest first |
| GET | `/{db}/readings/{id}` | one reading (SQL: integer `reading_id`; Mongo: `ObjectId` string) |
| PUT | `/{db}/readings/{id}` | update the fields that are sent |
| DELETE | `/{db}/readings/{id}` | delete (SQL cascades to `circuit_energy`) |
| POST / GET | `/{db}/forecasts` | store / list predictions (used by Task 4) |
| GET | `/health` | reports whether MySQL and MongoDB are reachable |

Design notes
* **SQL**: a reading spans `hourly_readings` + `circuit_energy` (long format). Writes touch both inside one transaction;
  reads pivot the circuits back into columns. **Mongo**: the Task 2 nested layout (`measurements`, `sub_metering_wh`, `quality`, `calendar`).
* Derived fields follow the Task 1 rules: `is_valid_hour = n_obs_minutes >= 45`, and
  `other_energy_wh = global_active_power_kw*1000 - (sub_metering_1+2+3)` (recomputed on create/update unless sent explicitly).
* Validation: timestamps must be on the hour and time-zone free, values non-negative; duplicate hours → 409; database down → 503.
* Note: the pre-aggregated Mongo `daily_summaries` are not recomputed by the CRUD routes.

Example requests:
```bash
curl "http://127.0.0.1:8000/sql/readings/latest?valid_only=true"
curl "http://127.0.0.1:8000/mongo/readings/range?start=2010-11-26T00:00:00&end=2010-11-26T05:00:00"
curl -X POST http://127.0.0.1:8000/mongo/readings -H "Content-Type: application/json" \
     -d '{"timestamp":"2010-11-27T00:00:00","global_active_power_kw":1.25,"voltage_v":240.3,"global_intensity_a":5.2,"global_reactive_power_kvar":0.11,"sub_metering_1_wh":0,"sub_metering_2_wh":30,"sub_metering_3_wh":600}'
curl -X PUT http://127.0.0.1:8000/mongo/readings/<id> -H "Content-Type: application/json" -d '{"global_active_power_kw":1.4}'
curl -X DELETE http://127.0.0.1:8000/mongo/readings/<id>
```

## Team contributions

| Member | Main components |
|---|---|
| Honette Marie Ihozo | Task 2: ERD, SQL schema, MongoDB collection design, loaders, sample documents, queries and results |
| Mildred Ewenrim Ebomah | _fill in_ |
| Ketia Ineza Gakwaya | Task 3: FastAPI CRUD / time-series endpoints for MySQL and MongoDB |
