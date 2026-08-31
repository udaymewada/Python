# DDEA — Unity Catalog Governance & Security Deep Dive

*Condensed from detailed exam notes + additional related concepts for full coverage*

---

## Privileges & Security Hierarchy

- **To create schemas in a catalog** → principal needs `CREATE SCHEMA` privilege **on the catalog**.
- **Access hierarchy is cumulative**: to `SELECT` on a table, a user needs `SELECT` on the table **AND** `USE CATALOG` + `USE SCHEMA` on all parent objects. Revoking `USE SCHEMA` blocks access even if table-level `SELECT` still exists — the path to the object is broken.
- **Privileges are additive**: a `REVOKE` on an individual user only removes what was granted *directly* to that user. If the user still inherits the same privilege via group membership, they retain access. Revoking from a user ≠ revoking from their group.
- **DENY always wins**: an explicit `DENY` overrides any `GRANT`, from any source (direct or group-inherited). If a user's group is denied, the user is blocked — no exceptions.
- **`BROWSE` privilege**: lets a user discover an object and see its metadata (schema/table/column names via Catalog Explorer or `DESCRIBE`) **without** being able to query actual row data. Useful for data discovery without granting read access.
- **`READ FILES` privilege**: allows reading raw files directly from cloud storage paths governed by a UC **External Location**, for ad hoc path-based queries (e.g., querying JSON/CSV directly, not via a registered table).
- **`WRITE VOLUME` privilege**: required to upload, add, or modify files inside a UC **Volume**.
- **`APPLY TAG` privilege**: needed (along with table ownership) to modify a table's security metadata — e.g., attaching a column mask.

## Ownership

- **`ALTER TABLE <name> OWNER TO <principal>`** (or `SET OWNER TO`) — official way to transfer table ownership in UC.
  ```sql
  ALTER TABLE prod.core.customers OWNER TO data_eng_admins;
  ```
- **Materialized Views** run/refresh using the **identity and group memberships of the MV owner (definer)** — not the querying user. Important for security review: if the owner has broad access, the MV reflects that at refresh time regardless of who queries it.

## Row Filters & Column Masks (Table-Level)

- **Row filter**: SQL UDF returning `BOOLEAN`; rows where it returns `FALSE`/`NULL` are excluded. Applied via `ALTER TABLE ... SET ROW FILTER <udf> ON (<column>)`.
- **Multiple conditions in one filter**: UC supports **only one row filter function per table**. To filter on multiple attributes (e.g., region AND department), combine all logic into a **single UDF** using `AND`/`OR` — you cannot stack multiple row filter functions.
- **Multi-column row filters**: a UDF can take multiple columns as parameters (e.g., `contract_type`, `region`) mapped to the function's arguments for combined-condition filtering.
- **Column mask with extra context columns**: syntax passes the masked column implicitly as the first UDF argument, plus `USING COLUMNS (<other_column>)` for additional inputs.
  ```sql
  ALTER TABLE t ALTER COLUMN national_id
  SET MASK mask_udf USING COLUMNS (country);
  ```
- **Column masks affect the WHOLE query, not just the display**: if a mask returns `0` for a non-authorized user's `salary` column, that `0` is used everywhere in the query — including `WHERE` clauses. So `WHERE salary > 100000` becomes `WHERE 0 > 100000` (always false) → zero rows returned, not just masked display values.
- **Native column masks vs dynamic views**: native masks live directly on the base table column — users querying the table transparently see masked values, with **no separate view object** to manage/grant. Simpler governance than maintaining dynamic views.
- **Permissions to apply a column mask**: need `EXECUTE` on the masking UDF, **and** either table ownership or `APPLY TAG` privilege on the table.
- **Removing a row filter**: `ALTER TABLE ... DROP ROW FILTER` — detaches the filter; all users with `SELECT` then see all rows.
- **Inspecting applied policies**: `DESCRIBE EXTENDED` / `DESCRIBE TABLE EXTENDED` shows row filters (in table properties) and column masks (listed next to the specific column). `DESCRIBE GRANTS ON TABLE` shows only privilege grants (`SELECT`, `MODIFY`, `OWN`, etc.) — **not** row/column security policies. Different commands, different scope.

## ABAC (ties row filters/masks together at scale)

- **Recommended for consistent row filtering across many tables** — define access logic once as a SQL UDF, apply centrally via **governed tags**, rather than configuring per-table filters individually.
- Can incorporate **lookup/mapping tables** (e.g., a `region_managers` table) inside the UDF logic to determine dynamic permissions — more powerful than static per-table UDFs.

## Group-Membership Functions for Row-Level Security

