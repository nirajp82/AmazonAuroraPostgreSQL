# Summary: Best Practices for Amazon RDS PostgreSQL Replication

## Overview

Amazon RDS for PostgreSQL supports Read Replicas to offload reads and provide disaster recovery. Replicas can be in the same region or cross-region. Proper configuration and monitoring are required to minimize replication lag.

### General Recommendation

- Run read queries on replicas only when they have the latest data.
- Monitor replication lag in CloudWatch.
- Minimize lag to avoid stale reads and protect source instance health.

---

## Intra-Region Replication

Uses PostgreSQL streaming replication. Data streams from source to replica; delays cause lag.

**Image Reference:** The article includes a diagram showing how RDS PostgreSQL performs replication between a source and replica in the same region using streaming replication.

### 1. Configure `wal_keep_size` Properly

- Purpose: Minimum size of WAL segments kept in `pg_wal` before archiving to S3.
- Default: 2GB.
- Impact: If a replica can’t find a segment locally, it downloads from S3, which is slower.
- Best practices:
  - Increase `wal_keep_size` (e.g., 10GB ≈ 625 WAL files) to reduce archive restores.
  - Set this before creating replicas.
  - Monitor for "Streaming replication has stopped/terminated" in logs.
  - Dynamic parameter (no restart required).

### 2. Avoid Heavy Write Activity at Source

- Problem: High writes create many WAL files; replaying them slows replication.
- Monitor:
  - `TransactionLogsGeneration` (WAL size per second)
  - `TransactionLogsDiskUsage`
  - `WriteIOPS`, `WriteThroughput`, `WriteLatency`
- Best practices:
  - Distribute writes evenly; avoid bursts.
  - Set CloudWatch alerts on write metrics.
  - Enable `wal_compression` to reduce WAL size and lag.

**Image Reference:** The article includes diagrams showing how high write activity at the source affects replication lag. One diagram shows metrics (`TransactionLogsDiskUsage`, `TransactionLogsGeneration`, `WriteIOPS`, `WriteThroughput`, `WriteLatency`) indicating heavy write pressure around 16:20 and 17:00, with replication lag peaking at 11 minutes during the same period.

### 3. Avoid Exclusive Locks on Source Tables

- Problem: Commands like `DROP TABLE`, `TRUNCATE`, `REINDEX`, `CLUSTER`, `VACUUM FULL`, and `REFRESH MATERIALIZED VIEW` (without `CONCURRENTLY`) acquire `ACCESS EXCLUSIVE` locks.
- Impact: Locks are replayed on replicas, increasing lag.
- Best practice: Monitor locks using:
```sql
SELECT pid, 
       usename, 
       pg_blocking_pids(pid) AS blocked_by, 
       QUERY AS blocked_query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### 4. Read Replica Parameter Settings

- `hot_standby_feedback`:
  - Enables replica feedback to source about running queries.
  - Allows long-running queries but can cause table bloat at source.
  - Monitor long-running queries to avoid storage issues and Transaction ID wraparound.
- `max_standby_archive_delay` / `max_standby_streaming_delay`:
  - Pauses WAL replay if source data changes during replica reads.
  - Value `-1` waits until query completes (can increase lag and WAL accumulation).
  - Monitor long-running queries; consider terminating if needed:
```sql
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
```

### 5. Read Replica Instance Configuration

- Use same or higher instance class than source.
- Match storage types between source and replica.
- Replicas replay writes and handle reads; undersizing increases lag.

---

## Cross-Region Replication

Provides read scaling, disaster recovery, and cross-region migration.

### Differences from Intra-Region

- Uses physical replication slots (not `wal_keep_size`).
- Monitor:
  - `OldestReplicationSlotLag` (WAL size in MB)
  - `TransactionLogsDiskUsage` (WAL storage usage)
- Considerations:
  - Longer distances increase lag.
  - WAL accumulates at source if lag occurs.
  - Monitor source IOPS; low IOPS can delay WAL reads and increase lag.
  - Monitor lag closely to prevent storage issues.

---

## Logical Replication

Enables selective replication (table/database level) and supports different PostgreSQL versions.

### Common Use Case

AWS DMS (Database Migration Service) for change data capture.

### Critical Considerations

1. Enable `rds.logical_replication`:
   - Required for DMS source.
   - Monitor DMS tasks; if paused/dropped or CDC disabled, disable the parameter.
2. Monitor replication slots:
   - Unconsumed WAL accumulates if replication pauses.
   - Find inactive slots:
```sql
SELECT slot_name FROM pg_replication_slots WHERE active='f';
```
   - Drop inactive slots:
```sql
SELECT pg_drop_replication_slot('slot_name') 
FROM pg_replication_slots 
WHERE active = 'f';
```

### Limitations

- Schema changes: Commit at subscriber first, then publisher.
- Sequence data: Update sequences to latest values after switchover/failover.
- Truncation: Use DELETE instead of TRUNCATE for large objects; consider REVOKE TRUNCATE.
- Partition tables: Replicate as regular tables (one-to-one).
- Foreign tables: Not replicated.

---

## Key Metrics to Monitor

| Metric | Purpose |
|--------|---------|
| `ReplicationLag` | Time delay between source and replica |
| `TransactionLogsGeneration` | WAL generation rate |
| `TransactionLogsDiskUsage` | WAL storage consumption |
| `WriteIOPS`, `WriteThroughput`, `WriteLatency` | Source write performance |
| `OldestReplicationSlotLag` | Cross-region replication delay (MB) |

---

## Summary of Best Practices

1. Configure `wal_keep_size` appropriately (intra-region).
2. Distribute writes evenly; avoid bursts.
3. Enable `wal_compression` to reduce WAL size.
4. Monitor and avoid long-running exclusive locks.
5. Use same or higher instance class for replicas.
6. Match storage types between source and replica.
7. Monitor replication slots (logical replication).
8. Set CloudWatch alerts on key metrics.
9. Monitor long-running queries on replicas.
10. For logical replication: monitor DMS tasks and clean up inactive slots.

---

## Reference

Source: [AWS Database Blog - Best Practices for Amazon RDS PostgreSQL Replication](https://aws.amazon.com/blogs/database/best-practices-for-amazon-rds-postgresql-replication/)

**Note:** The original article includes visual diagrams showing:
- Intra-region replication architecture flow
- CloudWatch metrics graphs showing correlation between write activity and replication lag
- Visual representations of WAL file streaming and archiving processes

These practices help minimize replication lag, optimize disaster recovery, distribute read workloads, and maintain source instance health.
