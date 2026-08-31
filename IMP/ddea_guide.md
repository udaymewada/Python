# Databricks Certified Data Engineer Associate (DDEA) — Full Study Guide

*Based on official exam guide, version as of May 4, 2026*
*45 scored MCQs · 90 minutes · Passing weight distributed across 7 sections below*

---

# Section 1: Databricks Intelligence Platform (6%)

### 1.1 Core Platform Components

- **Data Intelligence Platform**: Databricks' unified platform combining data warehousing, data engineering, streaming, ML, and BI on top of a lakehouse.
- **Delta Lake**: open-source storage layer that sits on top of cloud object storage (S3/ADLS/GCS) and adds:
  - **ACID transactions** — safe concurrent reads/writes.
  - **Schema enforcement & evolution** — reject bad writes, or allow controlled schema changes.
  - **Time travel** — query previous table versions (`VERSION AS OF`, `TIMESTAMP AS OF`).
  - **Unified batch + streaming** — same table can be read/written by both.
- **Unity Catalog (UC)**: centralized governance layer across all workspaces.
  - Three-level namespace: `catalog.schema.table`.
  - Manages access control, lineage, auditing, and data discovery in one place — not per-workspace like the legacy Hive metastore.

### 1.2 Compute Services

| Compute Type | Best For | Notes |
|---|---|---|
| **All-Purpose Cluster** | Interactive dev/notebooks, ad hoc exploration | Manually created, can be shared by multiple users, billed while running |
| **Job Cluster** | Scheduled/automated ETL jobs | Spun up for a job run and terminated after — cheaper for scheduled workloads, no idle cost |
| **High-Concurrency / Serverless SQL Warehouses** | Many simultaneous BI/SQL users | Fast startup, autoscaling, good for concurrent ad hoc SQL |
| **Single Node** | Lightweight dev/testing, small ML workloads | No worker nodes, everything runs on driver |
| **Serverless Compute** (jobs/notebooks/SQL) | Fast startup, no cluster management | Databricks manages infra elasticity; good default for cost + simplicity when available |

**Selection logic the exam tests:** match workload pattern (scheduled vs interactive vs concurrent BI) → cheapest/fastest-fitting compute type. E.g., many analysts running ad hoc SQL all day → high-concurrency (or serverless) cluster with autoscaling, not a fixed all-purpose cluster.

---

# Section 2: Data Ingestion and Loading (21%)

### 2.1 Ingestion Patterns
- **Batch**: scheduled, bounded loads (e.g., nightly file drop).
- **Streaming**: continuous, unbounded ingestion (Structured Streaming).
- **Incremental**: only new/changed data is processed each run (avoids full reprocessing).

### 2.2 COPY INTO
- SQL command to **incrementally** load files from cloud object storage (S3/ADLS/GCS) into a Delta/UC table.
- Idempotent — automatically tracks which files were already loaded, so re-running is safe.
- Good for simple, scheduled, file-based batch loads where schema is fairly stable.

### 2.3 Auto Loader
- Incrementally and efficiently processes **new files as they arrive** in cloud storage.
- Two file-discovery modes:
  - **Directory listing** — lists directory contents to find new files (simpler, default for smaller volumes).
  - **File notification** — uses cloud provider event notifications (e.g., S3 notifications) to detect new files (scales better for large/high-volume directories).
- Supports **schema inference**, **schema enforcement**, and **schema evolution** (adapts as new columns appear, controlled via `mergeSchema` / rescue column settings).
- Preferred over COPY INTO when file volume is large or schema changes frequently.

### 2.4 Lakeflow Connect
- **Standard connectors**: simpler, common connectors for typical enterprise sources.
- **Managed connectors**: fully managed ingestion for SaaS/enterprise sources (e.g., Salesforce, Workday) directly into UC-governed Delta tables, with less setup/maintenance.
- Best choice when ingesting from **diverse enterprise SaaS sources** where you want managed reliability without hand-building pipelines.

### 2.5 JDBC/ODBC & REST Clients
- Used in notebooks to pull data from external databases/APIs into cloud storage or directly into UC tables.
- Typically orchestrated/scheduled via **Lakeflow Jobs**.

### 2.6 Choosing the Right Ingestion Method

