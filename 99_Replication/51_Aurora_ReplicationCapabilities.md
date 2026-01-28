# Amazon Aurora PostgreSQL Replication Capabilities (2026)

## Overview

Amazon Aurora PostgreSQL-Compatible Edition is a cloud-native relational database that provides **high performance, durability, and PostgreSQL compatibility**.

It supports multiple replication strategies for:

* High availability (HA)
* Disaster recovery (DR)
* Global low-latency reads
* Data migrations and change data capture (CDC)

Aurora maintains **>99.99% availability** by storing **six copies of data across three Availability Zones (AZs)** and backing up continuously to **Amazon S3**.

**Memory Hook**

* **Physical replication** → Copy entire database quickly, low-latency, full data
* **Logical replication** → Stream only the changes (INSERT/UPDATE/DELETE), flexible but slightly higher lag

---

## 1. High Availability & Durability

* Data written to **shared cluster volume** → replicated to six copies across three AZs
* Automatic recovery from **storage and instance failures**
* Failover to Aurora Replica typically <30 seconds
* **Read replicas** scale reads without impacting writer

---

## 2. Aurora PostgreSQL Replication Options

| Option                         | Type               | Scope                                          | Latency / Lag             | Failover (RTO)             | Use Cases         |
| ------------------------------ | ------------------ | ---------------------------------------------- | ------------------------- | -------------------------- | ----------------- |
| **Aurora Replicas**            | Physical           | Same Region, up to 15 replicas                 | Near-zero (shared volume) | <30s (automatic)           | See section below |
| **Aurora Global Database**     | Physical           | Cross-Region, 1 primary + up to 10 secondaries | Typically <1 second       | <1 min (planned/unplanned) | See section below |
| **Native Logical Replication** | Logical            | Table-level, same/cross-Region                 | Seconds (depends on load) | Manual (app redirect)      | See section below |
| **pglogical Extension**        | Logical (enhanced) | Table/row/column-level                         | Seconds                   | Manual                     | See section below |

**Memory Hook**

* Physical = "Copy the whole book page-by-page" → fast, low-lag, full-database
* Logical = "Tell me only the changed sentences" → flexible, selective, higher lag

---

## 3. Aurora Replicas (Intra-Cluster, Storage Replication)

* Shares the **cluster volume** → no extra replication overhead
* Up to **15 replicas** across AZs in a Region

**Key Features (2026)**

* **Read availability** during writer restarts/failovers (PostgreSQL 12.16+, 13.12+, 14.9+, 15.4+, 16.1+)
* Automatic recovery from lag, network issues, or restarts
* Monitoring via CloudWatch `ReplicaLag` and `aurora_replica_status()` function

**Use Cases (Detailed)**

* **Scale reads without impacting writer** – multiple read-intensive apps (analytics dashboards, reporting, BI tools) can query replicas
* **High availability within a single Region** – failover if primary instance goes down
* **Maintenance or upgrades** – replicas continue serving reads during primary upgrades
* **Load balancing** – distribute read traffic across replicas to reduce latency

**Limitations**

* No cross-Region replication
* Replicas may restart during transaction ID wraparound or heavy load

---

## 4. Aurora Global Database (Cross-Region Physical Replication)

* One **primary cluster (writes)** + up to **10 secondary clusters (read-only)** in other Regions
* Sub-second replication lag
<img width="1083" height="520" alt="image" src="https://github.com/user-attachments/assets/65b4a860-ea6c-4a3a-bfc0-51e521ba5b98" />

**Architecture (Text Diagram)**

```
Primary Region (Writer)
  ↓ (sub-second replication)
Secondary Region 1 ─┬─ Aurora replicas
Secondary Region 2 ─┘
...
Secondary Region 10
```

**Key Features**

* Replication lag typically **<1 second**
* Managed planned switchover
* Manual unplanned failover (<1 min RTO)
* Near-zero RPO
* Independent scaling per secondary cluster

**Use Cases (Detailed)**

* **Global low-latency reads** – offices and users worldwide access their local Region with minimal latency
* **Multi-Region disaster recovery** – fast failover if entire primary Region is down
* **Continuous availability during maintenance/upgrades** – secondary clusters continue serving reads
* **Scalable secondary clusters for analytics or reporting** – up to 90 replicas per secondary cluster
* **Cross-Region application workloads** – SaaS apps, gaming, fintech apps requiring local data access globally
* **Read-heavy workloads distributed geographically** – e.g., dashboard queries, BI reports, ad-hoc analytics

**Limitations**

* Secondaries are read-only
* No Aurora Serverless v1, backtracking, or auto-scaling on secondaries
* Major version upgrades restricted if RPO management enabled

---

## 5. Native Logical Replication (PostgreSQL Built-in)

* Table-level replication using **publish/subscribe model** (since PostgreSQL 10)
* Tracks changes via **replication slots** and WAL

**Step-by-Step Flow**

1. **WAL Records Generated**

   * INSERT/UPDATE/DELETE written to WAL
   * WAL contains **physical change records**
   * WAL can normally be recycled after commit

2. **Replication Slot Tracks Progress**

   * Stores **starting WAL LSN** and **last confirmed read LSN**
   * Slot **does not read or decode** WAL; acts as a **bookmark**

3. **Logical Decoding**

   * Reads WAL from slot
   * Converts physical changes to logical changes (rows, columns, values)

4. **Changes Sent to Subscriber / Tool**

   * Subscriber (PostgreSQL, AWS DMS, Kafka, S3) applies changes
   * WAL retention continues until slot confirms consumption

5. **Progress Reported**

   * Slot updated with `confirmed_flush_lsn`
   * Old WAL freed safely

**Use Cases (Detailed)**

