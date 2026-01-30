# Best Practices for Amazon RDS PostgreSQL Replication - Complete Guide

## Overview

Amazon RDS for PostgreSQL supports Read Replicas to offload reads and provide disaster recovery. Replicas can be in the same region or cross-region. This guide explains how to configure and monitor replicas to minimize replication lag.

### General Recommendation

- How to verify replica data freshness:
  1. Open CloudWatch in the AWS Console.
  2. Navigate to RDS metrics for your Read Replica.
  3. Check the `ReplicationLag` metric (seconds).
  4. Only route read queries to replicas when lag is low (e.g., < 5 seconds).
  5. Set CloudWatch alarms to alert when lag exceeds a threshold.

---

## Intra-Region Replication

Uses PostgreSQL streaming replication. Changes stream from source to replica; delays cause lag.

**Image Reference:** The article includes a diagram showing how RDS PostgreSQL performs replication between a source and replica in the same region using streaming replication.

### 1. Configure `wal_keep_size` Properly

**What it does:**
- Controls how many WAL segments are kept on the source before archiving to S3.
- Default: 2GB.

**Why it matters:**
- If a replica can’t find a segment locally, it downloads from S3, which is slower.
- More segments kept locally = faster replication.

**How to configure:**

1. Open RDS Console → Parameter Groups.
2. Select the parameter group for your source instance.
3. Find `wal_keep_size`.
4. Set a higher value (e.g., 10GB ≈ 625 WAL files).
5. Apply changes (dynamic, no restart).
6. Set this before creating new replicas.

**How to monitor:**

- Check source logs for:
  - `"Streaming replication has stopped"`
  - `"Streaming replication has been terminated"`
- Check replica logs for:
  - `"restored log file ... from archive"` (indicates S3 restore)
  - `"started streaming WAL from primary"` (indicates streaming resumed)

**Best practice:**
- Increase `wal_keep_size` before creating replicas to avoid archive restores during initial sync.

### 2. Avoid Heavy Write Activity at Source

**What happens:**
- High writes create many WAL files.
- Replicas must replay all WAL, which increases lag.

**How to monitor write activity:**

1. Open CloudWatch → RDS metrics for the source.
2. Monitor:
   - `TransactionLogsGeneration` (MB/s)
   - `TransactionLogsDiskUsage` (MB)
   - `WriteIOPS`, `WriteThroughput`, `WriteLatency`
3. Correlate spikes with `ReplicationLag` on replicas.

**Image Reference:** The article includes diagrams showing correlation between write activity and replication lag. One example shows metrics spiking around 16:20 and 17:00, with replica lag peaking at 11 minutes during the same period.

**How to reduce write impact:**

1. Distribute writes evenly:
   - Break large batches into smaller transactions.
   - Spread operations over time.
   - Avoid bulk operations during peak hours.
2. Enable WAL compression:
   - Open Parameter Groups → select source parameter group.
   - Set `wal_compression` = `ON`.
   - Apply (dynamic, no restart).
3. Set CloudWatch alarms:
   - Create alarms for `WriteLatency` and `WriteIOPS`.
   - Alert when thresholds are exceeded.

**Best practice:**
- Monitor write metrics and replica lag together to identify correlation.

### 3. Avoid Exclusive Locks on Source Tables

**What causes exclusive locks:**
- Commands: `DROP TABLE`, `TRUNCATE`, `REINDEX`, `CLUSTER`, `VACUUM FULL`, `REFRESH MATERIALIZED VIEW` (without `CONCURRENTLY`).
- These acquire `ACCESS EXCLUSIVE` locks, blocking other operations.

**How locks affect replication:**
- Locks are replayed on replicas.
- Longer locks = longer lag.

**How to monitor locks:**

1. Connect to the source database.
2. Run:
```sql
SELECT pid, 
       usename, 
       pg_blocking_pids(pid) AS blocked_by, 
       QUERY AS blocked_query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```
3. Review blocked queries and their blockers.
4. Check `pg_locks` for lock details.

**How to avoid:**
- Use `CONCURRENTLY` options when available (e.g., `REFRESH MATERIALIZED VIEW CONCURRENTLY`).
- Schedule maintenance during low-traffic periods.
- Monitor for long-running locks and investigate.

**Best practice:**
- Set up periodic monitoring of `pg_locks` and `pg_stat_activity` to catch blocking operations.

