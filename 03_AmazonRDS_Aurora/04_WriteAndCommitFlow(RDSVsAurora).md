## 14. Step-by-Step Write and Commit Flow (RDS vs Aurora)

This section explains **what actually happens when an application writes data** and how **commit, durability, replication, and failover** differ between RDS PostgreSQL and Aurora PostgreSQL.

---

## 14.1 Write & Commit Flow – RDS PostgreSQL

### High-level idea

* RDS follows **standard PostgreSQL behavior**
* WAL is the source of truth
* Replication happens **between database instances**

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
   * Updates its own data files

---

### Commit durability guarantee (RDS)

* Transaction is considered **committed** when:

  * WAL is safely written on the primary
  * (If Multi-AZ) WAL is synchronously written to the standby

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
* Replicas consume CPU to replay WAL
* Failover requires:

  * Promoting standby
  * Re-establishing connections
  * Possible delay due to WAL sync

---

## 14.2 Write & Commit Flow – Aurora PostgreSQL

### High-level idea

* Aurora separates **compute** and **storage**
* Replication happens **inside the storage layer**
* Database instances never ship WAL to each other

---

### Step-by-step write flow (INSERT / UPDATE / DELETE)

1. Application sends a write query to the **writer instance**
2. PostgreSQL engine:

   * Modifies pages in shared buffers
   * Generates **Aurora redo records**
3. Writer sends redo records to **Aurora distributed storage**
4. Storage layer:

   * Applies changes to database pages
   * Replicates data across **3 AZs (6 copies)**
   * Confirms durability using quorum
5. Transaction commits
6. Reader instances immediately see the change by reading from storage

---

### Commit durability guarantee (Aurora)

* Transaction is considered **committed** when:

  * Redo is durably written to a quorum of storage nodes
* No replica acknowledgment is required

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
  └── Read latest pages directly from storage
```

---

## 14.3 Key Difference Highlight (WAL vs Aurora Redo)

| Aspect               | RDS PostgreSQL | Aurora PostgreSQL |
| -------------------- | -------------- | ----------------- |
| What is shipped      | WAL            | Aurora redo       |
| Shipped to replicas  | Yes            | No                |
| Replicas replay logs | Yes            | No                |
| Replication location | DB engine      | Storage layer     |
| Replication latency  | Seconds        | Sub-100 ms        |

---

## 14.4 Failover Flow Comparison

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
3. A reader is promoted to writer
4. Storage remains unchanged
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
WAL sync / recovery
   ↓
Clients reconnect
```

#### Aurora

```
Writer fails
   ↓
Reader promoted
   ↓
Same storage, no sync
   ↓
Clients reconnect
```

---

## 14.5 Why This Matters in Real Systems

### Aurora advantages explained by the flow

* No WAL replay = near-zero replica lag
* Shared storage = faster failover
* Redo is smaller than WAL = faster commits
* Storage replication = higher durability

### RDS trade-offs

* Simpler, pure PostgreSQL model
* Predictable behavior
* More replication overhead at scale

---

## 14.6 Final One-Line Comparison

> **RDS PostgreSQL replicates by shipping logs between databases; Aurora PostgreSQL replicates by applying changes directly in a shared, distributed storage system.**
