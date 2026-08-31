# DDEA Exam Notes — Section 6: Troubleshooting, Monitoring & Optimization (10%)

*Databricks Certified Data Engineer Associate — Exam Guide (May 2026)*

---

## 1. Job Performance Trends (Lakeflow Jobs Run History)

- The **Run History** view shows every past execution of a job/task.
- Use it to compare **current run duration vs historical baseline** — spot jobs getting progressively slower (data growth, skew, resource contention).
- Look at: run duration, start/end time, retry attempts, per-task timing.

**Key idea:** Don't just check pass/fail — trend the *duration* over time to catch silent degradation before it becomes a failure.

---

## 2. Monitoring Pipeline Health (Lakeflow Jobs UI)

- **Job statuses**: Succeeded, Running, Failed, Skipped, Terminated.
- **DAG task graph**: visual map of task dependencies — lets you quickly spot which **upstream task blocked** everything downstream.
- Track **run times** and **failure rates** across runs to judge overall pipeline reliability.

**Key idea:** The DAG view is your first stop when a multi-task job fails — find the red node, not just the overall red job status.

---

## 3. Performance Bottlenecks (Spark UI)

| Bottleneck | What it is | How to spot it | How to fix it |
|---|---|---|---|
| **Data Skew** | One partition/key has far more data than others | Stages tab: Min/Median task time & shuffle read look normal, but **Max** is wildly higher (e.g., 400MB median vs 5GB max) | Enable **AQE skew join handling**, or manually **salt the key** before joining |
| **Shuffling** | Wide transformations (join, groupBy, distinct) move data across the cluster | High "Shuffle Read/Write" in Stages tab; long shuffle stages | Reduce unnecessary wide transforms; tune `spark.sql.shuffle.partitions`; use broadcast joins for small tables |
| **Disk Spill** | Data doesn't fit in executor memory, spills to disk | "Spill (Memory)" / "Spill (Disk)" metrics in Spark UI | Increase executor memory; repartition to smaller partitions; reduce data per task |

**Key exam pattern:** A question describing "most tasks fast, one task very slow, shuffle read wildly higher for one task" = **data skew**, not "just add more executors."

**Core tuning parameters to know:**
- `spark.sql.shuffle.partitions`
- `spark.default.parallelism`
- `spark.executor.memory` / `spark.driver.memory`
- `spark.sql.autoBroadcastJoinThreshold`

---

## 4. Liquid Clustering & Predictive Optimization

- **Liquid Clustering**
  - Modern replacement for manual partitioning / Z-ordering.
  - Clusters data **incrementally on write** — no need to pick a fixed partition column upfront.
  - Adapts automatically as query patterns change over time.

- **Predictive Optimization**
  - Databricks **automatically runs** `OPTIMIZE`, `VACUUM`, and clustering maintenance on Unity Catalog–managed tables.
  - Based on actual usage patterns — removes the need to manually schedule maintenance jobs.

**Key idea:** Both features reduce the manual tuning burden — good keywords: "automatic," "adaptive," "no manual partition choice."

---

## 5. Cluster & Environment Troubleshooting

| Issue | Common Cause | Fix / Diagnosis |
|---|---|---|
| **Cluster startup failure** | Cloud capacity limits, invalid instance type, permission/network misconfig | Check cluster event log for the specific error; verify instance type availability and IAM/network settings |
| **Library conflicts** | Mismatched versions between cluster-installed libraries and notebook-scoped (`%pip install`) libraries | Standardize library versions at cluster level; avoid mixing notebook-scoped and cluster-scoped installs |
| **Out-of-Memory (OOM)** | Skewed joins, oversized broadcast joins, too few partitions for data volume | Increase executor memory; adjust `spark.sql.autoBroadcastJoinThreshold`; repartition data |

---

## Quick Recall Cheat Sheet

- **Skew** → uneven task/shuffle metrics → salt keys or AQE.
- **Shuffle** → wide transforms → tune shuffle partitions / broadcast joins.
- **Spill** → not enough memory → more memory or smaller partitions.
- **Liquid Clustering** → auto-clustering, no fixed partition column.
- **Predictive Optimization** → auto `OPTIMIZE`/`VACUUM` on UC tables.
- **DAG view** → find blocked upstream task.
- **Run History** → compare against historical baseline, not just pass/fail.

---

*Source: Databricks Certified Data Engineer Associate Exam Guide, version as of May 4, 2026. See the full 7-section guide in `DDEA_Full_Exam_Guide.md`.*
