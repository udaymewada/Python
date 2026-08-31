# DDEA — Advanced Concepts Summary (Quick Reference)

*Condensed from detailed exam notes — ingestion, jobs, tuning, governance, CI/CD*

---

## Ingestion

- **COPY INTO `VALIDATE`**: `VALIDATE ALL` or `VALIDATE <n> ROWS` previews a load (checks parsing/schema errors) **without writing data**.
- **COPY INTO**: incremental, idempotent, retriable — skips already-loaded files.
- **Auto Loader**: most scalable ingestion method, built on Structured Streaming; handles billions of files; supports Python and SQL (Declarative Pipelines).
- **Auto Loader `addNewColumns`**: on detecting a new column, the stream fails with `UnknownFieldException` and auto-updates schema in `cloudFiles.schemaLocation`. Pair with a Job/Workflow retry policy so the stream auto-resumes with the new schema.
- **`cloudFiles.useNotifications = true`**: switches Auto Loader from directory listing to cloud event-based detection (S3 Event Notifications/SQS, Event Grid, Pub/Sub) — far more scalable/cheaper for large file volumes. Requires cloud permissions to create/manage these event resources.
- **CTAS with `read_files()`**: simple batch ingestion pattern for smaller/ad hoc datasets — `CREATE TABLE x AS SELECT * FROM read_files(path, format => '...')`.
- **No native connector, small payload (<1MB REST API)**: just write a custom Python script (`requests` library) in a notebook/Job — no need for a heavyweight connector.
- **Lakeflow Connect / Lakehouse Federation**: both use **Unity Catalog Connection objects** to centrally store/govern credentials and connection details for external systems.

## Lakeflow Jobs

- **`For each` task**: handles an empty array gracefully — completes successfully with zero iterations (empty list = valid input).
- **Table Update trigger**: only signals "source changed, start the job." It does **not** filter to new rows — use Structured Streaming (checkpoints) or **Delta Change Data Feed (CDF)** inside the job to process only changed rows.
- **Retry policy**: set at the task level to auto-retry transient failures (e.g., HTTP 502) — no code changes needed.
- **`max_concurrent_runs` reached, queueing off**: new run is immediately marked **Skipped** (never executes, not auto-retried). Enable **queueing** to hold runs up to 48 hours until capacity frees.
- **Dynamic Value References (`{{ }}`)**: runtime templating for adaptive workflows —
  - Job context: `{{job.start_time.day}}`, `{{job.run_id}}`, `{{job.parameters.environment}}`
  - Task context: `{{task.name}}`, `{{task.retry_count}}`
  - Inter-task: `{{tasks.<task-name>.values.<key>}}` — pass computed values/paths between tasks.

## Data Quality (Declarative Pipelines / DLT)

- `CONSTRAINT ... EXPECT (condition)` — flags violations, keeps rows (default).
- `... ON VIOLATION FAIL UPDATE` — fails the whole pipeline update on violation.
- `... ON VIOLATION DROP ROW` — silently drops the violating row.
- All three can coexist on one streaming table for layered data quality control.

## Delta Lake / Optimization

- **`ALTER TABLE ... CLUSTER BY`**: updates table metadata only — future writes use the new key, but **existing files are not reorganized** until `OPTIMIZE` runs (manually or via Predictive Optimization).
- **`spark.sql.autoBroadcastJoinThreshold`**: max byte size of a table Spark will auto-broadcast in a join.
- **Decreasing `spark.sql.shuffle.partitions`**: for small datasets, default 200 shuffle partitions creates too many tiny tasks (scheduling overhead > processing time) — lowering it consolidates work into fewer, more efficient tasks.
- **Photon**: Databricks' vectorized C++ query engine, API-compatible with Spark, accelerates SQL/DataFrame workloads via modern CPU instructions.
- **MERGE INTO**: standard upsert/delete pattern — `WHEN MATCHED ... UPDATE/DELETE`, `WHEN NOT MATCHED ... INSERT`, with conditional logic (e.g., by `status` flag).

## Cluster / Infra Troubleshooting

- **Bootstrap Timeout**: node (data plane) can't reach the control plane/cloud endpoints — usually missing NAT gateway, bad route tables, or firewall blocking outbound HTTPS (443).
- **File notification mode permissions**: Databricks needs cloud permissions to create/manage SNS/SQS (AWS), Event Grid/Queue Storage (Azure), or Pub/Sub (GCP) to subscribe to storage events.
- **High GC (Garbage Collection) time** (e.g., 80% of task time): strong signal of memory pressure — caused by insufficient executor memory, inefficient object allocation, or poor serialization.
- **Fleet instance types**: Databricks auto-picks from a pool of similar instance types in a size range — improves chance of getting compute when a specific instance type is unavailable in a region.
- **Serverless compute for Workflows**: no infra to provision/manage; Databricks handles scaling/security; much faster startup than classic compute.

## Governance (Unity Catalog)

- **`DESCRIBE EXTENDED` / `DESCRIBE TABLE EXTENDED`**: shows detailed metadata, including which **row filter** and **column mask** policies are applied and to which columns.
- **`WITH ROW FILTER`**: attaches a UDF (must return `BOOLEAN`) to a table; rows where it returns `FALSE`/`NULL` are filtered out.
- **`system.information_schema`** (and per-catalog `information_schema`): standardized SQL-queryable metadata — catalogs, schemas, tables, owners, creation timestamps — across the whole environment.
- **Governed storage for UC pipelines**: checkpoint/schema locations should live in a UC **external location** or **UC volume** — keeps state files secured/audited under UC access controls.

## CI/CD

- **`databricks bundle plan`**: previews what a bundle deployment will change (create/update/delete) — like `terraform plan` — without applying anything.
- **Databricks CLI v0.200+**: required for bundle deployments. Install via official curl script (Linux) or the `databricks/setup-cli` GitHub Action (CI/CD pipelines).
- **Git automation for workspace-level Git Folders**: use webhooks + the **Repos API** update endpoint to auto-pull latest changes into a production Git folder after a merge. Valid, but Databricks recommends **Declarative Automation Bundles (DABs)** for more complete CI/CD.

---

*Condensed reference — pair with the full 7-section DDEA study guide for complete exam coverage.*
