# Real-Time Stock Market Data Pipeline

*A Dockerized, near-real-time pipeline that streams stock quotes via Kafka, lands them in an S3-compatible bucket, loads to Snowflake, and transforms with dbt for analytics.*

![Architecture](screenshot/Architecture.jpeg)

---

## 1) Project Description
**What it is.** An end-to-end, production-style data pipeline that continuously ingests market quotes from a public API, buffers them through **Apache Kafka**, persists raw events in **S3/MinIO**, micro-batches them into **Snowflake**, and models **Bronze → Silver → Gold** tables with **dbt** for BI consumption (e.g., Power BI).

**Why it matters.** This design demonstrates real-time patterns (stream → land → warehouse → transform) with clear separation of concerns, reliability (idempotent loads + raw retention), and governance (tests + lineage). It’s fully containerized with **Docker**, orchestrated by **Apache Airflow**, and delivers minute-level SLAs suitable for dashboards or downstream ML.

**Tech:** Snowflake • dbt • Apache Airflow • Apache Kafka • Python • Docker • S3/MinIO

---

## 2) Data Source + Kafka Use
**Description.** A Python **producer** polls a stock-quotes API (e.g., Finnhub) for symbols such as `AAPL, MSFT, TSLA, GOOGL, AMZN` and publishes JSON to Kafka topic **`stock-quotes`**. Using Kafka provides durability, back-pressure handling, and decouples ingestion from downstream loading. A Python **consumer** subscribes to the topic and writes each message as a small JSON object to S3/MinIO. Partitioning by **`symbol`** allows parallelism and ordered consumption per symbol.

**Message shape (per event).**
```json
{
  "symbol": "AAPL",
  "c": 187.42,  "o": 185.90, "h": 188.10, "l": 184.75, "pc": 184.03,
  "d": 3.39,    "dp": 1.85,  "t": 1713456789
}
```

**Producer (excerpt).**
```python
from kafka import KafkaProducer
import requests, time, json, os

SYMBOLS = ["AAPL","MSFT","TSLA","GOOGL","AMZN"]
API = "https://finnhub.io/api/v1/quote"
KEY = os.getenv("STOCKS_API_KEY")

producer = KafkaProducer(
    bootstrap_servers="kafka:9092",
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
)

while True:
    for s in SYMBOLS:
        r = requests.get(API, params={"symbol": s, "token": KEY})
        q = r.json()
        producer.send("stock-quotes", {"symbol": s, **q})
    producer.flush()
    time.sleep(6)  # ~real-time cadence
```

---

## 3) S3 Bucket + Snowflake Source
**Description.** The consumer stores events in an **S3-compatible landing zone** (MinIO) like `s3://source-transaction/symbol=TSLA/1713456789.json`. This raw zone is audit-friendly and enables simple replays/backfills. **Airflow** then micro-batches: it lists new objects, downloads them, `PUT`s to a Snowflake **stage**, and runs `COPY INTO` a raw/bronze table with a single **VARIANT** column.

**Snowflake raw setup (excerpt).**
```sql
CREATE SCHEMA IF NOT EXISTS RAW;
CREATE TABLE IF NOT EXISTS RAW.SOURCE_STOCK_QUOTES (v VARIANT);
CREATE STAGE IF NOT EXISTS RAW.STG_STOCK_QUOTES;

COPY INTO RAW.SOURCE_STOCK_QUOTES
FROM @RAW.STG_STOCK_QUOTES
FILE_FORMAT = (TYPE = JSON)
ON_ERROR = CONTINUE;
```

**Why this approach.** Micro-batching is simple, robust, and cost-efficient. It provides predictable loads while keeping the option open to move to **Snowpipe Auto-Ingest / Snowpipe Streaming** for sub-minute latency.

---

## 4) dbt + Snowflake
**Description.** **dbt** turns JSON **VARIANT** into typed, documented models.  
- **Bronze**: parse raw JSON → typed columns; keep fidelity.  
- **Silver**: type normalization, dedupe, and consistent time semantics.  
- **Gold**: business-ready marts: *latest KPIs per symbol*, *OHLC candlesticks*, and *volatility/treemap* aggregates.  
Tests (`not_null`, `unique`) ensure trust in downstream analytics.

**Bronze model (excerpt).**
```sql
-- dbt_stocks/models/bronze_stg_stock.sql
WITH src AS (
  SELECT v FROM {{ source('raw','SOURCE_STOCK_QUOTES') }}
)
SELECT
  v:"symbol"::string              AS symbol,
  v:"c"::float                    AS price,
  v:"o"::float                    AS open_price,
  v:"h"::float                    AS high_price,
  v:"l"::float                    AS low_price,
  v:"pc"::float                   AS prev_close,
  v:"d"::float                    AS change_abs,
  v:"dp"::float                   AS change_pct,
  TO_TIMESTAMP_NTZ(v:"t"::number) AS ts_ingested
FROM src;
```

**Gold KPI (latest per symbol).**
```sql
WITH ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY symbol ORDER BY ts_ingested DESC) AS rn
  FROM {{ ref('silver_clean_stock') }}
)
SELECT *
FROM ranked
WHERE rn = 1;
```

**dbt DAG**
![dbt DAG](screenshot/DBT%20DAG.png)

---

## 5) Airflow + Docker
**Description.** **Airflow** orchestrates the end-to-end workflow on a cron of **every minute** (configurable):  
1) **List/Download** new JSON objects from S3/MinIO.  
2) **Load to Snowflake** via `PUT` + `COPY`.  
3) **(Optional)** run `dbt build` to refresh models after load.  
The entire stack (Kafka, Zookeeper, MinIO, Airflow, Postgres metadata) runs in **Docker** using `docker-compose`, enabling one-command spin-up and reproducible environments.

**DAG skeleton (excerpt).**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def load_to_snowflake(**_): ...
# downloads from MinIO → PUT → COPY INTO RAW.SOURCE_STOCK_QUOTES

with DAG(
    "stocks_realtime",
    start_date=datetime(2024,1,1),
    schedule_interval="* * * * *",
    catchup=False
) as dag:
    load = PythonOperator(task_id="load_snowflake", python_callable=load_to_snowflake)
    # Optionally: dbt_build = BashOperator(task_id="dbt_build", bash_command="cd dbt_stocks && dbt build")
    # load >> dbt_build
```

**Airflow DAG**
![Airflow DAG](screenshot/Airflow%20DAG.png)

**Why Airflow + Docker.** Airflow gives visibility, retries, SLAs, and alerting; Docker standardizes the stack across machines and makes CI/CD onboarding trivial.

---

## 6) Final Word
**Impact.** This repo showcases a clean, scalable template for **near-real-time analytics**: streaming ingestion, durable landing, governed transformations, and orchestrated delivery. It’s easy to extend (add symbols, tests, marts) and easy to deploy (compose up, set credentials, run).

**Next steps.**
- Switch MinIO → AWS S3 and enable **Snowpipe Auto-Ingest** or **Snowpipe Streaming** for sub-minute latency.  
- Add **dbt-expectations** for richer data quality.  
- Automate **Power BI** dataset refresh and add monitoring metrics (load counts/latency).

