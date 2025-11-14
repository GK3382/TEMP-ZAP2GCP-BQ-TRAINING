# BigQuery Pipelines — Training Guide
---

## 1. Executive Overview

**What it is:**

BigQuery Pipelines (within BigQuery Studio) lets you create, sequence, schedule, and run data workflows made of SQL queries, notebooks, and data preparation steps — all without leaving BigQuery Studio. It’s a lightweight orchestrator for SQL + notebook-based ETL/ELT that keeps work close to your data.

**Google Cloud Documentation:**
- [Pipelines Introduction](https://cloud.google.com/bigquery/docs/pipelines-introduction)
- [BigQuery](https://cloud.google.com/bigquery)

**YouTube Links:**
- [How to build a data pipeline with Google Cloud](https://www.youtube.com/watch?v=yVUXvabnMRU)

---

## 2. When to Use BigQuery Pipelines

Use pipelines when you need:

- Orchestrated, scheduled execution of SQL + notebooks inside BigQuery Studio
- A simple DAG-style workflow (task sequencing) without introducing external orchestrators (Airflow) for straightforward ELT or transformation flows
- Close integration with BigQuery datasets and lightweight automation for dashboards or downstream consumers

**Reference:** [Pipelines Introduction](https://cloud.google.com/bigquery/docs/pipelines-introduction)

---

## 3. Key Concepts & Components

- **Pipeline:** A sequence of code assets (SQL, notebook, data preparation) executed in order
- **Code assets:** SQL queries (GoogleSQL/standard SQL), Python notebooks (BigQuery Studio notebooks), and Dataform-based assets
- **Scheduling:** Built-in scheduler to run pipelines at configured intervals (cron-like)
- **Dataform:** Underpins pipeline management and versioning concepts inside BigQuery Studio; recommended for larger corp teams that need testing/version control

**References:**
- [Create Pipelines](https://cloud.google.com/bigquery/docs/create-pipelines)
- [Create Notebooks](https://cloud.google.com/bigquery/docs/create-notebooks)
- [Dataform](https://cloud.google.com/dataform)

---

## 4. Step-by-Step: Create Your First Pipeline (Simple SQL Pipeline)

This lab is minimal and suitable for a beginner.

**Precondition:** Access to BigQuery Studio and a project (no IAM instructions here).

**Steps:**
1. Open the Google Cloud Console → BigQuery (BigQuery Studio)
2. Create or pick a dataset that will contain your output table
3. Prepare a SQL query in the editor that writes to a destination table, for example:

```sql
CREATE OR REPLACE TABLE `project.dataset.daily_orders_summary` AS
SELECT
  DATE(order_ts) AS order_date,
  COUNT(*) AS orders,
  SUM(total_amount) AS revenue
FROM `project.dataset.raw_orders`
WHERE DATE(order_ts) = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY);
```

4. Save this SQL as an asset (**Save > Save query**)
5. From the editor tab bar: click the **+ (arrow)** and choose **Pipeline → Get started** to create a pipeline. Add your saved SQL query as the first pipeline step
6. Set the pipeline schedule (e.g., daily @ 02:00) and other settings. Save & run
7. Monitor the run using the pipeline runs page / Jobs page inside BigQuery Studio

**Reference:** [Create Pipelines](https://cloud.google.com/bigquery/docs/create-pipelines)

---

## 5. Step-by-Step: Notebook + SQL Pipeline (Example)

Use notebooks when you need richer processing (Python, pandas, custom libraries, ML prep).

**Steps:**
1. Create a notebook from a query result or new notebook (BigQuery → Run Query → Query results → Open in > Notebook)
2. In the notebook, write Python cells using the BigQuery client to run queries, transform data, and write back to BigQuery. Example snippet:

```python
from google.cloud import bigquery
client = bigquery.Client()

# run a query
job = client.query("""
SELECT customer_id, COUNT(*) as orders
FROM `project.dataset.raw_orders`
WHERE order_ts >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY customer_id
""")
df = job.to_dataframe()

# sample transform in pandas
df['flag_high_value'] = df['orders'] > 10

# load back to BigQuery
table_ref = client.dataset('dataset').table('monthly_customer_flag')
job = client.load_table_from_dataframe(df, table_ref, job_config=bigquery.LoadJobConfig(write_disposition="WRITE_TRUNCATE"))
job.result()
```

3. Save the notebook as an asset and add it to a pipeline as a step (notebook step). Schedule as needed

---

## 6. Orchestration Patterns & Tips

- **Linear pipelines:** Simple, straight sequence of steps (SQL → notebook → SQL). Good for simple ELT
- **Branching/conditional logic:** Not a native complex orchestration engine; for advanced branching, prefer using Dataflow/Airflow or Dataform + CI/CD. Use pipelines for mostly linear flows
- **Reusability:** Keep SQL queries modular (views or stored query assets) so multiple pipelines can reuse them

**Reference:** [Practical Intro & When to Use](https://medium.com/google-cloud/when-to-use-bigquery-pipelines-a-practical-introduction-2e1d4b2e92df)

---

## 7. Best Practices — Data & Query Design (Performance & Cost)

These are the highest impact operational recommendations:

- **Partition your tables** (date partitioning or ingestion-time) to reduce scanned data and enable partition pruning in MERGE/DML. When using MERGE, try to include the partition column in the predicate so only target partitions are scanned
- **Cluster frequently filtered/joined columns** to speed selective queries and reduce IO
- **Use appropriate data types** (INT64 for numeric keys rather than STRING) — comparisons on narrow types cost less and run faster
- **Denormalize when it helps** — BigQuery is columnar; nested & repeated fields (STRUCT/ARRAY) reduce joins and improve performance for many analytic queries
- **Avoid SELECT \*** in production queries; select only required columns to reduce bytes scanned
- **Use materialized views** for frequently-run, expensive aggregations to precompute results and save query cost

**References:**
- [Best Practices Performance Overview](https://cloud.google.com/bigquery/docs/best-practices-performance-overview)
- [Materialized Views Intro](https://cloud.google.com/bigquery/docs/materialized-views-intro)

---

## 8. Best Practices — Pipelines & Orchestration

- **Keep steps small & idempotent.** Smaller tasks are easier to retry/test and less likely to reprocess huge volumes
- **Use staging tables for upserts** — write new data to a staging table and MERGE into target (use partition pruning). See MERGE DML for atomic upserts
- **Version control code assets** (Dataform or GIT workflows). For teams, Dataform inside BigQuery Studio helps follow software engineering practices (tests, environments, CI)
- **Testing:** Write unit/integration checks (Dataform tests or queries that assert row counts / expected values)

**Reference:** [Dataform](https://cloud.google.com/dataform)

---

## 9. Incremental & Upsert Patterns (Practical)

MERGE is the canonical upsert in BigQuery: you can MERGE staging → target to insert new rows and update existing ones. For partitioned targets, include partition column in conditions to enable pruning and avoid scanning the whole table.

**Example pattern (staging → target upsert):**

```sql
MERGE `project.dataset.target_table` T
USING `project.dataset.staging_table` S
ON T.primary_key = S.primary_key
WHEN MATCHED THEN
  UPDATE SET column1 = S.column1, ...
WHEN NOT MATCHED THEN
  INSERT (primary_key, column1, ...) VALUES (S.primary_key, S.column1, ...);
```

---

## 10. Cost Controls & Pricing Considerations

- **Minimize bytes scanned:** Partitioning, clustering, projection (no SELECT *), and materialized views reduce cost
- **Choose pricing model:** On-demand vs flat-rate (reservations). BigQuery Reservations / slot management is useful for predictable heavy workloads. Check billing docs for your org

**Reference:** [BigQuery Pricing](https://cloud.google.com/bigquery/pricing)

---

## 11. Monitoring, Logging & Troubleshooting

- **Job history & run logs:** Use the BigQuery UI Jobs page to see query job details, bytes processed, job duration, and errors
- **Cloud Logging (Stackdriver):** Pipeline runs and notebook logs can be integrated with Cloud Logging for centralized alerting. (Search for “BigQuery job logs Cloud Logging”)

**Common failures & remedies:**

| Issue                                 | Remedy                                                                 |
|---------------------------------------|------------------------------------------------------------------------|
| Exceeded resources / timeouts         | Simplify query, break work into smaller steps, or use flat-rate slots  |
| MERGE scanned whole table             | Ensure partitioning and include the partition filter in MERGE condition|

---

## 12. Example Training Exercises (Suggested Curriculum)

**Module 1 — Fundamentals (hands-on)**
- Lab A: Run a simple query and save as SQL asset
- Lab B: Create and run a scheduled SQL pipeline that writes a daily summary table

**Module 2 — Notebooks (hands-on)**
- Lab C: Create a notebook that reads a BigQuery table, runs a pandas transform, writes results back as a table. Add it to a pipeline step

**Module 3 — Production patterns**
- Lab D: Implement staging + MERGE upsert into a partitioned table (show partition pruning)
- Lab E: Use materialized view for a common dashboard query and compare query bytes/time before/after

---

## 13. CI/CD, Testing & DataOps Recommendations

- Keep SQL/notebook code in Git
- Use Dataform or pipeline workspaces for environment separation
- Add automated tests that validate row counts, schema expectations, and sample data checks

**Reference:** [Dataform](https://cloud.google.com/dataform)

---

## 14. Additional Resources (Official + Recommended Reading)

**Official docs:**
- [Pipelines Introduction](https://cloud.google.com/bigquery/docs/pipelines-introduction)
- [Create Pipelines](https://cloud.google.com/bigquery/docs/create-pipelines)
- [Create Notebooks](https://cloud.google.com/bigquery/docs/create-notebooks)
- [Dataform](https://cloud.google.com/dataform)
- [Best Practices Performance Overview](https://cloud.google.com/bigquery/docs/best-practices-performance-overview)
- [Materialized Views Intro](https://cloud.google.com/bigquery/docs/materialized-views-intro)
- [Standard SQL DML Syntax](https://cloud.google.com/bigquery/docs/reference/standard-sql/dml-syntax)
- [BigQuery Pricing](https://cloud.google.com/bigquery/pricing)

**Helpful articles & posts:**
- [When to Use BigQuery Pipelines: A Practical Introduction](https://medium.com/google-cloud/when-to-use-bigquery-pipelines-a-practical-introduction-2e1d4b2e92df)
- [Performance Optimization with BigQuery](https://medium.com/google-cloud/performance-optimization-with-bigquery-821c8b648c84)
- [Google BigQuery ETL (Estuary)](https://estuary.dev/blog/google-bigquery-etl/)
- [BigQuery ETL (HevoData)](https://hevodata.com/blog/bigquery-etl/)