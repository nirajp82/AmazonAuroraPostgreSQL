# Best Practices for Amazon RDS PostgreSQL Replication — Complete Guide

## Introduction

Amazon RDS for PostgreSQL lets you add Read Replicas to offload read traffic and improve disaster recovery. Replicas can be in the same region or in another region. This guide explains how to configure and operate them so replication stays healthy and lag stays low.

---

## General Recommendation: Why and How You Check the Read Replica

### What is the goal?

Reads on a Read Replica should see the **same data** as the source. You only want to send read traffic to a replica when it has **caught up** (low lag).

### Why you must check the Read Replica

- A replica is always **behind** the source by some amount (replication lag).
- If you don’t check that lag, you may:
  - Serve **stale data** (e.g. old balances, outdated inventory).
  - Assume the replica is “live” when it is actually far behind.
- So you **must** verify that the replica is up to date before trusting it for reads.

**In short:** “Check the read replica” means “Check that it has the latest data (low lag).”

### How you confirm the replica has the latest data

You use **CloudWatch** to monitor **replication lag**:

- **Replication lag** = how far behind the replica is (in time or WAL).
- **Low lag** (e.g. &lt; 5–10 seconds) → replica is effectively up to date → **safe for reads**.
- **High lag** (e.g. &gt; 60 seconds) → replica is **not** up to date → **not** safe for reads until lag is fixed.

So: **Use CloudWatch replication lag to confirm whether the replica has the latest data.**

### What you should do

1. **Monitor replication lag in CloudWatch**  
   - This is how you confirm the replica has the latest data.  
   - Use the **ReplicationLag** metric (and slot lag for cross-region/logical, if applicable).

2. **Only use the replica for reads when lag is acceptable**  
   - Low lag → treat replica as having latest data → send reads there.  
   - High lag → do not use for “latest data” reads until lag is reduced or you accept staleness.

3. **Set CloudWatch alarms on replication lag**  
   - Get alerted when lag is high so you know the replica is **not** safe for fresh reads until you fix the cause.

**One-sentence summary:**  
Use CloudWatch to monitor replication lag so you can confirm the replica has the latest data; only run read queries on the replica when lag is low, and set alarms when it is high.

---

## Intra-Region Replication (Same Region)

RDS uses PostgreSQL **streaming replication**: changes stream from source to replica. Any delay in that process shows up as replication lag.

**Image reference:** The article includes a diagram of how RDS performs replication between source and replica in the same region.

---

### 1. Set `wal_keep_size` Correctly

**What it is**  
- `wal_keep_size` = minimum amount of WAL (transaction log) the source keeps on disk before archiving to S3.  
- Default in RDS: 2GB.

**Why it matters**  
- Replicas normally read WAL by **streaming** from the source.  
- If the source has already archived a segment to S3, the replica must **download and restore** it from S3, which is slower.  
- If that happens often, replication lag increases.  
- Keeping more WAL on the source (larger `wal_keep_size`) reduces how often replicas fall back to S3 and helps keep lag low.

**How to set it**  
1. RDS Console → Parameter groups → parameter group used by the **source** instance.  
2. Find `wal_keep_size`.  
3. Set a higher value (e.g. 10GB; the article notes ~625 WAL files at 16MB each).  
4. Save; change is dynamic (no restart).  
5. **Do this before creating new replicas** so they can stream from disk instead of S3 during initial sync.

**How to tell if replicas are using archive (S3)**  
- **Source logs:** look for “Streaming replication has stopped” or “Streaming replication has been terminated.”  
- **Replica logs:** look for “restored log file … from archive” (S3 restore) and “started streaming WAL from primary” (streaming resumed).

**Best practice:**  
Increase `wal_keep_size` on the source before adding replicas so streaming can start and stay on disk as much as possible.

---

### 2. Avoid Heavy Write Bursts on the Source

**What happens**  
- Every change on the source is written to WAL.  
- Replicas must **replay** that WAL.  
- A burst of writes creates a lot of WAL; replay takes longer → replication lag goes up.

**How to monitor**  
1. CloudWatch → Metrics → RDS → select the **source** instance.  
2. Use:  
   - **TransactionLogsGeneration** — WAL generated per second.  
   - **TransactionLogsDiskUsage** — WAL disk usage.  
   - **WriteIOPS**, **WriteThroughput**, **WriteLatency** — write load.  
3. Compare these with **ReplicationLag** on the replica; spikes in writes should align with spikes in lag.

**Image reference:** The article includes graphs showing high write activity (e.g. around 16:20 and 17:00) and replication lag rising to about 11 minutes at the same time.

