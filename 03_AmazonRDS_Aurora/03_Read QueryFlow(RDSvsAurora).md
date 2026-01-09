## 15. Read Query Flow (RDS vs Aurora)

This section explains **how SELECT queries are served**, how **read replicas work**, and why **Aurora read latency and consistency behave differently** from RDS PostgreSQL.

---

## 15.1 Read Query Flow – RDS PostgreSQL

### High-level idea

* Each instance (primary or replica) has:

  * Its **own buffer cache**
  * Its **own data files**
* Replicas serve reads **only after WAL replay**
* Read consistency depends on **replication lag**

---

### Step-by-step read flow (SELECT)

1. Application sends a SELECT query
2. Query is routed to:

   * Primary instance **or**
   * A read replica
3. PostgreSQL checks:

   * Shared buffers (memory cache)
4. If page is not cached:

   * Reads from **local EBS storage**
5. Query returns results

---

### Diagram: RDS PostgreSQL Read Flow

```
Application
     |
     v
Read Replica (or Primary)
  ├── Check shared buffers
  |      ├── Cache hit → return rows
  |      └── Cache miss
  |
  └── Read from local EBS
         └── Data files owned by this instance
```

---

### Important behavior in RDS

* Each replica:

  * Has its **own cache**
  * Has its **own disk I/O**
* Recently committed writes:

  * May **not be visible** yet on replicas
* Stale reads are possible if:

  * WAL replay is delayed

---

## 15.2 Read Query Flow – Aurora PostgreSQL

### High-level idea

* All DB instances:

  * Share **the same distributed storage**
* Only caches are per-instance
* Reads are always from **the same source of truth**

---

### Step-by-step read flow (SELECT)

1. Application sends SELECT query
2. Query is routed to:

   * Writer instance **or**
   * Any reader instance
3. PostgreSQL checks:

   * Local buffer cache
4. If page is not cached:

   * Reads from **Aurora distributed storage**
5. Query returns results

---

### Diagram: Aurora PostgreSQL Read Flow

```
Application
     |
     v
Reader or Writer Instance
  ├── Check local buffer cache
  |      ├── Cache hit → return rows
  |      └── Cache miss
  |
  └── Read from Aurora Distributed Storage
         └── Same data pages for all instances
```

---

## 15.3 Read Consistency Comparison

| Aspect                 | RDS PostgreSQL         | Aurora PostgreSQL          |
| ---------------------- | ---------------------- | -------------------------- |
| Source of data         | Instance-local storage | Shared distributed storage |
| Read replica freshness | Depends on WAL replay  | Near-real-time             |
| Stale reads possible   | Yes                    | Extremely rare             |
| Cache ownership        | Per instance           | Per instance               |
| Disk ownership         | Per instance           | Shared                     |

---

## 15.4 Why Aurora Reads Are Faster and More Consistent

### RDS Reads

* Replicas may lag
* Each replica must:

  * Replay WAL
  * Maintain its own data files
* Cache warm-up happens **per replica**

---

### Aurora Reads

* No WAL replay
* Storage already has latest committed data
* Readers immediately see changes
* Adding a reader does not increase replication load

---

## 15.5 Read Scaling Behavior

### RDS PostgreSQL

* Adding replicas:

  * Increases WAL shipping overhead
  * Increases replay cost
* Practical limits under heavy writes

---

### Aurora PostgreSQL

* Adding replicas:

  * Only adds compute
  * Does not increase replication overhead
* Scales reads horizontally with minimal impact

---

## 15.6 Read Flow Summary (One-liner)

> **RDS replicas read their own copies of data after replaying WAL, while Aurora readers read the same shared, already-replicated storage.**

---

## 15.7 Combined Mental Model (Write + Read)

```
RDS PostgreSQL:
  Write → WAL → Replica → Replay → Read

Aurora PostgreSQL:
  Write → Storage → Read (no replay)
```
