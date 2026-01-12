# Amazon Aurora Distributed Storage – Quorum, WAL, and Crash Recovery (Deep Dive)

This lesson focuses on **critical internal mechanisms** of Amazon Aurora’s distributed storage system. It explains **why traditional distributed storage techniques fail at scale** and **how Aurora solves those problems** using **quorum-based writes, optimized reads, continuous recovery, and storage-level WAL processing**.

This content is **critical** for understanding Aurora performance, durability, and crash recovery behavior.

---

## 1. Challenges in Distributed Storage Systems

Aurora storage is **distributed by design**, which introduces challenges that do not exist in single-node or attached-storage databases.

### 1.1 Inherent Challenges

1. **Component Failures**

   * Storage nodes can fail
   * Network links between DB instance and storage nodes can fail or slow down
   * Entire Availability Zones (AZs) can become unavailable

2. **Performance Variability**

   * Network latency can vary
   * One slow node can impact the entire write path
   * Traditional synchronous replication amplifies tail latency

Aurora storage consists of **6 storage nodes across 3 AZs**, and the **database instance communicates with them over the network**.

> More components = higher probability of partial or full failure

---

## 2. Why Traditional Synchronous Writes Don’t Work

### 2.1 Traditional Approach

In a traditional synchronous distributed write model:

1. Client sends a WRITE request
2. Database engine sends write requests to **all storage nodes**
3. Database waits until **every node acknowledges** the write
4. Only then is success returned to the client

### 2.2 Problems with This Model

* Database performance depends on the **slowest node**
* Any node failure or network delay blocks writes
* Partial failures cause transaction failures
* Poor tail latency

> Waiting for *all* nodes makes the system fragile and slow

Aurora **explicitly avoids** this design.

---

## 3. Aurora’s Quorum-Based Storage Model

Aurora storage nodes within a **protection group** form a **quorum group**.

### 3.1 Storage Layout Recap

* 6 storage nodes
* Spread across 3 AZs
* 2 nodes per AZ

---

## 4. Quorum Writes (4-of-6)

### 4.1 Write Quorum Rule

* A write is considered **successful when 4 out of 6 nodes acknowledge it**
* The remaining 2 nodes may be:

  * Slow
  * Temporarily unavailable
  * Recovering

> Aurora can tolerate **up to 2 node failures for writes**

### 4.2 Write Flow (Step-by-Step)

1. Client sends a WRITE request
2. Database engine processes SQL
3. Database engine generates **WAL (Write-Ahead Log) records**
4. WAL records are sent to **all 6 storage nodes**
5. Database engine enters a **non-blocking wait state**
6. Storage nodes:

   * Apply WAL records directly to data pages
   * Do NOT write to local WAL files
7. As soon as **4 acknowledgements** are received:

   * Database returns **COMMIT SUCCESS** to client

### 4.3 Why This Is Fast

* WAL records are **hundreds of bytes**
* Traditional PostgreSQL writes full **8 KB pages**
* Network bandwidth usage is dramatically reduced

> Smaller payloads + quorum = high write throughput

---

## 5. Continuous WAL Application (No WAL Files)

### 5.1 Key Difference from Community PostgreSQL

| PostgreSQL (Community) | Aurora                    |
| ---------------------- | ------------------------- |
| WAL written to disk    | No WAL files              |
| Pages updated later    | Pages updated immediately |
| Requires checkpoints   | No checkpoints            |

### 5.2 Continuous Recovery Mode

* Storage nodes **apply WAL records immediately**
* Data pages are always up to date
* Aurora is effectively **always in recovery mode**

### 5.3 Implications

* **No checkpoints** required
* **Crash recovery is extremely fast**
* No backlog of WAL records to replay

---

## 6. Quorum Reads (3-of-6) – When They Are Used

### 6.1 Quorum Read Rule

* Read quorum = **3 of 6 nodes** returning consistent data
* Aurora can tolerate **up to 3 node failures for reads**

### 6.2 Why Quorum Reads Are Expensive

1. DB instance requests data from all 6 nodes
2. Receives responses from at least 3 nodes
3. Performs **consistency checks**
4. Returns data to client

> High CPU and network overhead

---

## 7. Optimized Reads Using Internal Bookkeeping

Aurora **does NOT** use quorum reads for normal SELECT queries.

### 7.1 Internal Storage Bookkeeping

Aurora maintains metadata tracking:

* How up-to-date each node is
* Node latency
* Node health

### 7.2 Optimized Read Path

1. SELECT request arrives
2. Aurora consults bookkeeping metadata
3. Chooses the **best, most up-to-date node**
4. Fetches data from **a single node**
5. Returns result to client

> No cross-node comparison → minimal overhead

### 7.3 When Quorum Reads Are Used

* Crash recovery
* When node state metadata is unavailable

---

## 8. Crash Recovery Behavior

### 8.1 Why Recovery Is Fast

* No WAL files to replay
* Pages already updated
* Storage nodes already consistent

### 8.2 Recovery Read Strategy

* Aurora uses **3-of-6 quorum reads**
* Reconstructs consistent database state
* Resumes optimized read path after recovery

---

## 9. Continuous, Storage-Level Backups

### 9.1 How Backups Work

* WAL records are:

  * Applied to storage pages
  * **Simultaneously streamed to Amazon S3**

### 9.2 Backup Characteristics

* Incremental
* Asynchronous
* Not user-accessible
* Happens at storage layer

### 9.3 Performance Impact

* **Zero CPU usage** on DB instance
* No I/O impact on queries

---

## 10. Fault Tolerance Summary

| Operation        | Quorum | Failure Tolerance     |
| ---------------- | ------ | --------------------- |
| Writes           | 4 of 6 | Up to 2 node failures |
| Reads (recovery) | 3 of 6 | Up to 3 node failures |
| Reads (normal)   | 1 node | Optimized selection   |

---

## 11. Key Takeaways (Critical)

* Aurora avoids traditional synchronous replication
* Uses **quorum-based writes (4/6)** for performance and durability
* Uses **quorum reads (3/6) only during recovery**
* WAL records are small and applied directly to pages
* **No WAL files, no checkpoints**
* Storage is always in **continuous recovery mode**
* Crash recovery is fast and predictable
* Backups are incremental and storage-managed
* Database CPU is not consumed by backups or recovery

---

### Final Memory Hook

> “Aurora writes logs, not pages — commits at 4, reads at 3 only when recovering.”