**How to reduce impact**  
- **Spread writes:** break large batches into smaller transactions and spread them over time; avoid huge bulk operations at once.  
- **Enable WAL compression:** Parameter group → `wal_compression` = `ON` (dynamic). Less WAL = less to send and replay = lower lag over time.  
- **Alert on writes:** Create CloudWatch alarms on Write Latency and Write IOPS so you notice heavy write periods.

**Best practice:**  
Control and distribute write activity; use CloudWatch to correlate write spikes with replication lag.

---

### 3. Avoid Long-Lasting Exclusive Locks on the Source

**What causes them**  
- Commands that take **ACCESS EXCLUSIVE** locks:  
  `DROP TABLE`, `TRUNCATE`, `REINDEX`, `CLUSTER`, `VACUUM FULL`, `REFRESH MATERIALIZED VIEW` (without `CONCURRENTLY`).  
- These block other access to the table and their effect is written to WAL and replayed on replicas.

**Why it affects replication**  
- The lock is replayed on the replica; the longer the table is locked on the source, the longer the replica is affected and the more lag can build.

**How to monitor**  
- Connect to the **source** and run:

```sql
SELECT pid, 
       usename, 
       pg_blocking_pids(pid) AS blocked_by, 
       query AS blocked_query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

- Use `pg_locks` and `pg_stat_activity` to see who holds locks and what is blocked.

**How to avoid**  
- Prefer concurrent options where possible (e.g. `REFRESH MATERIALIZED VIEW CONCURRENTLY`).  
- Schedule heavy maintenance (VACUUM FULL, REINDEX, etc.) during low traffic.  
- Monitor for long-running blocking queries and fix or terminate them.

**Best practice:**  
Monitor `pg_locks` and `pg_stat_activity` on the source and avoid long-running exclusive locks during peak load.

---

### 4. Replica-Side Parameters: Standby and Lag

**`hot_standby_feedback`**  
- **What it does:** Replica tells the source which rows its queries might still need.  
- **Effect:** Source can delay removing those rows (e.g. from VACUUM), so long-running reads on the replica don’t conflict with recovery.  
- **Trade-off:** Long-running replica queries can cause **table bloat** on the source and risk Transaction ID wraparound if not monitored.  
- **How to set:** Replica’s parameter group → `hot_standby_feedback` = `ON` (dynamic).  
- **How to monitor:** Source logs for “canceling statement due to conflict with recovery”; replica for long-running queries (e.g. &gt; 5 minutes).  
- **Optional:** Terminate very long replica queries if needed:

```sql
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity
WHERE (now() - query_start) > interval '5 minutes'
  AND state = 'active';
