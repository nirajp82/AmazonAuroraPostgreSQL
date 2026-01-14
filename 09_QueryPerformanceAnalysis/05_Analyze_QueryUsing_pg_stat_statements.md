# Analyzing Query Performance Using `pg_stat_statements`

## Purpose of This README

This document explains how to **use `pg_stat_statements` in practice** to:

* Identify slow and expensive queries
* Detect high-frequency workload patterns
* Analyze CPU vs I/O bottlenecks
* Validate performance improvements
* Support real-world production troubleshooting

---

## Prerequisites

### Enable the Extension

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Required Configuration

```conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 5000
pg_stat_statements.track = top
pg_stat_statements.save = on
track_io_timing = on
```

> Restart required after changing `shared_preload_libraries`.

---

## How `pg_stat_statements` Works (Quick Recap)

* Queries are **normalized (parameterized)**
* Stats are **aggregated across semantically similar queries**
* Statistics are maintained **in shared memory**
* Can persist across restarts (`pg_stat_statements.save = on`)
* Survives connection churn (unlike `pg_stat_activity`)

---

## Key Columns You Will Use Most

These columns are critical when analyzing query performance using `pg_stat_statements`. They help you understand **query behavior, resource usage, and bottlenecks**.

| Column              | Meaning                                    | Practical Insight                                                                                                                                           |
| ------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `query`             | Parameterized SQL query                    | Literal values are replaced with `$1`, `$2`, etc. Allows aggregation of semantically similar queries. Use this to identify **frequent, expensive queries**. |
| `calls`             | Number of executions                       | Shows query frequency. High `calls` can indicate **hot queries** that may benefit from caching, indexing, or query optimization.                            |
| `total_exec_time`   | Total time spent executing this query (ms) | Measures **overall cost** of the query across all executions. Focus optimization on queries with high `total_exec_time`.                                    |
| `mean_exec_time`    | Average execution time per call (ms)       | Highlights queries that are individually **slow**. Useful for detecting performance issues caused by inefficient execution plans.                           |
| `max_exec_time`     | Slowest single execution (ms)              | Detects **worst-case latency**, helpful for troubleshooting tail-end performance issues or lock contention.                                                 |
| `rows`              | Total rows returned                        | High values may indicate **large result sets**, which can increase network latency or memory usage. Consider adding pagination or filters.                  |
| `shared_blks_hit`   | Buffer cache hits                          | Number of blocks served from memory (cache). High hits → query is memory-efficient; low hits → may need **better indexing** or tuning of `shared_buffers`.  |
| `shared_blks_read`  | Disk reads from shared buffers             | Number of blocks read from disk. High values → query is **I/O bound**, may indicate missing indexes or cold caches.                                         |
| `temp_blks_written` | Temporary blocks written to disk           | Temporary files written during sorts, hashes, or joins. High values → increase `work_mem` or optimize query to reduce disk spills.                          |
| `blk_read_time`     | Time spent reading blocks from disk (ms)   | Helps identify **I/O-bound queries**. High `blk_read_time` suggests storage latency is affecting performance.                                               |
| `blk_write_time`    | Time spent writing blocks to disk (ms)     | Highlights queries with heavy disk writes. Useful for **write-intensive workloads** and detecting inefficient updates/inserts.                              |

### How to Use These Columns

1. **Start with `total_exec_time`** → find the biggest time consumers.
2. **Check `mean_exec_time`** → identify slow queries per execution.
3. **Look at `calls`** → high-frequency queries can have an outsized impact.
4. **Analyze I/O metrics (`shared_blks_read`, `blk_read_time`)** → detect storage bottlenecks.
5. **Check temporary usage (`temp_blks_written`)** → identify memory pressure or suboptimal queries.
6. **Correlate `query` with EXPLAIN ANALYZE** → confirm execution plan issues.

> **Memory Hook 🧠**
> High `total_exec_time` + high `mean_exec_time` + high `shared_blks_read` → a candidate for query tuning or indexing.

---

## 🔥 Top Queries by Total Execution Time

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Why This Matters

These queries consume the **largest share of total database time**, even if individual executions are fast.

---

## 🐌 Slowest Queries by Average Execution Time

```sql
SELECT
    query,
    calls,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

Detects:

* Missing indexes
* Poor join strategies
* Inefficient filters

---

## 🔁 Most Frequently Executed Queries

```sql
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

Optimizing these usually gives **the highest ROI**.

---

## 📦 Queries Returning Large Result Sets

```sql
SELECT
    query,
    calls,
    rows / NULLIF(calls, 0) AS avg_rows_per_call
FROM pg_stat_statements
ORDER BY avg_rows_per_call DESC
LIMIT 10;
```

Large result sets often indicate:

* Missing pagination
* Over-fetching
* Excessive network transfer

---

## 💾 Disk-Heavy Queries (Low Cache Efficiency)