| Method | Use When |
|---|---|
| **COPY INTO** | Simple batch file loads, stable schema, don't need advanced streaming semantics |
| **Auto Loader** | High file volume, frequent new files, need schema evolution, near-real time |
| **Lakeflow Connect (standard)** | Common enterprise data sources with built-in connector support |
| **Lakeflow Connect (managed)** | SaaS platforms where you want Databricks to manage the full ingestion lifecycle |
| **JDBC/ODBC/REST + Jobs** | Custom/legacy sources with no native connector |
| **Partner Connect** | Third-party ETL tool integration |

Decision factors tested: **data volume, ingestion frequency, data type (structured/semi/unstructured), governance need**.

### 2.7 Semi-Structured / Unstructured Data
- JSON and nested data ingested via Lakeflow Connect / Auto Loader into UC-governed Delta tables.
- Use `explode()` and dot notation to flatten nested structures during transformation (see Section 3).

---

# Section 3: Data Transformation and Modeling (22%) — *Largest section*

### 3.1 Data Cleaning (Bronze → Silver)
- Read raw **bronze** tables → clean nulls (`dropna`, `fillna`) → standardize data types (`cast`) → write to **silver** tables.
- This is the classic **medallion architecture** pattern (see 3.6).

### 3.2 Joins & Combining DataFrames
| Join Type | Behavior |
|---|---|
| **Inner join** | Only matching rows from both sides |
| **Left join** | All left rows + matches from right (nulls if no match) |
| **Broadcast join** | Small table is broadcast to all executors — avoids shuffle, big performance win for small-dim × large-fact joins |
| **Cross join** | Cartesian product of both tables — use with caution (expensive) |
| **Union / Union All** | Stack rows from two DataFrames with same schema; `union` in Spark SQL/DataFrame API does **not** dedupe by default (unlike SQL's traditional UNION) — use `.distinct()` if you need de-duplication |
| **Multiple keys join** | Join condition on more than one column |

**Exam tip:** broadcast joins are triggered automatically when a table is under `spark.sql.autoBroadcastJoinThreshold` (default 10MB) — know this parameter (see 3.5).

### 3.3 Manipulating Columns, Rows, Tables
- Add/drop columns: `withColumn()`, `drop()`.
- Rename: `withColumnRenamed()`.
- Split a column: `split()` (e.g., splitting a full name into first/last).
- Filter rows: `filter()` / `where()`.
- Explode arrays: `explode()` — turns array elements into separate rows (critical for flattening nested JSON).

### 3.4 Deduplication & Aggregation
- Deduplication: `dropDuplicates()` (optionally on specific columns).
- Aggregations: `count()`, `approx_count_distinct()` (faster, approximate — trades a little accuracy for speed on huge datasets), `mean()`, `summary()`/`describe()`.

### 3.5 Spark Tuning Parameters

| Parameter | What it Controls |
|---|---|
| `spark.sql.shuffle.partitions` | Number of partitions used after a shuffle (default 200) — too high = many small tasks/overhead; too low = large partitions/skew risk |
| `spark.default.parallelism` | Default number of partitions for RDD operations |
| `spark.executor.memory` / `spark.driver.memory` | Memory allocated per executor/driver — raise to reduce spill/OOM |
| `spark.sql.autoBroadcastJoinThreshold` | Max size of a table Spark will automatically broadcast in a join (default 10MB); set to `-1` to disable auto-broadcast |

**Exam approach:** after any tuning change, **re-measure performance** — the exam guide explicitly calls this out; tuning is iterative, not "set and forget."

### 3.6 Medallion Architecture & Gold Layer Objects
- **Bronze**: raw ingested data, minimal transformation.
- **Silver**: cleaned, conformed, deduplicated data.
- **Gold**: business-level aggregates for BI/analytics.

| Gold Object | Behavior |
|---|---|
| **View** | Virtual, computed at query time, always reflects latest underlying data, no storage cost |
| **Materialized View** | Precomputed and stored results, refreshed automatically/incrementally — faster reads, some staleness possible between refreshes |
| **Streaming Table** | Continuously updated table fed by a streaming source, ideal for near-real-time gold layers |
| **Table** | Standard Delta table for stable BI consumption |

### 3.7 Data Quality Checks
- Apply validation rules (e.g., expectations, constraints, `CHECK` constraints, or Lakeflow Declarative Pipelines' expectations) to ensure Silver/Gold data reliability.
- Typical checks: not-null, uniqueness, value-range, referential checks.

---

# Section 4: Working with Lakeflow Jobs (16%)

### 4.1 Control Flows
- **Retries**: automatically re-run a failed task N times before marking it failed.
- **Conditional tasks**: branching (run task B only if task A succeeds/fails) and looping (for-each style task iteration over a list).

### 4.2 Task Types & Dependencies
- **Notebook task, SQL query task, Dashboard task, Pipeline task** — the common building blocks.
- Dependencies are defined via a **DAG (Directed Acyclic Graph)** — task B can be set to run only after task A completes.

### 4.3 Triggers
| Trigger Type | Behavior | Best For |
|---|---|---|
| **Scheduled** | Cron-like time-based trigger | Predictable, regular batch jobs |
| **File arrival** | Fires when a new file lands in a specified storage location | Event-driven ingestion pipelines |
| **Table update** | Fires when an upstream Delta table changes | Data-driven pipelines chained to upstream tables |

**Exam logic:** choose **time-based** triggers when data arrival is predictable/regular; choose **data-driven** (file arrival/table update) triggers when pipeline execution should depend on actual data availability rather than a fixed clock.

---

# Section 5: Implementing CI/CD (10%)

### 5.1 Databricks Git Folders (formerly Repos)
- Native Git integration inside the workspace UI.
- Create/switch **branches**, commit, push, and open **pull requests** without leaving Databricks.
- Enables standard SDLC practices (feature branches → PR → merge → deploy) directly on notebooks/code.

### 5.2 Databricks Asset Bundles (DABs) — now **Declarative Automation Bundles**
- YAML-based way to **package, configure, and version** Databricks resources (Jobs, Pipelines, notebooks, etc.) as code.
- **Variables & targets/overrides**: define environment-specific config (dev/test/prod) once, override just what differs per environment — same codebase promoted across environments without duplicating logic.
- Deployed and validated via the **Databricks CLI** (`databricks bundle deploy`, `databricks bundle validate`).

### 5.3 CI/CD Workflow (typical exam scenario)
1. Develop in a Git branch inside Databricks Git Folders.
2. Push code, open PR, get it merged.
3. Define resources (Jobs, Pipelines) in a Bundle (DAB) with environment-specific variables.
4. Use Databricks CLI in a CI/CD pipeline (GitHub Actions, Azure DevOps, etc.) to `validate` and `deploy` the bundle to dev → test → prod targets.

**Exam tip:** the "right answer" for modular, versioned, repeatable ETL deployment is almost always **DABs + Git**, not ad hoc notebook triggering, model registries, or manually managed wheel files.

---

# Section 6: Troubleshooting, Monitoring, and Optimization (10%)

*(See detailed companion file: `DDEA_Section6_Troubleshooting_Monitoring_Optimization.md`)*

### 6.1 Job Performance Trends
- Use **Lakeflow Jobs Run History** to compare current run durations against historical baselines — catch silent performance degradation early.

### 6.2 Monitoring Pipeline Health
- Job statuses: Succeeded / Running / Failed / Skipped / Terminated.
- **DAG task graph** in the Jobs UI — visually spot which upstream task is blocking downstream tasks.
- Track run times and failure rates over time.

### 6.3 Spark UI Bottlenecks

| Bottleneck | Signature in Spark UI | Fix |
|---|---|---|
| **Data Skew** | Most tasks fast, one/few tasks very slow; Max shuffle read/write far exceeds Min/Median | Enable **AQE skew join handling**; salt the join key manually |
| **Shuffling** | High "Shuffle Read/Write" bytes in Stages tab from wide transforms (join/groupBy) | Tune `spark.sql.shuffle.partitions`; prefer broadcast joins for small tables |
| **Disk Spill** | "Spill (Memory)" / "Spill (Disk)" metrics appear in Stages tab | Increase executor memory; repartition to reduce per-task data size |

### 6.4 Liquid Clustering & Predictive Optimization
- **Liquid Clustering**: incremental, adaptive data clustering on write — no fixed partition column needed, adapts as query patterns evolve, replaces manual partitioning/Z-ordering.
- **Predictive Optimization**: Databricks automatically schedules and runs `OPTIMIZE`/`VACUUM` and clustering maintenance on UC-managed tables based on real usage — removes manual maintenance job scheduling.

### 6.5 Cluster/Environment Issues
| Issue | Cause | Fix |
|---|---|---|
| Cluster startup failure | Cloud capacity limits, bad instance type, network/permission misconfig | Check cluster event log; verify instance availability & IAM/network config |
| Library conflicts | Mismatch between cluster-installed vs notebook-scoped (`%pip install`) libraries | Standardize library management at one level (cluster or notebook, not mixed) |
| Out-of-Memory (OOM) | Skewed joins, oversized broadcasts, too few partitions | Increase executor memory, adjust `autoBroadcastJoinThreshold`, repartition |

---

# Section 7: Governance and Security (15%)

### 7.1 Managed vs External Tables (Unity Catalog)
| Type | Storage | Behavior |
|---|---|---|
| **Managed table** | Databricks manages both metadata AND underlying data files in the UC-managed storage location | Dropping the table deletes the underlying data too |
| **External table** | Databricks manages metadata only; data lives in a customer-specified cloud storage path | Dropping the table leaves the underlying data files intact |

- You can **convert** between managed and external, and perform standard CRUD (`CREATE`, `ALTER`, `DROP`) via SQL or UI.

### 7.2 Access Controls: GRANT / REVOKE / DENY
- Applied to **principals**: users, groups, service principals.
- Applied at appropriate levels of the **security hierarchy**: account → metastore → catalog → schema → table/view/column.
- **GRANT** — give a privilege (e.g., `SELECT`, `MODIFY`, `CREATE TABLE`).
- **REVOKE** — remove a previously granted privilege.
- **DENY** — explicitly block a privilege even if granted elsewhere (overrides inherited grants) — useful for exceptions within a broader grant.

### 7.3 Row-Level Security & Column Masking (table-level)
- **Row filters**: SQL UDF applied to a table that filters which rows a user sees at query time — configured per table, managed by table owner (`ALTER TABLE ... SET ROW FILTER`).
- **Column masks**: SQL UDF that masks/redacts a column's value based on the querying user's group membership (`ALTER TABLE ... ALTER COLUMN ... SET MASK`).
- Both are configured **per table** — fine for a small number of tables, but each must be set up individually.

### 7.4 ABAC (Attribute-Based Access Control) Policies
- The **recommended, scalable** approach vs table-level row filters/column masks (GA as of 2026).
- Policies attach at the **catalog, schema, or table level** and apply automatically based on **governed tags** — no per-table configuration needed.
- Benefits over table-level controls:
  - Consistent rules across many tables at once.
  - **Separation of duties**: policy authors define rules; data stewards just apply tags — no need to touch every table.
  - **Automatic coverage**: new tables get protected automatically once tagged, without manual setup.
- A catalog-level ABAC policy **cannot be bypassed or removed by individual table owners** — enables org-wide enforcement from higher-level admins.
- Requires **governed tags** (account-level, access-controlled tag definitions) — not arbitrary/ungoverned tags.
- Compute requirement: serverless, or standard/dedicated compute on Databricks Runtime 16.4+.

**Exam logic:** if a scenario describes needing the *same* rule across *many* tables, automatic protection of *new* tables, or *separation of duties* between rule-writers and data owners → answer is **ABAC**, not table-level row filters/masks.

---

# Cross-Cutting Exam Tips

- Questions are **scenario-based**, not definition-recall — expect a business situation + "which config/approach solves this."
- Watch for **distractor answers that sound technical but solve the wrong problem** (e.g., "add more executors" for a skew problem — skew isn't fixed by more compute, it's fixed by redistributing the work).
- Terminology has been renamed recently — know both old and new names:
  - Databricks Repos → **Databricks Git Folders**
  - Databricks Asset Bundles (DABs) → **Declarative Automation Bundles**
  - Delta Live Tables (DLT) → **Lakeflow Spark Declarative Pipelines**
  - Databricks Workflows → **Lakeflow Jobs**
- Always tie a fix back to root cause: skew → redistribute keys; spill → memory/partition size; shuffle → reduce wide transforms or tune partitions; OOM → memory/broadcast threshold.

---

*Source: Databricks Certified Data Engineer Associate Exam Guide (version as of May 4, 2026) + Databricks documentation on Unity Catalog ABAC (2026).*
