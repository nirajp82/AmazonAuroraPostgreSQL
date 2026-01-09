# Comparing Amazon RDS (Postgres) and Amazon Aurora (Postgres)

---

## 1. Managed Database Overview

| Feature                 | RDS Postgres         | Aurora Postgres                                       |
| ----------------------- | -------------------- | ----------------------------------------------------- |
| Managed Service         | Yes                  | Yes                                                   |
| Underlying Engine       | Community Edition    | Built on Community Edition with architectural changes |
| Application Perspective | No difference        | Compatible with existing PostgreSQL apps and schemas  |
| Storage Type            | EBS attached storage | Purpose-built, distributed storage replicated across three Availability Zones. Purpose-built distributed storage refers to a storage layer designed specifically for database workloads, providing high availability, durability, and performance through native replication across three Availability Zones.   |
| Storage Understanding   | Block storage        | Database-aware storage tuned for Postgres I/O         |

---

## 2. Storage and Scaling

| Feature            | RDS Postgres                     | Aurora Postgres                                    |
| ------------------ | -------------------------------- | -------------------------------------------------- |
| Max Storage        | 64 TB                            | 128 TB                                             |
| Auto-scaling       | +10 GB at a time, depends on EBS | +5 GB or 10% of current allocation                 |
| IOPS Limit         | Depends on EBS type, max 80,000  | No hard upper limit (depends on instance type)     |
| Performance Impact | Scaling may impact performance   | Efficient storage layer, minimal impact on compute |

---

## 3. Read Replicas

| Feature               | RDS Postgres                                    | Aurora Postgres                                          |
| --------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| Max Read Replicas     | 5                                               | 15                                                       |
| Replication Mechanism | Asynchronous PostgreSQL streaming replication   | Storage-level replication, shared volume across replicas |
| Replication Lag       | Seconds (up to 30s)                             | <100 ms                                                  |
| Cross-Region Replicas | Supported                                       | Not directly; use Global Database option                 |
| Compute Load          | Replication consumes primary instance resources | Minimal compute overhead; replication is storage-based   |

---

## 4. High Availability and Failover

| Feature                               | RDS Postgres                                | Aurora Postgres                                              |
| ------------------------------------- | ------------------------------------------- | ------------------------------------------------------------ |
| Failover Type                         | Multi-AZ deployments with primary & standby | Read replicas act as failover targets                        |
| Failover Time                         | 60–120 seconds                              | ~30 seconds                                                  |
| Replication Between Primary & Standby | Synchronous                                 | Shared storage across instances (no traditional replication) |
| Use of Standby                        | Standby instance for failover               | Read replicas for failover or disaster recovery              |
| Multi-Site / Multi-AZ                 | Supported                                   | Storage automatically replicated across AZs                  |

---

## 5. Backups and Disaster Recovery

| Feature             | RDS Postgres                             | Aurora Postgres                                           |
| ------------------- | ---------------------------------------- | --------------------------------------------------------- |
| Backup Type         | Daily backups during user-defined window | Continuous incremental backups                            |
| Backup Impact       | May affect performance                   | Minimal performance impact                                |
| Recovery            | Point-in-time recovery                   | Point-in-time recovery, faster due to distributed storage |
| Cross-Region Backup | Supports cross-region replication        | Use **Aurora Global Database** for DR strategy            |

---

## 6. Key Differences Summary

| Feature                  | RDS Postgres         | Aurora Postgres               | Notes                                                      |
| ------------------------ | -------------------- | ----------------------------- | ---------------------------------------------------------- |
| Storage Architecture     | EBS attached storage | Distributed, DB-aware storage | Aurora is optimized for Postgres workloads                 |
| Maximum Storage          | 64 TB                | 128 TB                        | Aurora scales larger automatically                         |
| Read Replicas            | 5                    | 15                            | Aurora replication at storage tier, efficient              |
| Replication Lag          | Seconds (up to 30s)  | <100 ms                       | Aurora replication is faster                               |
| Cross-Region Replication | Supported            | Not directly                  | Use Aurora Global Database for cross-region DR             |
| Backups                  | Daily backups        | Continuous incremental        | Aurora backup has minimal performance impact               |
| Failover                 | 60–120 sec           | ~30 sec                       | Aurora failover is faster and uses storage tier efficiency |

---

### 7. Summary Points

* **RDS Postgres:** Traditional managed Postgres with standard storage and streaming replication.
* **Aurora Postgres:** Enterprise-class Postgres with high performance, distributed storage, fast replication, more read replicas, and cost-efficient scaling.
* **When to choose Aurora:**

  * Applications needing high performance, high availability, and low replication lag
  * Enterprise workloads replacing Oracle or SQL Server with PostgreSQL
  * Use cases with heavy read scaling or continuous backups
