##  Step-by-Step Read Query Flow (RDS vs Aurora)

This section explains **how SELECT queries are executed** and how **read consistency, caching, and replication** differ between RDS PostgreSQL and Aurora PostgreSQL.

It also clarifies **shared buffer behavior** and why Aurora readers can see fresh data without WAL shipping.

---

### 1 RDS PostgreSQL Read Flow

#### High-Level Idea

* Each database instance (primary or replica) maintains **its own shared buffers** and **local storage (EBS)**.
* Replication occurs via **WAL shipping and replay**.
* A read from a replica **may not see the latest committed data** if WAL replay is delayed.
* PostgreSQL **always loads pages from storage into shared buffers** when they are not already cached.

---

#### Step-by-Step Read Flow

1. Application sends a SELECT query.
2. Query is routed to:

   * Primary instance **or**
   * Read replica
3. PostgreSQL checks **shared buffers** (instance memory cache):

   * **Cache hit:** page is in memory → return rows immediately
   * **Cache miss:** page is not in memory → fetch from **local EBS storage**
4. Page is **loaded into shared buffers** for future reads.
5. Query results are returned to the application.

---

#### Diagram: RDS PostgreSQL Read Flow

```
Application
     |
     v
Primary / Read Replica
  ├── Check shared buffers (memory cache)
  |      ├── Cache hit → return rows
  |      └── Cache miss
  |            ├── Read from local EBS storage
  |            └── Load page into shared buffers
  |
  └── Return result to application
```

---

#### Implications

* **Read freshness:** Replicas may lag due to WAL replay, so reads may not reflect the latest writes.
* **CPU / I/O usage:** Each replica must replay WAL to maintain consistency.
* **Caching:** Shared buffers improve repeated read performance, but are **per instance**, not shared across replicas.

---

### 2 Aurora PostgreSQL Read Flow

#### High-Level Idea

* Aurora separates **compute from storage**.
* All instances (writer and readers) share the **same distributed storage**.
* Replication happens **inside storage**, so **no WAL shipping or replay** is needed for readers.
* Each instance still maintains a **local buffer cache** for performance.

---

#### Step-by-Step Read Flow

1. Application sends a SELECT query.
2. Query is routed to:

   * Writer instance **or**
   * Reader instance
3. PostgreSQL checks **local buffer cache**:

   * **Cache hit:** page is in memory → return rows immediately
   * **Cache miss:** page is not in memory → fetch from **Aurora distributed storage**
4. Page is **loaded into the instance’s buffer cache** for future reads.
5. Query results are returned to the application.

---

#### Diagram: Aurora PostgreSQL Read Flow

```
Application
     |
     v
Reader / Writer Instance
  ├── Check local buffer cache
  |      ├── Cache hit → return rows
  |      └── Cache miss
  |            ├── Read from Aurora Distributed Storage
  |            └── Load page into local buffer cache
  |
  └── Return result to application
```

---

#### Key Advantages

* **Fresh reads:** Readers see the latest committed data almost immediately because all instances share the same storage.
* **No WAL shipping:** Reduces CPU and network overhead.
* **Read scaling:** Adding more reader instances has minimal replication cost.
* **Buffer cache:** Improves performance for repeated reads without affecting other instances.

---

### 3 Read Consistency and Caching Comparison

| Feature         | RDS PostgreSQL              | Aurora PostgreSQL                                |
| --------------- | --------------------------- | ------------------------------------------------ |
| Source of truth | Instance-local storage      | Shared distributed storage                       |
| Buffer cache    | Shared buffers per instance | Local buffer per instance                        |
| Read freshness  | Depends on WAL replay       | Near real-time                                   |
| Replication     | WAL shipped to replicas     | Storage-level replication                        |
| Stale reads     | Possible                    | Extremely rare                                   |
| Scaling reads   | Limited by WAL replay       | Easy to add readers without replication overhead |

---

### 4 Combined Mental Model (Write + Read)

```
RDS PostgreSQL:
Write → WAL → Replica → Replay → Read (may lag)
Aurora PostgreSQL:
Write → Storage → Read (all instances read same storage, no WAL)
```

---

### 5 Key Takeaways

* **RDS replicas** have independent copies of data; reads may lag behind writes.
* **Aurora replicas** read directly from **shared, replicated storage**; reads are almost instant.
* In both RDS and Aurora, **pages read from storage are loaded into the instance buffer cache** for faster subsequent reads.
* Aurora avoids **WAL shipping/replay** overhead because **storage itself is authoritative**.

