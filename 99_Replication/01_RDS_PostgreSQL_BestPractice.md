# Amazon RDS PostgreSQL Replication Best Practices

## Memory Hook

Think of **replication as “replaying the transaction movie”**:

* The **source instance writes WAL logs** (like filming the movie)
* **Read replicas replay the WAL logs** (watch the movie)
* Any delay in the movie streaming = **replication lag**

> Goal: keep the movie **streaming smoothly** to avoid stale data and storage overload.

---

## Purpose of Read Replicas

* Offload **read-heavy workloads** from the primary instance
* Keep the **primary instance focused on writes**
* Enable **disaster recovery (DR)** within the same region or across regions
* Support **cross-region migration and high availability**

> Memory Hook: Replicas are “extra eyes” on the data—they can read but cannot write.

---

## General Recommendations

* Always monitor **replication lag** via CloudWatch (`ReplicaLag`) to ensure **read queries return fresh data**
* Ensure that **read workloads don’t overload replicas**, which could increase lag
* Optimize the **source instance** to maintain healthy WAL generation and storage

---

## 1. Intra-Region Replication (Streaming Replication)

* RDS PostgreSQL uses **native PostgreSQL streaming replication**
* **How it works**: WAL changes stream continuously from source to replica
* **Replication lag occurs** if streaming is delayed or if WAL needs restoration from archives
<img width="772" height="503" alt="image" src="https://github.com/user-attachments/assets/369bcf83-aea3-410c-b092-570a9cea68e5" />

### Key Parameter: `wal_keep_size`

* Minimum WAL segments kept on source before archiving to S3
* Default: **2 GB**, can be increased via RDS Parameter Group
* **Why it matters**:

  * If WAL segments are removed too quickly, replicas fetch from S3 → slower replication
  * Increasing `wal_keep_size` reduces reliance on archive fetch
* Example: `wal_keep_size = 10GB` ≈ **625 WAL files kept locally**

### Error Logs to Watch

* `Streaming replication has stopped` → temporary streaming halt
* `Streaming replication has been terminated` → longer halt
* Logs show WAL restoration:

  ```
  restored log file "000000010000001A000000D3" from archive
  started streaming WAL from primary
  ```

> Memory Hook: More WAL locally = faster replication

---

## 2. Avoid Heavy Writes at Source

* High write activity → rapid WAL generation → replicas fall behind
* Metrics to monitor in CloudWatch:

  * `TransactionLogsGeneration` → WAL generated per second
  * `TransactionLogsDiskUsage` → disk used by WAL
  * `WriteIOPS`, `WriteThroughput`, `WriteLatency`

### Best Practices

* **Distribute write tasks** into smaller transactions
* Avoid bulk writes during peak replication periods
* Enable **`wal_compression = ON`** to reduce WAL size
* Consider **CloudWatch alerts** for high write latency or IOPS

> Memory Hook: Source writes = movie scenes; too many at once = replica can’t keep up

---

## 3. Avoid ACCESS EXCLUSIVE Locks

* Commands causing locks:

  * `DROP TABLE`, `TRUNCATE`, `REINDEX`, `CLUSTER`
  * `VACUUM FULL`, `REFRESH MATERIALIZED VIEW` (without CONCURRENTLY)
* These locks block access and slow replication because **replicas must wait to apply WAL**
* Monitor locks using:

```sql
SELECT pid, 
       usename, 
       pg_blocking_pids(pid) AS blocked_by, 
       query AS blocked_query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

> Memory Hook: Long locks = “pause the movie” = longer replication lag

---

## 4. Replica-Side Parameters

### `hot_standby_feedback`

* Lets replica tell source about long-running queries
* Prevents **VACUUM from removing rows** needed by replica
* Risk: **table bloat** and **storage exhaustion** if not monitored

### `max_standby_archive_delay` / `max_standby_streaming_delay`

* Allow long-running queries to complete on replica
* Value `-1` → WAL replay waits indefinitely → replication lag grows
* Best Practice: monitor long-running read queries and avoid indefinite delays

### Kill Long-Running Queries

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE (now() - query_start) > interval '5 minutes';
```

> Memory Hook: Protect long reads but monitor source health

---

## 5. Read Replica Instance Configuration

* **Instance class**: same or larger than source to handle write replay + reads
* **Storage type**: match source type for consistent I/O performance
* Misconfigured replicas → high replication lag and slow read performance

> Memory Hook: Underpowered replicas = lag and bottlenecks

---

## 6. Cross-Region Replication

* Provides **disaster recovery** and **read scaling** across regions
* Uses **physical replication slots** instead of just `wal_keep_size`
* Key metrics:

  * `OldestReplicationSlotLag` → WAL lag in MB
  * `TransactionLogsDiskUsage` → storage used by WAL
* Risks:

  * Network latency increases replication lag
  * WAL accumulation can fill source storage
  * Monitor IOPS and lag continuously

> Memory Hook: Long-distance replication = slower movie delivery

---

## 7. Logical Replication

* Used for **AWS DMS**, **selective table replication**, or **cross-version upgrades**
* Key risks:

  * WAL accumulation if slots are inactive
  * Requires monitoring and cleanup

### Useful SQL Commands

* Find inactive slots:

```sql
SELECT slot_name FROM pg_replication_slots WHERE active='f';
```

* Drop inactive slots:

```sql
SELECT pg_drop_replication_slot('slot_name');
```

* Combine both:

```sql
SELECT pg_drop_replication_slot(slot_name)
FROM pg_replication_slots
WHERE active='f';
```

### Limitations

* Schema changes → subscriber first, publisher second
* Sequence data may need manual update after failover
* Partitioned tables treated as normal tables → replicate one-to-one
* Foreign tables not replicated
* TRUNCATE may be replaced with DELETE to avoid accidental replication issues

> Memory Hook: Logical replication = selective movie streaming; watch for “buffer overflow” (WAL accumulation)

---

## 8. Summary Checklist

* ✅ Increase `wal_keep_size` appropriately
* ✅ Avoid heavy write bursts on source
* ✅ Avoid ACCESS EXCLUSIVE locks
* ✅ Monitor WAL metrics (`TransactionLogsGeneration`, `DiskUsage`)
* ✅ Monitor replication lag (time & storage)
* ✅ Match replica instance class & storage to source
* ✅ Use cross-region replication carefully; monitor IOPS & lag
* ✅ Clean up logical replication slots when inactive
* ✅ Monitor and manage long-running queries on replicas

---

## FAQ

**Q1: What causes replication lag most often?**

* Heavy writes, WAL archiving delays, exclusive locks, slow replicas, inactive replication slots

**Q2: Is some replication lag acceptable?**

* Yes. Small lag is fine for read scaling, but large lag risks **stale data and storage exhaustion**

**Q3: Can replica fall permanently behind?**

* Yes, especially in logical or cross-region replication if not monitored

**Q4: Can replicas be smaller than the source?**

* Technically yes, but underpowered replicas **increase lag and query latency**

**Q5: How to monitor replication health?**

* CloudWatch metrics: `ReplicaLag`, `TransactionLogsGeneration`, `OldestReplicationSlotLag`, `WriteIOPS`, `WriteThroughput`

---

**Memory Anchor:**

> In RDS PostgreSQL, **replication health = WAL health**.
> Keep WAL flowing smoothly, avoid heavy writes or locks, and monitor replication lag to maintain a healthy source and replica.

