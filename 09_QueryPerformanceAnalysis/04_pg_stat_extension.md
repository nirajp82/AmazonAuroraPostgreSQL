# PostgreSQL Query Performance Monitoring with `pg_stat_statements`

## Lesson Objective

Understand how PostgreSQL provides **persistent, aggregated query performance statistics** using the
`pg_stat_statements` extension, and how it differs from point-in-time monitoring using `pg_stat_activity`.

---

## Why `pg_stat_activity` Is Not Enough

The statistics exposed by `pg_stat_activity` are:

* **Point-in-time only**
* **Not persistent**
* Useful only for **currently running queries**

Because of this, `pg_stat_activity` **cannot help investigate performance issues that occurred in the past**.

To analyze historical query performance trends, PostgreSQL provides the
**`pg_stat_statements` extension**.

---

## What Is `pg_stat_statements`?

`pg_stat_statements` is a **monitoring extension** that:

* Aggregates query performance statistics
* Persists data across time (and optionally across restarts)
* Tracks queries across **all connections**

It must be **enabled per database** using:

```sql
CREATE EXTENSION pg_stat_statements;
```

Once enabled, it **automatically starts collecting statistics** for that database.

---

## How Queries Are Tracked (Parameterized Queries)

`pg_stat_statements` **does not store individual query executions**.

Instead, it tracks a **parameterized (normalized) form** of queries.

### Example

Original queries:

```sql
SELECT * FROM film WHERE film_id = 10;
SELECT * FROM film WHERE film_id = 20;
SELECT * FROM film WHERE film_id = 30;
```

Parameterized representation:

```sql
SELECT * FROM film WHERE film_id = $1;
```

Key points:

* Literal values are replaced with **positional parameters** (`$1`, `$2`, …)
* All **semantically equivalent queries** share one entry
* Statistics are **aggregated across all executions**

So if the query runs **10,000 times**, it appears as **one row** with aggregated metrics.

---

## Internal Query Identification and Hashing

Internally, each parameterized query is identified using a **hash code**.

Important caveats:

* **Hash collisions are possible**

  * Two different queries may map to the same hash
  * Statistics may be incorrectly aggregated
* The reverse is also possible:

  * Semantically identical queries may generate different hashes
  * Stats may be split across multiple rows

👉 **Always correlate the parameterized query text with real application queries** when analyzing data.

---

## Accessing the Statistics

Statistics are exposed via the view:

```sql
pg_stat_statements
```

### Compound Primary Key

Each row is uniquely identified by:

* `dbid` (Database ID)
* `userid` (User ID)
* `queryid` (internal hash of the parameterized query)

---

## Metrics Captured by `pg_stat_statements`

### 1. Execution Counts

* `calls` – number of times the query executed

---

### 2. Timing Statistics (milliseconds)

* Total execution time
* Minimum execution time
* Maximum execution time
* Mean (average) execution time
* Standard deviation

---

### 3. I/O Statistics – Shared Blocks

Shared blocks represent **regular tables and indexes**.

Captured metrics include:

* Blocks hit from shared buffers (cache hits)
* Blocks read from disk
* Dirty blocks generated
* Blocks written to disk

---

### 4. Local and Temporary Blocks

* **Local blocks**: data from temporary tables and indexes
* **Temporary blocks**: working data for operations like sorting and hashing

👉 If queries with `ORDER BY`, `GROUP BY`, or hashes are slow,
**temporary block statistics are critical**.

---

### 5. Block I/O Timing

* Time spent reading and writing blocks
* Captured **only if**:

```conf
track_io_timing = on
```

This parameter was discussed in earlier lectures.

---

## Difference Between `pg_stat_activity` and `pg_stat_statements`

| Aspect      | `pg_stat_activity`   | `pg_stat_statements` |
| ----------- | -------------------- | -------------------- |
| Scope       | Active sessions only | All queries          |
| Timeframe   | Point-in-time        | Aggregated over time |
| Persistence | No                   | Yes (configurable)   |
| Granularity | Connection-level     | Query-level          |
| Use case    | Live troubleshooting | Historical analysis  |

---

## Persistence Across Server Restarts

`pg_stat_statements` stores data **in memory**, but can persist it across restarts.

Configuration parameter:

```conf
pg_stat_statements.save = on
```

* Default: `on`
* Recommendation: **Keep enabled**

---

## Key Configuration Parameters

### 1. `pg_stat_statements.max`

* Maximum number of queries tracked
* Default: `5000`
* Controls **memory usage**

---

### 2. `pg_stat_statements.track`

Controls which queries are tracked:

| Value           | Behavior                                   |
| --------------- | ------------------------------------------ |
| `none`          | Disable tracking                           |
| `top` (default) | Track only top-level queries               |
| `all`           | Track top-level + queries inside functions |

Default is `top` for **performance reasons**.

---

### 3. `pg_stat_statements.track_utility`

* Tracks utility commands such as:

  * `VACUUM`
  * `ANALYZE`
* Default: `on`

---

## Resetting Statistics

Use the function:

```sql
pg_stat_statements_reset()
```

### Options:

* No arguments → reset **all statistics**
* Specify database ID, user ID, or query ID to reset selectively

---

## When to Use `pg_stat_statements`

* Identifying **slow queries over time**
* Finding **high-frequency queries**
* Comparing performance **before and after indexing**
* Detecting queries with excessive I/O

---

## Memory Hook 🧠

> **`pg_stat_activity` tells you what is happening now.
> `pg_stat_statements` tells you what has been happening over time.**

---

## FAQ

### Q1. Does `pg_stat_statements` store every query execution?

No. It stores **aggregated statistics** for parameterized queries.

---

### Q2. Can stats survive PostgreSQL restarts?

Yes, if `pg_stat_statements.save = on`.

---

### Q3. Can hash collisions make stats inaccurate?

Yes. Always validate parameterized queries against real workloads.

---

### Q4. Does it track queries inside functions by default?

No. You must set:

```conf
pg_stat_statements.track = all
```

---

### Q5. Does this extension impact performance?

Yes, slightly. Defaults are chosen to **balance visibility and overhead**.

---

## Summary

* `pg_stat_statements` provides **persistent, aggregated query statistics**
* Queries are tracked in **parameterized form**
* Statistics help diagnose **historical performance issues**
* Proper configuration is essential to balance **insight vs overhead**