| Function | Use Case |
|---|---|
| `is_member()` | Legacy — checks membership in **workspace-local groups** (non-UC or pre-UC environments) |
| `is_account_group_member()` | Modern UC governance — checks membership **only at the account level**; **cannot** verify workspace-local group membership |

**Rule of thumb:** legacy Hive Metastore / workspace-local groups → `is_member()`. Modern Unity Catalog / account-level groups → `is_account_group_member()`. Don't mix them up — each only sees its own scope.

## Managed vs External Tables & Cloning

- **Deep Clone**: creates a **full, independent copy of both data and transaction history**. Adding a `LOCATION` clause pointing to your own storage path makes the clone an **external table** at that path (vs a UC-managed default location).
- **`df.write.saveAsTable("table_name")`**: standard PySpark way to create a table. In UC, if no `.option("path", ...)` is given, the table defaults to **managed**.
- **Renaming a table** (managed or external): `ALTER TABLE <name> RENAME TO <new_name>`.
- **Delta `appendOnly` property**: setting `delta.appendOnly = true` blocks `UPDATE`/`DELETE` on the table — set via `ALTER TABLE ... SET TBLPROPERTIES ('delta.appendOnly' = 'true')`.

## Recovery: UNDROP TABLE

- **`UNDROP TABLE`** recovers managed tables, external tables, **and materialized views** within a **7-day retention window** (Databricks Runtime 12.2 LTS+).
- **Managed table drop lifecycle**: data isn't deleted immediately.
  1. Retained for **7 days** — recoverable via `UNDROP TABLE`.
  2. After 7 days, files are **permanently, asynchronously deleted within 48 hours**.

## Identities: Service Principals vs Human Users

| Identity | Use Case |
|---|---|
| **Service Principal** | Non-human identity for jobs/automation/production workloads — stable and auditable even if the human who set it up leaves the org. **Best practice for production jobs.** |
| **Workspace Admin** | Human-tied administrative role — **not suitable** as the running identity for automated/non-human production jobs (creates a fragile dependency on one person's account) |

- **Referencing a service principal by `applicationId` in SQL**: must be wrapped in backticks (`` ` ``) because the UUID contains hyphens, which SQL would otherwise interpret as special characters.

---

## Additional Related Concepts (Exam-Relevant Extensions)

### Table Constraints (beyond DLT expectations)
- `ALTER TABLE ... ADD CONSTRAINT <name> CHECK (<condition>)` — enforces a boolean condition on every row at write time; violating writes are rejected (unlike DLT `EXPECT`, which can just flag or drop).
- `NOT NULL` and `PRIMARY KEY`/`FOREIGN KEY` (informational only in Delta — not enforced, used for query optimization and documentation/lineage).

### Table Properties Worth Knowing
- `delta.autoOptimize.optimizeWrite` / `delta.autoOptimize.autoCompact` — auto-compact small files on write.
- `delta.logRetentionDuration` / `delta.deletedFileRetentionDuration` — control how long transaction log history / time-travel data is kept before `VACUUM` can clean it up.
- `delta.enableChangeDataFeed = true` — turns on **Change Data Feed (CDF)** so downstream consumers can read only inserted/updated/deleted rows via `table_changes()`.

### External Locations & Storage Credentials
- **Storage Credential**: UC object holding the cloud IAM role/service principal that has access to a cloud storage path.
- **External Location**: UC object combining a storage credential + a specific cloud path — the thing external tables and `READ FILES` privilege are actually governed against.
- Both are metastore-level objects, need their own `CREATE STORAGE CREDENTIAL` / `CREATE EXTERNAL LOCATION` privileges to set up.

### Lineage & Auditing
- Unity Catalog automatically captures **table & column-level lineage** across notebooks, jobs, dashboards, and pipelines — viewable in Catalog Explorer's "Lineage" tab.
- **Audit logs** (via system tables `system.access.audit`) record who did what (queries, grants, table drops) — key for compliance scenarios in exam questions about "who can prove X happened."

### Delta Sharing (cross-org governance)
- Lets you share **live** UC-governed data with other organizations/UC metastores **without copying data** — recipient queries the same underlying files via a secure sharing protocol.
- Shared objects can still carry row filters/column masks, but the **share owner must be exempt** from the policy for it to work through Open Sharing.

### Volumes vs Tables
- **Volumes**: UC-governed storage for **non-tabular files** (arbitrary files, models, images) — not queryable with SQL as rows/columns like tables are.
- Privileges: `READ VOLUME` (list/download files), `WRITE VOLUME` (upload/modify files) — separate from table-level `SELECT`/`MODIFY`.

---

*Pair with the full 7-section DDEA guide and the advanced concepts summary for complete exam prep coverage.*
