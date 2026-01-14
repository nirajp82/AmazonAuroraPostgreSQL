# PostgreSQL Stats Collector & Activity Statistics

## Lesson Objective

This lesson covers the **PostgreSQL Stats Collector subsystem**, types of statistics available to users, and how to access these statistics. You will learn how to monitor database operations, table/index usage, I/O, and in-progress activities to identify performance bottlenecks or optimize database performance.

---

## 1. Overview of the Stats Collector Subsystem

* The **PostgreSQL Stats Collector** is responsible for **collecting and reporting statistical information** for the database.

* Various backend processes in PostgreSQL send statistics to the collector, which manages the information.

* **Two main types of stats:**

  1. **Counters (pg_stat)** – Statistics for operations on databases, tables, and indexes. Also includes I/O and vacuuming stats.
  2. **Activity (pg_stat_activity)** – A snapshot of all in-progress activities at a given point in time.

> **Memory Hook:** Counters = history; Activity stats = live snapshot of current queries.

* **Overhead:** Stats collection adds some load on the database, but it is generally worth it for performance tuning.

---

## 2. Controlling Stats Collection

* PostgreSQL provides **Boolean parameters** to control which stats are gathered:

| Parameter          | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `track_counts`     | Enables/disables counters for tables, indexes, and databases |
| `track_functions`  | Enables/disables stats for user-defined functions            |
| `track_io_timing`  | Enables/disables collection of I/O timing stats              |
| `track_activities` | Enables/disables collection of activity stats                |

* **Recommendation:** Keep all stats enabled unless there is noticeable degradation in performance.

> **Memory Hook:** Stats collection is like turning on multiple sensors. Only disable them if performance issues arise.

---

## 3. Accessing Collected Statistics

* Stats are accessed via **system views**.
* **Collected Stats Views** provide information on **counters** for tables, indexes, and databases.
* **Dynamic Stats Views** provide **snapshots of in-progress activities**.

### 3.1 Table & Index Statistics

* **Views:**

  * `pg_stat_all_tables` → Stats for all tables (system + user)
  * `pg_stat_user_tables` → Stats for user tables
  * `pg_stat_all_indexes` → Stats for all indexes
  * `pg_stat_user_indexes` → Stats for user indexes
  * `pg_stat_io` → I/O statistics for tables, indexes, and sequences
  * `pg_stat_user_functions` → Stats for user-defined functions

* **Pre-aggregated vs In-progress:**

  * `pg_stat_pre_exec_tables` → Stats for transactions **not yet aggregated** (in-progress)
  * Use `\d+ view_name` in psql to explore view attributes.

#### Key Attributes in `pg_stat_user_tables`:

| Attribute                         | Description                                   |
| --------------------------------- | --------------------------------------------- |
| `seq_scan`                        | Number of sequential scans run on the table   |
| `seq_tup_read`                    | Number of tuples returned by sequential scans |
| `idx_scan`                        | Number of index scans run on the table        |
| `idx_tup_fetch`                   | Number of tuples returned by index scans      |
| `n_tup_ins`                       | Total rows inserted                           |
| `n_tup_upd`                       | Total rows updated                            |
| `n_tup_del`                       | Total rows deleted                            |
| `n_live_tup`                      | Estimated number of live tuples               |
| `n_dead_tup`                      | Estimated number of dead tuples               |
| `last_vacuum` / `last_autovacuum` | Last vacuum timestamps                        |

> **Memory Hook:** These metrics help you identify **hot tables**, index effectiveness, and whether vacuuming is needed.

---

### 3.2 Database-Level Statistics (`pg_stat_database`)

* Counters for **each database**:

  * Row operations (`tup_insert`, `tup_update`, `tup_delete`)
  * Transactions (`xact_commit`, `xact_rollback`)
  * I/O (`blks_read`, `blks_hit`)
  * I/O timing (`blk_read_time`, `blk_write_time` in milliseconds)

> **Memory Hook:** If I/O is excessive, consider **moving the database to its own cluster**.

* `pg_stat_database_conflicts` → Tracks conflicts for each database.

---

### 3.3 I/O Statistics (`pg_stat_io`)

* Provides per-table, per-index, per-sequence metrics:

  * Disk blocks read
  * Buffer hits
  * Disk blocks written
* One row per table/index, identified by **schema name** and **relation name**.

---

### 3.4 User Functions

* `pg_stat_user_functions` → Tracks execution counts and timing for **user-defined functions**.

---

### 3.5 Activity Statistics (`pg_stat_activity`)

* Provides **live snapshots of queries in progress**.
* One row per **client connection**, including:

  * `client_addr` → Client IP address
  * `datname` → Database name
  * `query` → SQL currently executing
  * `state` → Query state (active, idle, waiting)
  * `wait_event_type` / `wait_event` → What the query is waiting for

> **Memory Hook:** Think of `pg_stat_activity` as a **real-time dashboard** for queries.

---

### 3.6 Background Processes

* `pg_stat_progress_vacuum` → One row per **vacuum process**, showing progress
* Other dynamic views exist for **application processes** (Aurora may vary)

> Use these views to **estimate vacuum time** or monitor long-running maintenance processes.

---

## 4. Example Queries

```sql
-- View table stats
SELECT * FROM pg_stat_user_tables;

-- View database-level stats
SELECT * FROM pg_stat_database;

-- View current client activity
SELECT * FROM pg_stat_activity;

-- View index usage stats
SELECT * FROM pg_stat_user_indexes;

-- View vacuum progress
SELECT * FROM pg_stat_progress_vacuum;

-- Example: Identify top tables by sequential scans
SELECT relname, seq_scan, idx_scan
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
```

---

## 5. Summary

* The **Stats Collector subsystem** gathers statistics for:

  1. **Counters** – Aggregated stats on tables, indexes, databases, I/O, and functions
  2. **Activity** – Dynamic snapshot of queries in progress, locks, and waits

* Stats are critical for:

  * **Performance tuning**
  * Identifying hot tables or indexes
  * Monitoring database I/O
  * Diagnosing query bottlenecks and long-running queries

> **Memory Hook:** Collected stats = historical counters, Dynamic stats = live snapshot.

---

## 6. FAQ

**Q1: Does stats collection affect performance?**

* Slightly, but generally negligible. Disable selectively only if needed.

**Q2: How frequently are stats updated?**

* Counters: Continuously updated in memory, periodically flushed to disk.
* Activity: Always reflects **current state**.

**Q3: How to check activity per client?**

* `pg_stat_activity` shows one row per connection with current query and wait event.

**Q4: How to monitor vacuum progress?**

* `pg_stat_progress_vacuum` shows progress for each background vacuum process.

**Q5: Can I monitor I/O per table or index?**

* Yes, using `pg_stat_io` and per-table/index views.

---