### 4. Configure Read Replica Parameters

**Parameter: `hot_standby_feedback`**

**What it does:**
- Sends feedback from replica to source about running queries.
- Prevents source from removing rows needed by replica queries.

**How to enable:**
1. Open Parameter Groups → select replica parameter group.
2. Set `hot_standby_feedback` = `ON`.
3. Apply (dynamic, no restart).

**Trade-offs:**
- Allows long-running queries on replicas.
- Can cause table bloat at source if queries run too long.

**How to monitor:**
- Check source logs for:
  - `"ERROR: canceling statement due to conflict with recovery"`
- Monitor long-running queries on replicas:
```sql
SELECT pid, usename, query_start, state, query
FROM pg_stat_activity
WHERE state = 'active' 
  AND query_start < now() - interval '5 minutes';
```

**How to manage long-running queries:**
- Terminate if needed:
```sql
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state = 'active';
```

**Parameters: `max_standby_archive_delay` / `max_standby_streaming_delay`**

**What they do:**
- Control how long WAL replay waits when replica queries conflict with source changes.
- `-1` = wait indefinitely (can cause high lag and WAL accumulation).

**How to configure:**
1. Open Parameter Groups → select replica parameter group.
2. Set appropriate values (e.g., `30s` or `1min`).
3. Avoid `-1` unless necessary.
4. Apply (dynamic, no restart).

**Best practice:**
- Monitor replica query duration and adjust these parameters accordingly.

### 5. Configure Read Replica Instance

**Instance class:**
- Use same or higher class than source.
- Replicas replay writes and handle reads; undersizing increases lag.

**How to configure:**
1. Open RDS Console → Create Read Replica.
2. Select same or higher instance class.
3. Match storage type (e.g., gp3, io1).

**Storage type:**
- Match source and replica storage types.
- Mismatches can increase lag.

**Best practice:**
- Monitor replica CPU, memory, and IOPS; upgrade if needed.

---

## Cross-Region Replication

Provides read scaling, disaster recovery, and cross-region migration.

### Differences from Intra-Region

**Replication mechanism:**
- Uses physical replication slots (not `wal_keep_size`).
- Slots retain WAL until consumed by the replica.

**How to monitor cross-region lag:**

1. Open CloudWatch → RDS metrics for source.
2. Monitor:
   - `OldestReplicationSlotLag` (MB of WAL not consumed)
   - `TransactionLogsDiskUsage` (total WAL storage)
3. Set alarms for high lag or WAL accumulation.

**How to prevent issues:**

1. Monitor source IOPS:
   - Low IOPS can delay WAL reads.
   - Ensure sufficient IOPS for WAL operations.
2. Monitor WAL accumulation:
   - High lag = more WAL retained.
   - Set alarms on `TransactionLogsDiskUsage`.
3. Monitor lag closely:
   - Cross-region lag is typically higher.
   - Set tighter thresholds for alerts.

**Best practice:**
- Monitor `OldestReplicationSlotLag` and `TransactionLogsDiskUsage` together to prevent storage issues.

---

## Logical Replication

Enables selective replication (table/database level) and supports different PostgreSQL versions.

### Common Use Case

AWS DMS (Database Migration Service) for change data capture.

### How to Configure Logical Replication

**Step 1: Enable logical replication parameter**

1. Open Parameter Groups → select source parameter group.
2. Set `rds.logical_replication` = `1`.
3. Apply (requires restart).
4. Restart the instance.

**Step 2: Monitor DMS tasks**

1. Open DMS Console → Replication Tasks.
2. Monitor task status.
3. If tasks pause or drop, investigate immediately.
4. If CDC is disabled, disable `rds.logical_replication`.

**Step 3: Monitor replication slots**

**How to find inactive slots:**
```sql
SELECT slot_name, slot_type, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots 
WHERE active = 'f';
```

**How to drop inactive slots:**
```sql
-- Drop a specific inactive slot
SELECT pg_drop_replication_slot('slot_name');

-- Or drop all inactive slots in one query
SELECT pg_drop_replication_slot(slot_name) 
FROM pg_replication_slots 
WHERE active = 'f';
```

**How to monitor slot lag:**
```sql
SELECT 
    slot_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag_size
FROM pg_replication_slots;
```

**Best practice:**
- Set up periodic checks for inactive slots and clean them up.
- Monitor slot lag to prevent WAL accumulation.