* **On-premises to Aurora migration** – near-zero downtime migration
* **Cross-cluster replication within same Region** – HA and DR scenarios
* **Database version upgrades** – replicate to newer major version cluster using AWS DMS
* **Role-based access / offloading reads** – create read-only replicas for developers, data scientists, analysts
* **Selective replication to external systems** – Redshift, S3, Kafka for analytics or ML
* **Minimal downtime for application upgrades** – subscriber mirrors only required tables

**Limitations**

* No automatic DDL replication (manual schema sync needed)
* Sequences not replicated automatically (manual sync required)
* Large objects (OID type) not replicated
* Base tables only (no views, materialized views, partitions differently)
* TRUNCATE with foreign keys may fail if not all tables subscribed

---

## 6. pglogical Extension (Logical – Enhanced)

* Adds advanced features:

  * Row/column filtering
  * Conflict detection & resolution
  * Sequence replication
  * Delayed replication
  * DDL replication helpers

* Supports bidirectional replication and cross-version upgrades

### 6a. pglogical Extension – Step-by-Step Flow

1. **WAL Records Generated**

   * INSERT/UPDATE/DELETE changes are written to WAL as usual
   * WAL contains physical change records
   * WAL can normally be recycled after commit

2. **Replication Slot Tracks Progress**

   * A **pglogical replication slot** is created
   * Tracks starting WAL LSN and last confirmed read LSN
   * Slot acts as a **bookmark** — does not decode WAL itself

3. **Logical Decoding (Enhanced)**

   * WAL read starting from slot position
   * pglogical plugin converts WAL changes to logical row-level operations
   * Additional enhancements:

     * **Row and column filtering** (only replicate subset)
     * **Conflict detection/resolution** (last-write wins, error, or custom rule)
     * **Delayed replication** if configured

4. **Changes Sent to Subscriber / Target**

   * Changes applied to subscriber database
   * Optionally, DDL commands replicated using `replicate_ddl_command()`
   * Supports **bidirectional replication**, cross-version replication

5. **Progress Reported**

   * Slot updated with `confirmed_flush_lsn`
   * WAL freed safely

**Memory Hook**

* pglogical = “Selective, smart replication with rules” → row/column filters, conflict resolution, optional DDL, delayed replication

**Use Cases (Detailed)**

* **Cross-version upgrades** – replicate from older PostgreSQL versions to newer Aurora versions
* **Bidirectional replication** – multi-master with conflict detection
* **Selective replication** – subset of tables, specific columns or rows only
* **Recovery scenarios** – delayed replication to recover from accidental deletes
* **Zero-downtime migration** – replicate production DB while running applications
* **Conflict resolution** – UPDATE/INSERT conflicts resolved automatically
* **Table schema replication** – copy initial schema and replicate DDL changes with helper functions

**Limitations**

* Per-database setup (no cluster-wide replication)
* Requires primary/unique keys
* UNLOGGED/TEMPORARY tables not replicated

---

## 7. Enabling Logical Replication

1. Set `rds.logical_replication = 1` (parameter group) → **reboot**
2. Grant `rds_replication` role
3. Native: `CREATE PUBLICATION` / `CREATE SUBSCRIPTION`
4. pglogical: enable extension → create nodes/subscriptions

---

## 8. Common Pitfalls & Monitoring

| Issue                        | Fix / Prevention                               |
| ---------------------------- | ---------------------------------------------- |
| Unused replication slots     | Drop them → avoid WAL bloat                    |
| Schema drift                 | Manually sync DDL                              |
| Sequence lag on failover     | Use `pg_dump` or set high values post-failover |
| High logical replication lag | Monitor `ReplicaLag`, scale consumer           |
| Global DB version mismatch   | Ensure compatible major/minor for switchover   |

Monitor via CloudWatch, `pg_replication_slots`, `aurora_replica_status()`

---

## 9. Memory Hook Summary

* **Aurora Replicas** → Fast HA reads in one Region
* **Aurora Global Database** → Global reads + disaster recovery
* **Native Logical Replication** → Table-level changes, migrations, CDC
* **pglogical** → Advanced logical replication (conflicts, selective, cross-version)

---

## 10. FAQ (2026)

**Q1: Physical vs Logical replication?**

* Physical = block-level, fast, full DB
* Logical = row-level, flexible/selective

**Q2: Best for multi-Region DR?**

* Aurora Global Database — sub-second lag, <1 min failover

**Q3: Can logical replication be cross-Region?**

* Yes, but lag is higher than Global Database physical replication

**Q4: Do I still need pglogical?**

* Only for advanced conflict handling, selective replication, or legacy versions

**Q5: Zero-downtime upgrades?**

* Use logical replication + AWS DMS

**Q6: Global Database vs read replicas?**

* Read replicas = same Region, fast failover
* Global = cross-Region, low-latency reads + DR

**Q7: Do I need logical decoding for PostgreSQL → PostgreSQL replication?**

* No, subscriptions handle decoding internally
* Replication slot still required

**Q8: Do I need logical decoding for third-party targets?**

* Yes, use `pg_recvlogical` or DMS with decoding plugin (`wal2json` / `test_decoding`)

**Q9: What is a replication slot?**

* Tracks **where WAL reading starts/ends**, ensures WAL retention

**Q10: Can multiple subscribers share one slot?**

* No, one active reader per slot

**Q11: What happens if a replication slot is unused?**

* WAL accumulates → potential storage bloat

---

**References**

* [AWS Aurora PostgreSQL Replication](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Replication.html)
* [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
[* Historical blog post (2021)
](https://aws.amazon.com/blogs/database/understand-replication-capabilities-in-amazon-aurora-postgresql/)
