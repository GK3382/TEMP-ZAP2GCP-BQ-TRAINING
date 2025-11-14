

# BigQuery

---

## 📚 Navigation

- [Home (README)](./README.md)
- [BigQuery Pipelines](./Bigquery%20Pipelines.md)
- [BigQuery Python Notebook](./Bigquery%20Python%20Notebook.md)
- [When to use BigQuery SQL Editor and BigQuery Python Notebook](./When%20to%20use%20BigQuery%20SQL%20Editor%20and%20BigQuery%20Python%20Notebook.md)

---


## 1. What is BigQuery?


**BigQuery** is Google Cloud’s fully managed, serverless data warehouse for running SQL analytics on very large datasets—without managing infrastructure. SQL (GoogleSQL) is used to query data; BigQuery automatically handles scaling.

- [Google Cloud Documentation](https://cloud.google.com/bigquery)
- [Official Docs Homepage](https://docs.cloud.google.com/bigquery/docs)

---

## 2. Key Concepts 

| Concept         | Description |
|-----------------|-------------|
| **Project**     | Top-level container in Google Cloud (your org/project). |
| **Dataset**     | Logical container inside a project used to group tables and views. Like a folder or database. Each table belongs to a dataset. [Docs](https://docs.cloud.google.com/bigquery/docs/datasets-intro) |
| **Table**       | Where actual rows/columns are stored. Types: native BigQuery tables, external tables (data kept in Cloud Storage), and views (saved queries). [Docs](https://docs.cloud.google.com/bigquery/docs/tables-intro) |
| **Partitioning**| Splitting a table by date (or integer range) so queries scan only relevant slices. |
| **Clustering**  | Ordering data by one or more columns to speed up queries that filter/join on those columns. [Docs](https://docs.cloud.google.com/bigquery/docs/clustered-tables) |


**Quick Analogy:**

> **Project** → **Dataset** (folder) → **Table** (spreadsheet)


**Useful Docs:**
- [Datasets Intro](https://docs.cloud.google.com/bigquery/docs/datasets-intro)
- [Tables Intro](https://docs.cloud.google.com/bigquery/docs/tables-intro)
- [Create Tables](https://docs.cloud.google.com/bigquery/docs/tables)

---

## 3. Step-by-Step: Common Tasks in the Cloud Console

### A. Open BigQuery in the Google Cloud Console
1. Sign into Google Cloud Console.
2. From the left menu, choose **BigQuery** (or search “BigQuery”).
	 - [Open BigQuery Console](https://console.cloud.google.com/bigquery)

### B. Navigate to a Dataset (a folder for tables)
1. In BigQuery left pane → select your project.
2. Click on **Dataset**.
### C. Create or Load a Table

**Options:**
- Upload CSV/JSON from your computer
- Load from Cloud Storage
- Create from query results
- Create an empty table and then load data

**Console Quick Flow — Upload CSV:**
1. Within dataset → **Create table**.
2. **Source:** Upload → select file.
3. **File format:** CSV/JSON/AVRO/Parquet etc.
4. **Destination:** Confirm dataset and table name.
5. **Schema:** Provide column names and types (or auto-detect for simple files).
6. **Advanced:** Choose Partitioning (date column or ingestion time) and Clustering columns if helpful.
7. Click **Create table**.
	 - [Create Tables](https://docs.cloud.google.com/bigquery/docs/tables)

### D. Run a Simple Query
1. Click **Compose new query**.
2. Example:
	 ```sql
	 SELECT date, product_id, SUM(amount) AS total
	 FROM yourdataset.yourtable
	 WHERE date BETWEEN '2025-01-01' AND '2025-01-31'
	 GROUP BY date, product_id
	 LIMIT 100;
	 ```
3. Click **Run**.
	 - *Tip: Avoid `SELECT *` when not needed — select only required columns.*

---


## 4. Basic Table Properties to Know (and Set When Creating Tables)

- **Table ID / Name:** Choose clear, descriptive names (e.g., `orders_2025`).
- **Schema:** List of columns and types (`STRING`, `INT64`, `FLOAT64`, `TIMESTAMP`, `DATE`, etc.). Keep types correct to avoid surprises.
- **Partitioning:** Choose a date column (or ingestion time) for large time-series tables — speeds queries and reduces scanned data.
- **Clustering:** Pick 1–3 high-cardinality columns used often in WHERE or JOIN clauses (e.g., `user_id`, `country`).
- **Table expiration / partition expiration:** Set to automatically remove old data (good housekeeping).
- **Description & labels:** Add short description and labels (e.g., `team:analytics`) to make tables easy to discover.


**Docs References:**
- [Datasets & Tables](https://docs.cloud.google.com/bigquery/docs/datasets-intro)
- [Tables](https://docs.cloud.google.com/bigquery/docs/tables-intro)

---


## 5. Simple, Practical Optimization Tips


*No code changes required beyond query writing!*

These are high-impact, easy rules to keep queries fast and efficient:



### Quick Optimization Rules

- **Filter early & narrow the time range**
	- Always include a date filter (or partition filter) in WHERE when possible so BigQuery scans fewer bytes.
	- Example: `WHERE date BETWEEN '2025-01-01' AND '2025-01-31'`
- **Select only the columns you need**
	- Don’t use `SELECT *`. Each column read increases data scanned.
- **Use partitioned tables for time-series data**
	- If your table has date/time, use partitioning so queries target fewer partitions.
- **Cluster on commonly filtered columns**
	- Clustering helps when you filter or group on those columns often.
- **Avoid cross-joining very large tables**
	- If you must join, reduce datasets before the join (pre-filter or aggregate).
- **Use materialized views for repeated aggregations**
	- If the same summary query runs often, a materialized view can be faster.
- **Use cached results**
	- If your query hasn’t changed and caching is enabled, reuse cached results (console shows cache usage).
- **Preview small samples**
	- When exploring data, use `LIMIT` and `ORDER BY`/`RAND()` to preview instead of scanning full table.
- **Prefer native formats like Parquet/Avro for bulk loads**
	- Those formats compress and read faster than CSV.
- **Leverage BigQuery’s recommendations**
	- BigQuery provides partitioning & clustering recommendations you can apply.


**Recommended Reading:**
- [BigQuery Performance Article — Google Cloud Blog](https://cloud.google.com/blog/products/data-analytics/new-bigquery-partitioning-and-clustering-recommendations)
- [Performance Optimization with BigQuery — Google Cloud / Medium](https://medium.com/google-cloud/performance-optimization-with-bigquery-821c8b648c84)

---


## 6. Maintenance & Housekeeping



### Weekly / Monthly Checklist (Simple):

- Keep table descriptions up to date (who owns it, what it contains).
- Set sensible table or partition expirations for data you don’t need forever (e.g., raw logs older than 90 days). This prevents clutter.
- Use clear naming conventions: `dataset.table_name` — include environment and date when helpful (e.g., `prod.events_v1`, `prod.orders_2025_01`).
- Remove or archive obsolete tables — if a table is outdated, archive to Cloud Storage (if needed) and delete from BigQuery.
- Use labels for discoverability (e.g., `team=marketing`, `retention=90d`).
- If you notice slow queries, add a partition/cluster recommendation request to your analytics team or apply BigQuery’s recommender in console.

---


## 7. “Do This / Don’t Do This” Cheat Sheet


### Do:
- Use partition filters (date ranges).
- Select only required columns.
- Use `LIMIT` for exploration.
- Name & describe tables.
- Use BigQuery recommended partitioning/clustering hints if available.


### Don’t:
- Run `SELECT *` on multi-GB/TB tables.
- Compare unfiltered huge tables without sampling.
- Keep infinite retention when data is not required.

---



## 8. Links & Reference Material (Plain Text for Copy/Paste)


### Official Docs (Primary Sources)
- [BigQuery](https://cloud.google.com/bigquery)
- [BigQuery Docs Index](https://docs.cloud.google.com/bigquery/docs)
- [Datasets Intro](https://docs.cloud.google.com/bigquery/docs/datasets-intro)
- [Tables Intro & Types](https://docs.cloud.google.com/bigquery/docs/tables-intro)
- [Create Tables](https://docs.cloud.google.com/bigquery/docs/tables)
- [Partitioned Tables & Clustering](https://docs.cloud.google.com/bigquery/docs/clustered-tables)
- [Manage Partition & Cluster Recommendations](https://docs.cloud.google.com/bigquery/docs/manage-partition-cluster-recommendations)


### Official Learning / Videos
- [Google Cloud YouTube Channel](https://www.youtube.com/googlecloudplatform)
- [BigQuery Workshop / Tutorials Playlist](https://www.youtube.com/playlist?list=PLA3TuOOaQOnnCB9ktgnY0OZb-fsU1K8nI)
- [Beginner BigQuery Tutorial — Example](https://www.youtube.com/watch?v=MH5M2Crn6Ag)


### Optimization / Practical Articles
- [Google Cloud Blog on Partitioning & Clustering](https://cloud.google.com/blog/products/data-analytics/new-bigquery-partitioning-and-clustering-recommendations)
- [Performance Optimization with BigQuery — Google Cloud / Medium](https://medium.com/google-cloud/performance-optimization-with-bigquery-821c8b648c84)