```

**`max_standby_archive_delay` and `max_standby_streaming_delay`**  
- **What they do:** How long the replica may delay replay when a conflict with a running query occurs (archive vs streaming).  
- **Value -1:** Replica waits until the conflicting query finishes → lag can grow and WAL can pile up on the source.  
- **How to set:** Replica parameter group; use finite values (e.g. 30s, 1min) unless you explicitly accept indefinite delay.  
- **Best practice:** Monitor long-running replica queries; avoid -1 unless necessary; keep source healthy and lag under control.

---

### 5. Replica Instance and Storage

**Why it matters**  
- The replica must **replay** the same write workload as the source and also serve read queries.  
- If the replica is weaker than the source (smaller instance class or slower storage), it can’t keep up → lag increases.

**What to do**  
- **Instance class:** Same or **larger** than the source.  
- **Storage type:** Match the source (e.g. gp3, io1).  
- **How:** When creating the Read Replica in RDS, choose the same or higher instance class and same storage type.

**Best practice:**  
Use the same or higher instance class and same storage type for the replica so it can keep up with the source.

---

## Cross-Region Replication

Cross-region replicas are used for read scaling in another region, disaster recovery, or migration.

**How it differs from same-region**  
- Uses a **physical replication slot** on the source (not `wal_keep_size`) to retain WAL until the remote replica consumes it.  
- If the replica falls behind, WAL is retained on the source; long lag can fill disk.

**How to monitor**  
1. CloudWatch → RDS → **source** instance.  
2. **OldestReplicationSlotLag** — WAL not yet consumed (e.g. in MB).  
3. **TransactionLogsDiskUsage** — total WAL storage.  
4. Set alarms: high slot lag and/or high WAL usage so you act before the source runs out of space.

**Why source IOPS matters**  
- Cross-region replication reads WAL from the source and sends it over the network.  
- If the source has low IOPS, reading WAL can be slow → replication lag increases.  
- Monitor source IOPS and ensure it is sufficient for both normal workload and replication.

**Best practice:**  
Monitor cross-region lag and WAL usage closely; ensure source has enough IOPS and set alarms so WAL doesn’t fill the source disk.

---

## Logical Replication

Logical replication replicates a **subset** of data (e.g. specific tables) and can work across different PostgreSQL versions. A common use is **AWS DMS** for change data capture.

**Important:** Logical replication slots don’t know if the consumer is connected. If the consumer (e.g. DMS) stops or lags, WAL keeps accumulating on the source and can fill disk.

### Enable and Use Logical Replication Safely

**Enable**  
- Parameter group for the **source** → `rds.logical_replication` = `1` (requires restart).  
- Use this when the instance is a DMS source or other logical replication publisher.

**Monitor consumers**  
- If using DMS: monitor replication tasks. If a task is paused, dropped, or CDC is disabled, WAL can still accumulate.  
- If logical replication is not in use, turn `rds.logical_replication` off to avoid accidental slot creation and WAL growth.

**Monitor and clean up replication slots**  
- Find **inactive** slots (consumer disconnected or idle):

```sql
SELECT slot_name, slot_type, active 
FROM pg_replication_slots 
WHERE active = 'f';
```

- Drop inactive slots to free WAL:

```sql
SELECT pg_drop_replication_slot(slot_name) 
FROM pg_replication_slots 
WHERE active = 'f';
```

- Monitor slot lag (e.g. how much WAL is retained) and act if it grows.

**Best practice:**  
If logical replication is enabled, monitor replication slots and consumer health; drop inactive slots to avoid WAL filling the source.

### Logical Replication Limitations

- **Schema changes:** Apply at subscriber first, then publisher, where possible.  
- **Sequences:** Replicated, but after failover/switchover to the subscriber, update sequences to current values (e.g. with `setval`).  
- **TRUNCATE:** Can be problematic; prefer DELETE for large deletes, or REVOKE TRUNCATE.  
- **Partitioned tables:** Replicate as regular tables (one-to-one).  
- **Foreign tables:** Not replicated.

---

## Monitoring Setup: Putting It All Together

### Confirm Replica Has Latest Data (General Recommendation)

1. CloudWatch → RDS → select the **Read Replica**.  
2. Metric: **ReplicationLag** (seconds).  
3. Low lag (e.g. &lt; 5–10 s) → replica has latest data → OK for reads.  
4. Create an alarm when lag exceeds your threshold (e.g. 60 s) so you know when the replica is **not** safe for fresh reads.

### Alarms to Create

- **ReplicationLag** (replica) — main metric for “does replica have latest data?”  
- **TransactionLogsGeneration** (source) — WAL generation rate  
- **TransactionLogsDiskUsage** (source) — WAL growth, especially for cross-region/logical  
- **WriteIOPS**, **WriteLatency** (source) — write load  
- **OldestReplicationSlotLag** (source) — for cross-region and logical replication

### Useful Queries

**Replication status (source):**  
```sql
SELECT client_addr, state, sync_state, sync_priority 
FROM pg_stat_replication;
```

**Long-running queries (replica):**  
```sql
SELECT pid, usename, query_start, state, query
FROM pg_stat_activity
WHERE state = 'active' 
  AND query_start < now() - interval '5 minutes';
```

**Blocking (source):**  
```sql
SELECT pid, usename, pg_blocking_pids(pid) AS blocked_by, query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

---

## Summary Checklist

**Before creating replicas**  
- [ ] Increase `wal_keep_size` on source (e.g. 10GB).  
- [ ] Enable `wal_compression` on source.  
- [ ] Confirm write patterns are distributed; set write-related alarms.

**Replica configuration**  
- [ ] Same or higher instance class than source.  
- [ ] Same storage type as source.  
- [ ] Replica parameters (`hot_standby_feedback`, standby delays) set and monitored.

**Ongoing**  
- [ ] Use CloudWatch **ReplicationLag** to confirm replica has latest data before using it for reads.  
- [ ] Set replication lag alarm; investigate when it fires.  
- [ ] Monitor write activity and WAL usage; avoid heavy bursts and long exclusive locks.  
- [ ] For logical/cross-region: monitor slots and WAL; drop inactive slots.

**Troubleshooting high lag**  
1. Check CloudWatch: write spikes, ReplicationLag, WAL usage, slot lag.  
2. Check source logs for locks and replication messages.  
3. Check replica for long-running queries.  
4. Verify instance class and storage match.  
5. For cross-region/logical: check slot status and consumer health.

---

## Reference

Source: [Best practices for Amazon RDS PostgreSQL replication | AWS Database Blog](https://aws.amazon.com/blogs/database/best-practices-for-amazon-rds-postgresql-replication/)

The original post includes diagrams for intra-region replication flow and CloudWatch metrics (write activity vs. replication lag). This rewrite keeps all important configuration, monitoring, and operational details and states the general recommendation explicitly: **use CloudWatch replication lag to confirm the Read Replica has the latest data, and only use it for reads when lag is low.**
