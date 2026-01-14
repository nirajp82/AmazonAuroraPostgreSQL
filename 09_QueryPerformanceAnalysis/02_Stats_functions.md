# PostgreSQL Stats Utility Functions (Activity & Reset)

## Lesson Objective

This lesson explains how to use **PostgreSQL statistics utility functions** to:

* Access **activity statistics** using functions (instead of directly querying views)
* Retrieve **backend process statistics**
* **Reset statistics** at different levels:

  * Entire server
  * Shared (archiver / bgwriter)
  * Individual tables
  * User-defined functions

This lesson builds on the previous lesson where statistics were accessed using **SELECT queries on pg_stat views**.

---

## 1. Two Ways to Access PostgreSQL Statistics

PostgreSQL exposes statistics in **two equivalent ways**:

### 1. Direct Queries on Stats Views

You can query views such as:

* `pg_stat_activity`
* `pg_stat_user_tables`
* `pg_stat_database`

Example:

```sql
SELECT * FROM pg_stat_activity;
```

### 2. Utility Functions (Focus of This Lesson)

PostgreSQL also provides **utility functions** that:

* Fetch statistics from underlying stats views
* Reset statistics counters
* Retrieve backend process information

> **Memory Hook:**
> Views = raw data tables
> Functions = convenience APIs on top of those views

---

## 2. Backend Processes and Activity Statistics

### How PostgreSQL Handles Client Connections

* Every client connection **forks a backend process**
* Each backend process has a **Process ID (PID)**
* Statistics for activity are tracked **per backend process**

---

## 3. Getting Backend Process ID

### Function: `pg_backend_pid()`

Returns the PID of the backend process associated with **your current session**.

```sql
SELECT pg_backend_pid();
```

Example output:

```
 pg_backend_pid
----------------
 140321
```

> **Memory Hook:**
> Every psql session = exactly one backend PID

---

## 4. Getting Activity Statistics for a Backend Process

### Function: `pg_stat_get_activity(pid)`

* Retrieves activity statistics for a given backend process
* Internally reads from `pg_stat_activity`
* Requires a **PID** as input

### Example Workflow

#### Step 1: Get Backend PID

```sql
SELECT pg_backend_pid();
```

Result:

```
140321
```

#### Step 2: Get Activity Stats

```sql
SELECT *
FROM pg_stat_get_activity(140321);
```

Result:

* Current query text
* Query state
* Wait events
* Client information

> You will notice that the query shown is the same `SELECT pg_stat_get_activity(...)` query itself.

> **Memory Hook:**
> `pg_stat_get_activity()` = programmatic access to `pg_stat_activity`

---

## 5. Resetting PostgreSQL Statistics

At times, statistics need to be **reset**:

* After schema changes
* After adding/removing indexes
* After performance testing
* To measure fresh workload behavior

---

## 6. Resetting All Statistics (Server Level)

### Function: `pg_stat_reset()`

Resets **all statistics counters** across the server.

```sql
SELECT pg_stat_reset();
```

⚠️ Use carefully — this clears all accumulated stats.

---

## 7. Resetting Shared Statistics

### Function: `pg_stat_reset_shared()`

Resets shared statistics related to:

* WAL archiver
* Background writer

```sql
SELECT pg_stat_reset_shared();
```

> **Memory Hook:**
> Shared stats = system-wide background processes

---

## 8. Resetting Table-Level Statistics

### Function: `pg_stat_reset_single_table_counters(oid)`

* Resets stats for **one specific table**
* Requires the **OID (Object ID)** of the table

---

### 8.1 Getting Table OID

Use `regclass` or `oid` casting with a fully qualified table name.

```sql
SELECT
  'public.test_table'::regclass AS table_regclass,
  'public.test_table'::regclass::oid AS table_oid;
```

Example output:

```
 table_regclass | table_oid
----------------+-----------
 public.test    | 140321
```

---

### 8.2 Reset Table Stats

```sql
SELECT pg_stat_reset_single_table_counters(140321);
```

> **Memory Hook:**
> No OID → no reset
> Always fetch OID first

---

## 9. Resetting Function-Level Statistics

### Function: `pg_stat_reset_single_function_counters(oid)`

* Resets statistics for **one user-defined function**
* Requires function OID

### Get Function OID

```sql
SELECT 'public.my_function(int)'::regprocedure::oid;
```

### Reset Function Stats

```sql
SELECT pg_stat_reset_single_function_counters(<function_oid>);
```

---

## 10. When Should You Reset Stats?

Typical scenarios:

* After **adding indexes**
* After **schema changes**
* After **major query rewrites**
* Before **performance benchmarking**

> **Memory Hook:**
> Reset stats = reset stopwatch before a new race

---

## 11. Other Backend Utility Functions

* PostgreSQL provides additional backend utility functions
* These expose internal backend and process-level statistics
* Not all are covered in this lesson

📌 **Recommendation:**
Refer to PostgreSQL documentation to explore these functions further.

---

## 12. Summary

* PostgreSQL stats can be accessed via:

  * Stats views (`pg_stat_*`)
  * Utility functions (`pg_stat_get_*`)
* Every client connection has a backend PID
* `pg_backend_pid()` identifies your backend process
* `pg_stat_get_activity(pid)` retrieves activity stats
* Stats can be reset at:

  * Server level
  * Shared process level
  * Table level
  * Function level
* OIDs are required to reset object-level stats

> **Memory Hook:**
> Views show stats
> Functions fetch stats
> Reset functions clean the slate

---

## 13. FAQ

### Q1: Are utility functions different from stats views?

No. Utility functions **read from the same underlying stats views**. They are convenience wrappers.

### Q2: Do I need superuser privileges to reset stats?

Some reset functions require elevated privileges, especially server-wide or shared resets.

### Q3: Does resetting stats affect running queries?

No. Resetting stats only clears counters—it does not interrupt queries.

### Q4: Why use functions instead of querying views?

Functions are useful when:

* You want backend-specific stats
* You want programmatic access
* You want to reset stats

### Q5: Can I reset stats for only one table?

Yes, using `pg_stat_reset_single_table_counters(oid)`.

---
