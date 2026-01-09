##  Step-by-Step Write and Commit Flow (RDS vs Aurora)

This section explains **what actually happens when an application writes data** and how **commit, durability, replication, and failover** differ between RDS PostgreSQL and Aurora PostgreSQL.

---

## 1 Write & Commit Flow – RDS PostgreSQL

### High-level idea

* RDS follows **standard PostgreSQL behavior**
* **WAL (Write-Ahead Log) is the primary replication and recovery mechanism**
* Replication happens **between database instances**
* Each instance maintains its **own independent copy of data files**

---

### Step-by-step write flow (INSERT / UPDATE / DELETE)

1. Application sends a write query to the **primary DB instance**
2. PostgreSQL:

   * Modifies pages in **shared buffers**
   * Generates **WAL records**
3. WAL is:

   * Written to local storage
   * Flushed to disk on commit
4. WAL is **streamed asynchronously** to read replicas
5. Each replica:

   * Receives WAL
   * Replays WAL
   * Updates its **own local data files**

---

### Commit durability guarantee (RDS)

* Transaction is considered **committed** when:

  * WAL is safely written on the primary
  * (If Multi-AZ) WAL is synchronously written to the standby instance

---

### Diagram: RDS PostgreSQL Write Flow

```
Application
     |
     v
Primary DB Instance
  ├── Update shared buffers
  ├── Generate WAL
  ├── Flush WAL to disk
  |
  ├── WAL streaming ──────────────► Read Replica 1
  |                                  ├── Replay WAL
  |                                  └── Update local data files
  |
  └── WAL streaming ──────────────► Read Replica 2
                                     ├── Replay WAL
                                     └── Update local data files
```

---

### Implications (RDS)

* Replication lag can grow under heavy write load
* Replicas consume CPU and I/O to replay WAL
* Failover requires:

  * Promoting standby
  * Applying remaining WAL
  * Re-establishing client connections

---

## 2 Write & Commit Flow – Aurora PostgreSQL

### High-level idea

* Aurora **decouples compute from storage**
* Replication happens **inside the storage layer**, not between DB instances
* **Database instances do NOT ship WAL to each other**
* Aurora still uses WAL internally for PostgreSQL correctness, **but not for replication**

---

### Step-by-step write flow (INSERT / UPDATE / DELETE)

1. Application sends a write query to the **writer instance**
2. PostgreSQL engine:

   * Modifies pages in shared buffers
   * Generates **Aurora redo records**
     *(a lightweight, storage-specific form of WAL information)*
3. Writer sends redo records to **Aurora distributed storage**
4. Storage layer:

   * Applies changes to database pages
   * Replicates data across **3 AZs (6 copies)**
   * Confirms durability using a quorum-based protocol
5. Transaction commits
6. Reader instances immediately see the change by reading from the **same shared storage**

---

### Commit durability guarantee (Aurora)

* Transaction is considered **committed** when:

  * Redo records are durably written to a quorum of storage nodes
* No replica acknowledgment is required
* No WAL replay occurs on reader instances

---

### Diagram: Aurora PostgreSQL Write Flow

```
Application
     |
     v
Writer DB Instance
  ├── Update shared buffers
  ├── Generate Aurora redo
  |
  └── Send redo ───────────────► Aurora Distributed Storage
                                  ├── Apply page updates
                                  ├── Replicate across 3 AZs
                                  └── Quorum acknowledgment

Reader DB Instances
  └── Read latest pages directly from shared storage
```

---

## 3 Key Difference Highlight (WAL vs Aurora Redo)

| Aspect                  | RDS PostgreSQL  | Aurora PostgreSQL       |
| ----------------------- | --------------- | ----------------------- |
| Primary log type        | PostgreSQL WAL  | Aurora redo             |
| WAL shipped to replicas | Yes             | No                      |
| Replicas replay logs    | Yes             | No                      |
| Replication location    | Database engine | Storage layer           |
| Storage ownership       | Per instance    | Shared across instances |
| Replication latency     | Seconds         | Sub-100 ms              |

> **Important clarification:**
> Aurora still uses WAL internally for PostgreSQL transaction semantics and crash recovery, but **WAL is not used for replication between instances**.

---

## 4 Failover Flow Comparison

### RDS PostgreSQL Failover

1. Primary instance fails
2. AWS promotes standby
3. Standby applies remaining WAL
4. DNS endpoint updates
5. Clients reconnect

⏱ **Typical time:** 60–120 seconds

---

### Aurora PostgreSQL Failover

1. Writer instance fails
2. Aurora detects failure
3. A reader instance is promoted to writer
4. Storage remains unchanged and already consistent
5. Clients reconnect

⏱ **Typical time:** ~30 seconds

---

### Diagram: Failover Comparison

#### RDS

```
Primary fails
   ↓
Standby promoted
   ↓
WAL replay / recovery
   ↓
Clients reconnect
```

#### Aurora

```
Writer fails
   ↓
Reader promoted
   ↓
Same shared storage
   ↓
Clients reconnect
```

---

## 5 Why This Matters in Real Systems

### Aurora advantages explained by the flow

* No WAL replay on replicas → near-zero replication lag
* Shared storage → faster and simpler failover
* Redo is smaller than WAL → faster commits
* Storage-level replication → higher durability

### RDS trade-offs

* Pure PostgreSQL behavior
* Simple and predictable architecture
* Higher replication and failover overhead at scale

---

## 6 Final One-Line Comparison

> **RDS PostgreSQL replicates by shipping and replaying WAL between database instances, whereas Aurora PostgreSQL applies redo directly to a shared, distributed storage layer that all instances read from.**

---
