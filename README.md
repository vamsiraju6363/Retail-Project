# Retail Analytics ETL on AWS

A serverless data pipeline that takes raw retail CSVs and turns them into analytics-ready dashboards without anyone touching a button. Drop a file in S3 → Lambda wakes up → Glue transforms it → Athena queries it → QuickSight shows it. That's the whole flow.

Source data: the [Global Superstore](https://www.kaggle.com/datasets/apoorvaappz/global-super-store-dataset) dataset from Kaggle — orders, products, categories, sales, profits.

<p>
  <img alt="AWS" src="https://img.shields.io/badge/AWS-serverless-232F3E?logo=amazonaws&logoColor=white&style=flat-square">
  <img alt="PySpark" src="https://img.shields.io/badge/PySpark-3.5-E25A1C?logo=apachespark&logoColor=white&style=flat-square">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.10+-3776AB?logo=python&logoColor=white&style=flat-square">
  <img alt="Parquet" src="https://img.shields.io/badge/Parquet-columnar-50B8E7?style=flat-square">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green?style=flat-square">
</p>

---

## Architecture

```
┌──────────┐     S3 event     ┌──────────┐     starts     ┌───────────────┐
│   S3     │ ───────────────▶ │ Lambda   │ ─────────────▶ │ Glue (PySpark)│
│  (raw)   │                  └──────────┘                │    ETL job    │
└──────────┘                                              └──────┬────────┘
                                                                 │
                    Bronze → Silver → Gold                       │
                                                                 ▼
                                                        ┌─────────────────┐
                                                        │    S3 (gold)    │
                                                        └───────┬─────────┘
                                                                │
                                                                ▼
                                                        ┌─────────────────┐
                                                        │ Athena → QuickSight │
                                                        └─────────────────┘
```

### Layered data lake

| Layer | What lives here | Format |
|---|---|---|
| **Bronze** | Raw CSV exactly as uploaded. No schema applied yet. | CSV |
| **Silver** | Cleaned, deduped, typed. One row per order line. | Parquet |
| **Gold** | Business-ready aggregates: monthly revenue, profit margin, region splits. | Parquet |

## Why this design

- **Serverless end to end.** No clusters to babysit. The pipeline scales with data, not with headcount.
- **Idempotent.** Re-running a file produces the same gold output. Safe for backfills.
- **Cheap at rest.** Parquet + Athena means you pay per query, not per instance.
- **Observable.** Every Glue run logs row counts, timing, and job name so you can debug from CloudWatch.

## What's in this repo

```
AWS_Glue_Spark.py         Glue PySpark job: raw → silver → gold
test_spark.py             local test harness that mirrors the Glue logic
data/                     sample CSVs and intermediate outputs
notebooks/                exploration and validation notebooks
scripts/                  small utilities (bucket setup, uploads)
Images/                   architecture + dashboard screenshots
```

## Running it

### Locally (development loop)

```bash
git clone https://github.com/vamsiraju6363/Retail-Project.git
cd Retail-Project

python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# Run the pipeline against the sample CSVs
python test_spark.py
```

### On AWS (production)

1. Create three S3 prefixes (or buckets): `raw/`, `silver/`, `gold/`.
2. Upload `AWS_Glue_Spark.py` as a **Glue job** with a Python 3 / Spark 3 worker type.
3. Create a **Lambda** that starts the Glue job, triggered by S3 `ObjectCreated` events on `raw/`.
4. Register the gold prefix as an **Athena** table.
5. Connect **QuickSight** to Athena and build the dashboards.

Detailed steps (including IAM) are in `reports/deployment.md`.

## Dashboards

The QuickSight analysis covers:

- Total sales and profit (MTD, YTD)
- Profit margin trend by region
- Category share of revenue
- Top N customers by lifetime value
- Monthly revenue time series with YoY overlay

## What I'd add next

- **Streaming ingestion** via Kinesis for near-real-time dashboards.
- **Airflow or Step Functions** for orchestration across multiple source feeds.
- **dbt on Athena** to manage the gold-layer transformations as code.
- **Data quality checks** (Great Expectations) on the silver layer.

## License

MIT — see [LICENSE](LICENSE).

---

Built by [Vamshi](https://www.linkedin.com/in/vamsi-raju/).
