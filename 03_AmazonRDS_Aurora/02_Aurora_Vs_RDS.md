# Comparing Amazon RDS (PostgreSQL) and Amazon Aurora (PostgreSQL)

---

## 1. Managed Database Overview

| Feature                 | RDS Postgres         | Aurora Postgres                                                               |
| ----------------------- | -------------------- | ----------------------------------------------------------------------------- |
| Managed Service         | Yes                  | Yes                                                                           |
| Underlying Engine       | Community Edition    | Built on Community Edition with architectural changes                         |
| Application Perspective | No difference        | Compatible with existing PostgreSQL apps and schemas                          |
| Storage Type            | EBS attached storage | Purpose-built, distributed storage replicated across three Availability Zones |
| Storage Understanding   | Block storage        | Database-aware storage tuned for PostgreSQL I/O                               |

### Explanation

* **Managed Service**
  Both services are fully managed by AWS (patching, backups, monitoring, failover).

* **Underlying Engine**
  RDS runs **unmodified community PostgreSQL**.
  Aurora is PostgreSQL-compatible but **re-architects the storage and replication layers** to improve availability, durability, and performance.

* **Application Perspective**
  Applications connect using the same PostgreSQL drivers, SQL syntax, schemas, and extensions (within Aurora-supported limits).
  No application-level code changes are typically required.

* **Storage Type**

  * RDS uses **EBS block storage** attached to the database instance.
  * Aurora uses a **purpose-built distributed storage system**, automatically replicated six ways across **three AZs** (2 copies per AZ).
    * Purpose-built distributed storage refers to a storage layer designed specifically for database workloads, providing high availability, durability, and performance through native replication across three Availability Zones.

* **Storage Understanding**

  * EBS is **generic block storage**, unaware of database semantics.
  * Aurora storage understands **PostgreSQL pages, redo, and crash recovery**, allowing smarter replication and faster recovery.

---

## 2. Storage and Scaling

| Feature            | RDS Postgres                   | Aurora Postgres                    |
| ------------------ | ------------------------------ | ---------------------------------- |
| Max Storage        | 64 TB                          | 128 TB                             |
| Auto-scaling       | +10 GB at a time, EBS-based    | +5 GB or 10% of current allocation |
| IOPS Limit         | EBS dependent (up to ~80,000)  | No hard storage IOPS limit         |
| Performance Impact | Scaling may impact performance | Minimal impact on compute          |

### Explanation

* **Maximum Storage**
  Aurora supports **larger databases** due to its distributed storage layer.

* **Auto-scaling**

  * RDS storage growth involves **EBS volume modification**, which may introduce latency.
  * Aurora storage grows **automatically and transparently**, without manual intervention.

* **IOPS Limits**

  * RDS IOPS are capped by the chosen EBS volume type.
  * Aurora spreads I/O across many storage nodes, eliminating a single storage bottleneck.

* **Performance Impact**
  Because Aurora decouples compute from storage, scaling storage does **not block or slow down query execution**.

---

## 3. Read Replicas

| Feature               | RDS Postgres                       | Aurora Postgres            |
| --------------------- | ---------------------------------- | -------------------------- |
| Max Read Replicas     | 5                                  | 15                         |
| Replication Mechanism | Asynchronous streaming replication | Storage-level replication  |
| Replication Lag       | Seconds (up to 30s)                | <100 ms                    |
| Cross-Region Replicas | Supported                          | Via Aurora Global Database |
| Compute Load          | Primary handles replication        | Minimal compute overhead   |

### Explanation

* **Replication Mechanism**

  * RDS uses **WAL streaming replication**, where replicas replay WAL records.
  * Aurora does **not ship WAL between instances**. All instances read from the same replicated storage.

* **Replication Lag**

  * WAL replay can lag during high write load.
  * Aurora replicas see changes almost immediately because data is already replicated at the storage layer.

* **Compute Load**
  RDS primaries spend CPU and I/O generating and shipping WAL.
  Aurora offloads replication to the storage layer, freeing compute for queries.

---

## 4. High Availability and Failover

| Feature           | RDS Postgres                  | Aurora Postgres            |
| ----------------- | ----------------------------- | -------------------------- |
| Failover Type     | Primary + standby             | Any read replica           |
| Failover Time     | 60–120 seconds                | ~30 seconds                |
| Replication Model | Synchronous primary → standby | Shared distributed storage |
| Standby Usage     | Idle until failover           | Actively serves reads      |
| Multi-AZ          | Supported                     | Built-in via storage       |

### Explanation

* **Failover Model**

  * RDS keeps a **passive standby** instance.
  * Aurora uses **active read replicas** that can be promoted instantly.

* **Failover Speed**
  Aurora avoids data resynchronization because all instances already share consistent storage.

* **Standby Usage**
  RDS standby provides no read value.
  Aurora replicas improve **both availability and read scalability**.

---

## 5. Backups and Disaster Recovery

| Feature                | RDS Postgres                | Aurora Postgres                |
| ---------------------- | --------------------------- | ------------------------------ |
| Backup Type            | Daily snapshots             | Continuous incremental backups |
| Backup Impact          | Possible performance impact | Minimal impact                 |
| Point-in-Time Recovery | Yes                         | Yes (faster)                   |
| Cross-Region DR        | Read replicas               | Aurora Global Database         |

### Explanation

* **Backup Method**

  * RDS snapshots are taken on a schedule.
  * Aurora continuously backs up changes at the storage layer.

* **Performance Impact**
  Aurora backups do not compete with database compute or I/O.

* **Disaster Recovery**
  Aurora Global Database enables **sub-second cross-region replication**, far faster than traditional replica-based DR.

---

## 6. Key Differences Summary

| Feature              | RDS Postgres        | Aurora Postgres              | Why It Matters                        |
| -------------------- | ------------------- | ---------------------------- | ------------------------------------- |
| Storage Architecture | EBS block storage   | Distributed DB-aware storage | Better durability and faster recovery |
| Max Storage          | 64 TB               | 128 TB                       | Supports larger SaaS databases        |
| Read Replicas        | 5                   | 15                           | Higher read scalability               |
| Replication Lag      | Seconds             | <100 ms                      | Near real-time reads                  |
| Backups              | Scheduled snapshots | Continuous                   | Lower operational risk                |
| Failover             | 1–2 minutes         | ~30 seconds                  | Higher availability                   |

---

## 7. Summary Points

* **RDS PostgreSQL**

  * Traditional PostgreSQL architecture
  * EBS-backed storage
  * WAL-based replication
  * Suitable for predictable, moderate workloads

* **Aurora PostgreSQL**

  * Distributed storage with built-in replication
  * Faster failover and recovery
  * Near-zero replication lag
  * Designed for large-scale, high-availability systems

### When to choose **Aurora PostgreSQL**

* Multi-tenant or SaaS platforms
* High read throughput
* Low RPO/RTO requirements
* Cross-region disaster recovery
* Enterprise migrations from Oracle or SQL Server