```sql
SELECT
    query,
    shared_blks_read,
    shared_blks_hit,
    ROUND(
      shared_blks_read * 100.0 /
      NULLIF(shared_blks_hit + shared_blks_read, 0), 2
    ) AS disk_read_pct
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;
```

High disk read percentage usually means:

* Poor indexing
* Cold data access
* Inefficient access patterns

---

## 🧠 Cache-Friendly Queries

```sql
SELECT
    query,
    shared_blks_hit,
    shared_blks_read
FROM pg_stat_statements
ORDER BY shared_blks_hit DESC
LIMIT 10;
```

Use these as **reference patterns** for good query design.

---

## 🧮 Queries Causing Temporary File Usage

```sql
SELECT
    query,
    temp_blks_written,
    calls
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;
```

Common causes:

* Large `ORDER BY`
* Hash joins
* Insufficient `work_mem`

---

## ⏳ I/O Latency Heavy Queries

```sql
SELECT
    query,
    blk_read_time,
    blk_write_time,
    calls
FROM pg_stat_statements
ORDER BY (blk_read_time + blk_write_time) DESC
LIMIT 10;
```

Helps differentiate:

* CPU-bound queries
* I/O-bound queries

---

## 🔐 Queries with High Variance (Unstable Performance)

```sql
SELECT
    query,
    mean_exec_time,
    stddev_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY stddev_exec_time DESC
LIMIT 10;
```

High variance often indicates:

* Lock contention
* Parameter sensitivity
* Data skew

---

## 🔍 Correlating with `EXPLAIN ANALYZE`

`pg_stat_statements` tells you **which queries hurt**.
`EXPLAIN ANALYZE` tells you **why they hurt**.

### Workflow

1. Identify a problematic query:

```sql
SELECT query, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 1;
```

2. Run execution analysis:

```sql
EXPLAIN (ANALYZE, BUFFERS)
<parameterized query>;
```

3. Correlate:

* High `shared_blks_read` → Seq scans / missing indexes
* Temp blocks → Sorts / hash joins
* High execution time → Join order or plan choice

---

## 🧰 Production Troubleshooting Playbook

**Step 1 – Identify offenders**

```sql
SELECT query, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC;
```

**Step 2 – Check live behavior**

```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state <> 'idle';
```

**Step 3 – Analyze plans**

```sql
EXPLAIN (ANALYZE, BUFFERS)
<query>;
```

**Step 4 – Apply fixes**

* Add indexes
* Rewrite queries
* Tune `work_mem`
* Reduce result size

**Step 5 – Reset and validate**

```sql
SELECT pg_stat_statements_reset();
```

---

## 🔗 Mapping Wait Events → Query Stats

Combine:

* `pg_stat_activity.wait_event_type`
* `pg_stat_activity.wait_event`
* `pg_stat_statements`

Examples:

| Wait Event Type | Interpretation             |
| --------------- | -------------------------- |
| `Lock`          | Lock contention            |
| `LWLock`        | Shared memory contention   |
| `IO`            | Storage or buffer pressure |
| `Client`        | Slow application behavior  |
| `Timeout`       | Explicit waits / sleeps    |

This mapping helps explain **why aggregated time is high**.

---

## ☁️ Aurora PostgreSQL–Specific Tuning Notes

* `pg_stat_statements` works natively in Aurora
* Storage I/O is handled by **Aurora distributed storage**
* Some OS-level wait events are abstracted

Best practices:

* Combine with **Performance Insights**
* Watch for:

  * High latency + low CPU → storage waits
  * Lock waits amplified by high concurrency
* Focus on:

  * Query shape
  * Index efficiency
  * Locking behavior

---

## 🔄 Reset Options

### Reset Everything

```sql
SELECT pg_stat_statements_reset();
```

### Reset for Specific Database

```sql
SELECT pg_stat_statements_reset(dbid := <db_oid>);
```

### Reset for Specific User

```sql
SELECT pg_stat_statements_reset(userid := <user_oid>);
```

---

## pg_stat_activity vs pg_stat_statements

| Feature     | pg_stat_activity | pg_stat_statements |
| ----------- | ---------------- | ------------------ |
| Scope       | Live sessions    | Historical         |
| Persistence | No               | Yes                |
| Granularity | Per connection   | Per query          |
| Best for    | Debugging now    | Optimization       |

---

## Memory Hook 🧠

> **`pg_stat_activity` shows what is slow now.
> `pg_stat_statements` shows what hurts over time.
> `EXPLAIN ANALYZE` explains why.**

---

## Common Pitfalls

* Ignoring query normalization
* Not resetting stats before testing
* Tuning without execution plans
* Forgetting to enable I/O timing

---

## Summary

* `pg_stat_statements` is essential for **query-level performance tuning**
* Always evaluate:

  * Total time
  * Frequency
  * Disk usage
  * Variance
* Correlate with:

  * `EXPLAIN ANALYZE`
  * Wait events
  * System metrics

---
