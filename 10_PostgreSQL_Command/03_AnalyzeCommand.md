## Query Planner and Statistics Overview

The PostgreSQL query planner relies heavily on statistics stored in the system catalog to choose the most efficient execution plan for a SQL statement. If these statistics are inaccurate or outdated, the planner may select a poor plan, resulting in suboptimal query performance.

The planner has a strong dependency on catalog statistics, especially those stored in `pg_statistic`.

---

## pg_statistic Catalog View

`pg_statistic` is a system catalog that stores **table-level and column-level statistics** used by the PostgreSQL query planner.

These statistics describe:

* Data distribution in columns
* Number of distinct values
* Frequency of common values
* Null fraction
* Correlation between physical row order and column values

You can query this catalog using a `SELECT` statement to inspect the statistics (this is typically discussed alongside query processing and execution planning lessons).

> **Key Idea:** The query planner uses `pg_statistic` data to estimate costs and decide join methods, index usage, and scan types.

---

## Why Statistics Matter

* The planner uses statistics to determine the **most efficient query plan**
* Bad or outdated statistics lead to:

  * Incorrect row count estimates
  * Wrong join order
  * Indexes not being used (or used incorrectly)
  * Slower queries

In short: **bad statistics = bad query plans = poor performance**.

---

## ANALYZE Command

The `ANALYZE` command is used to **update statistics** stored in `pg_statistic`.

### Who Can Run ANALYZE

* A **superuser**, or
* The **owner of the database or table**

This is required because statistics are stored in the shared system catalog.

---

## How ANALYZE Works

* `ANALYZE` collects statistics using a **sample of table data**, not the entire table
* The statistics generated are **approximate**, not exact
* Statistics change over time as data is modified via:

  * INSERT
  * UPDATE
  * DELETE

The more frequently a table changes, the faster its statistics become outdated.

---

## ANALYZE Syntax and Usage

### Analyze the Entire Database

```sql
ANALYZE;
```

* Generates statistics for **all tables in the database**

### Analyze a Specific Table

```sql
ANALYZE table_name;
```

### Analyze Specific Columns

```sql
ANALYZE table_name (column_name);
```

### Verbose Mode

```sql
ANALYZE VERBOSE;
```

* Displays progress messages while statistics are being updated
* Useful for large databases

---

## Sampling and statistics_target

PostgreSQL controls the **size of the sample** used during `ANALYZE` using the parameter:

```text
default_statistics_target
```

* Default value: **100**
* Higher value:

  * More accurate statistics
  * Longer ANALYZE runtime
* Lower value:

  * Faster ANALYZE
  * Less accurate estimates

> Increasing this value should be done carefully, especially on large or busy systems.

---

## Best Practices for ANALYZE

* Run `ANALYZE` **periodically**
* Frequency depends on how fast data changes:

  * Slowly changing tables → once per day
  * Highly volatile tables → multiple times per day

### After Bulk Data Loads

* Always update statistics after large data loads
* This ensures the planner has accurate estimates immediately

---

## Autovacuum and ANALYZE

* **Autovacuum** automatically runs `ANALYZE`
* If autovacuum is enabled:

  * Manual ANALYZE may not be required
* If autovacuum is disabled:

  * You must run `ANALYZE` manually and regularly

---

## Memory Hook 🧠

> **Planner is blind without statistics.**
>
> `pg_statistic` tells the planner *how much data exists and how it is distributed*.
>
> `ANALYZE` is how PostgreSQL keeps that vision sharp.

---

## Quick Summary

* PostgreSQL stores planner statistics in `pg_statistic`
* Query planner depends on these stats for execution plans
* `ANALYZE` updates table and column-level statistics
* Statistics are approximate and decay over time
* Autovacuum usually handles ANALYZE automatically
* Outdated stats lead to inefficient query plans

---

## FAQ

### Q: Why are PostgreSQL statistics approximate?

Because `ANALYZE` samples data instead of scanning entire tables, which keeps overhead low on large datasets.

### Q: What happens if I don’t run ANALYZE?

The planner may misestimate row counts and choose inefficient execution plans, slowing queries.

### Q: Is increasing `default_statistics_target` always good?

No. Higher values improve accuracy but increase ANALYZE runtime and memory usage.

### Q: Do I need to run ANALYZE manually?

Not usually, if autovacuum is enabled. You must run it manually if autovacuum is disabled or after bulk data loads.

### Q: Can I analyze just one column?

Yes. PostgreSQL allows column-level statistics collection using `ANALYZE table (column)`.

### Q: Does ANALYZE lock tables?

No. ANALYZE runs concurrently and does not block normal reads and writes.