### Limitations and How to Handle Them

**1. Schema Changes**

**How to handle:**
- Apply schema changes at subscriber first, then publisher.
- Test changes on subscriber before applying to publisher.

**2. Sequence Data**

**How to handle:**
- After switchover/failover, update sequences:
```sql
-- Get current sequence value from publisher
SELECT last_value FROM sequence_name;

-- Set sequence value on subscriber
SELECT setval('sequence_name', last_value);
```

**3. TRUNCATE Operations**

**How to handle:**
- Use DELETE instead of TRUNCATE for large objects.
- Or revoke TRUNCATE privileges:
```sql
REVOKE TRUNCATE ON table_name FROM user_name;
```

**4. Partition Tables**

**How to handle:**
- Replicate partitioned tables one-to-one.
- Don’t replicate parent and children separately.

**5. Foreign Tables**

**Note:** Foreign tables are not replicated. Handle separately if needed.

---

## Complete Monitoring Setup

### Step-by-Step CloudWatch Monitoring

**1. Create Replication Lag Alarm**

1. Open CloudWatch → Alarms → Create Alarm.
2. Select `ReplicationLag` metric for your replica.
3. Set threshold (e.g., > 60 seconds).
4. Configure SNS notification.
5. Set alarm name and description.

**2. Create Write Activity Alarms**

1. Create alarm for `WriteIOPS` on source.
2. Create alarm for `WriteLatency` on source.
3. Create alarm for `TransactionLogsGeneration` on source.
4. Set thresholds based on baseline.

**3. Create WAL Usage Alarm (Cross-Region)**

1. Create alarm for `TransactionLogsDiskUsage` on source.
2. Set threshold (e.g., > 80% of allocated storage).
3. Create alarm for `OldestReplicationSlotLag` (e.g., > 1GB).

**4. Create Dashboard**

1. Open CloudWatch → Dashboards → Create Dashboard.
2. Add widgets for:
   - Replication lag
   - Write metrics
   - WAL usage
   - Replica CPU/Memory
3. Save dashboard.

### SQL Monitoring Queries

**Monitor replication status:**
```sql
-- Check replication status
SELECT 
    client_addr,
    state,
    sync_state,
    sync_priority
FROM pg_stat_replication;
```

**Monitor long-running queries on replica:**
```sql
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < now() - interval '5 minutes'
ORDER BY query_start;
```

**Monitor locks:**
```sql
SELECT 
    locktype,
    relation::regclass,
    mode,
    granted,
    pid,
    usename
FROM pg_locks
WHERE NOT granted;
```

---

## Summary Checklist

### Before Creating Replicas

- [ ] Increase `wal_keep_size` on source (e.g., 10GB)
- [ ] Enable `wal_compression` on source
- [ ] Verify source write patterns are distributed
- [ ] Set up CloudWatch alarms for write metrics

### After Creating Replicas

- [ ] Use same or higher instance class
- [ ] Match storage types
- [ ] Configure `hot_standby_feedback` if needed
- [ ] Set `max_standby_streaming_delay` appropriately
- [ ] Create replication lag alarms
- [ ] Monitor replica performance metrics

### Ongoing Maintenance

- [ ] Monitor replication lag daily
- [ ] Check for long-running queries weekly
- [ ] Monitor write activity patterns
- [ ] Review CloudWatch alarms regularly
- [ ] For logical replication: check inactive slots weekly
- [ ] For cross-region: monitor WAL accumulation daily

### Troubleshooting High Lag

1. Check CloudWatch metrics for write spikes
2. Review source logs for lock messages
3. Check replica for long-running queries
4. Verify instance classes match
5. Check `wal_keep_size` settings
6. For cross-region: check `OldestReplicationSlotLag`
7. For logical replication: check slot status

---

## Reference

Source: [AWS Database Blog - Best Practices for Amazon RDS PostgreSQL Replication](https://aws.amazon.com/blogs/database/best-practices-for-amazon-rds-postgresql-replication/)

**Note:** The original article includes visual diagrams showing:
- Intra-region replication architecture flow
- CloudWatch metrics graphs showing correlation between write activity and replication lag
- Visual representations of WAL file streaming and archiving processes

This guide provides step-by-step instructions for configuring, monitoring, and maintaining RDS PostgreSQL Read Replicas to minimize lag and ensure reliable replication.
