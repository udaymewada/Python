* In Databricks Unity Catalog, to allow a principal to create schemas within a catalog, you must grant the specific CREATE SCHEMA privilege on that catalog. This is the standard privilege for this action in the security hierarchy.
* ***A DEEP CLON***E creates a full, independent copy of both the data and the transaction history of the source table. By providing a LOCATION clause pointing to an S3 bucket, the resulting table is created as an external table at that specific path, satisfying all the requirements.
* In Databricks Unity Catalog, to apply a column mask, the user must have the ***EXECUTE privilege*** on the User Defined Function (UDF) to call it, and must either be the owner of the table or have the APPLY TAG privilege on the table to modify its security metadata.
* ***when a column mask is applied***, the masking function determines the value that is used for all parts of the query, including the WHERE clause. For a non-HR user, the salary column effectively becomes 0 throughout the query's execution. Since the expression 0 > 100000 is always false, the query will return no results.
* ABAC is the recommended approach for consistent row filtering across many tables.*** It allows defining access rules once using SQL UDFs and applying them centrally via governed tags.*** This method can incorporate logic that references mapping tables like region_managers to determine user permissions, avoiding the limitations of direct row filter UDFs.
* This is the standard syntax for applying a column mask using a UDF that requires additional columns. The national_id column is implicitly passed as the first argument to the UDF, and the USING COLUMNS (country) clause passes the country column as an additional argument. This approach is documented for dynamic data masking in Unity Catalog.
* ```
  ```
* In Databricks SQL and Unity Catalog, privileges are additive. A REVOKE command only removes a privilege that was previously granted directly to the specified principal. Since Bob is a member of the 'reporting' group, and that group still has the SELECT privilege, Bob continues to inherit that privilege. A REVOKE from the individual user does not override the privilege inherited through group membership.
* The DESCRIBE EXTENDED command (or its synonym DESCRIBE TABLE EXTENDED) provides detailed metadata about the table. In Unity Catalog, this command is used to inspect table properties, including applied row filters and column masking functions. Row filters typically appear in the table properties section, while masking functions are listed alongside the specific column definitions.
* In Databricks (and Unity Catalog), an explicit DENY always overrides any GRANT. Since Alice is a member of the contractors group, which has been explicitly denied SELECT access, her attempt to query the table will be rejected.
* ```ALTER TABLE name OWNER TO group and ALTER TABLE ... SET OWNER TO ...``` are valid syntaxes, command is the standard and officially documented way to transfer ownership of a table in Unity Catalog. ```ALTER TABLE prod.core.customers OWNER TO data_eng_admins;```
*  In Unity Catalog, Materialized Views are refreshed using the identity and group memberships of the Materialized View owner (the definer)
*  df.write.saveAsTable("table_name") is the standard PySpark API for creating a table in a metastore. In Unity Catalog, when no external path is specified via .option("path", ...), the table is created as a managed table by default.
*  n Databricks Unity Catalog, the ```ALTER TABLE ... DROP ROW FILTER``` syntax is the standard DDL command used to remove an existing row filter association from a table. Once executed, the filter is detached, and all users with SELECT permissions can view all rows in the table
*  In Delta Lake, setting the table property delta.appendOnly to true is the standard way to prevent UPDATE and DELETE operations. The correct SQL syntax for modifying table-level metadata is ALTER TABLE ... SET TBLPROPERTIES
*  Databricks Unity Catalog row filters can accept multiple input columns as parameters. The UDF is defined with parameters for each required column (e.g., contract_type and region) and is applied to the table by mapping these columns to the function arguments. This allows complex filtering logic based on the combined values of multiple fields.
*  ***is_member()*** is the built-in Databricks SQL function that checks whether the current session user is a direct or indirect member of a specified workspace-local group. While Databricks recommends migrating to account groups and using is_account_group_member() for Unity Catalog-enabled scenarios, in this workspace without Unity Catalog, workspace-local groups are still in use and is_member() is the correct function to enforce row-level security based on such groups.
*  The ***UNDROP TABLE*** command in Unity Catalog can recover both managed and external tables (as well as materialized views) within a 7-day retention period. This is the officially recommended approach for recovering accidentally dropped tables, as documented in Databricks SQL and Databricks Runtime 12.2 LTS and above.
*  A ***service principal is a non-human identity*** created in Databricks for use with automated tools, jobs, and applications. It is the recommended best practice for production workloads because it is not tied to an individual user's account, ensuring that jobs remain stable and auditable even if a human user leaves the organization
*  ***A workspace admin is an administrative role assigned to human users or groups***. It is tied to a human user's identity and specific permissions within a workspace, making it unsuitable and insecure as a principal for automated, non-human production jobs.
*  You should use is_account_group_member() when working with modern Unity Catalog governance, and is_member() primarily when maintaining legacy Hive Metastore environments
*  Native column masks are defined directly on the base table column, so users querying the table automatically see masked data without needing access to a separate view object. This simplifies governance by eliminating the need to manage and grant permissions on dynamic views
*  In Databricks SQL, renaming an existing table (whether managed or external) is performed using the ALTER TABLE <table_name> RENAME TO <new_table_name>; syntax. This is the standard command for modifying the metadata to reflect a new table name.
*  In Unity Catalog, uploading, adding, or modifying files within a volume requires the WRITE VOLUME privilege
*  when a service principal is referenced by its applicationId in a SQL statement, the ID must be enclosed in backticks (``). This is because the UUID format contains hyphens, which are special characters that need to be escaped.
*  hen a Unity Catalog managed table is dropped, the underlying data files are not immediately removed. Instead, they are retained for a default recovery period of 7 days, during which the table can be restored with UNDROP TABLE. After that grace period, the data files are permanently and asynchronously deleted within 48 hours, as documented in the official Databricks documentation for Unity Catalog.
*  he BROWSE privilege in Unity Catalog allows users to discover an object and inspect its metadata (such as schema names, table names, and column-level information) in the Catalog Explorer or using SQL commands like DESCRIBE, without granting the ability to query the actual data rows.
*  ***The DESCRIBE GRANTS ON TABLE command*** is used to display the privileges and permissions (such as SELECT, MODIFY, or OWN) that have been granted to principals (users or groups) on the table. It does not provide information about row-level or column-level security policies.
*   In Databricks Unity Catalog, access to data objects follows a hierarchy. To perform an action on a table (like SELECT), a user must have the specific privilege on the table AND the 'USE' privilege on all parent objects (the catalog and the schema). Revoking USE SCHEMA blocks the user's access path to the table, even if the SELECT privilege still exists on the table itself.
*   ***The 'READ FILES' privilege*** is specifically designed to allow users to read files directly from cloud storage paths that are governed by an External Location object in Unity Catalog. This is required for ad-hoc path-based queries (e.g., querying JSON or CSV files directly) that do not target a registered table.
*   Unity Catalog currently supports applying at most one row filter function to a single table. To enforce multiple security constraints (such as filtering by both region and department), the logic for all conditions must be combined into a single SQL User-Defined Function (UDF) using logical operators like AND.
*   ***is_account_group_member() checks membership exclusively at the account level and cannot be used to verify membership in workspace-local groups.***



















