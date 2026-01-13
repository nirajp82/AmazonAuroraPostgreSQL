# Amazon Aurora Distributed Storage – Quorum, Redo (XLog), WAL, and Crash Recovery (Deep Dive)

This lesson provides a **deep, internals-level explanation** of Amazon Aurora’s distributed storage architecture. It explains **why traditional distributed storage approaches break down**, and how Aurora solves those problems using **quorum-based writes, optimized reads, continuous redo processing, and storage-level recovery**.

This content is **critical** for understanding Aurora **durability, performance, fault tolerance, and crash recovery**.

---

## 1. Challenges in Distributed Storage Systems

Aurora storage is **distributed by design**, which introduces challenges that do not exist in single-node or attached-storage databases.

### 1.1 Inherent Challenges

1. **Component Failures**

   * Individual storage nodes can fail
   * Network links between DB instance and storage nodes can fail or slow down
   * Entire Availability Zones (AZs) can become unavailable

2. **Performance Variability**

   * Network latency varies across nodes
   * One slow node can delay writes in synchronous systems
   * Tail latency becomes the dominant performance factor

Aurora storage consists of **6 storage nodes across 3 Availability Zones**, and the **database instance communicates with them over the network**.

> More distributed components = higher probability of partial or full failure

<img width="585" height="284" alt="Aurora Distributed Storage Nodes" src="https://github.com/user-attachments/assets/f3b86c43-c150-4128-af5c-9f02821f7a56" />

**Memory Hook:**

> “Distributed storage increases durability — but also increases failure probability.”

---

## 2. Why Traditional Synchronous Distributed Writes Don’t Work

### 2.1 Traditional Synchronous Write Model

In a traditional synchronous distributed database:

1. Client sends a WRITE request
2. Database engine sends write requests to **all storage nodes**
3. Database waits for **every node** to acknowledge
4. Only then is COMMIT returned to the client

### 2.2 Problems With This Model

* Performance depends on the **slowest node**
* A single degraded node delays every transaction
* Partial failures cause full transaction failures
* Poor tail latency and unpredictable throughput

> If **1 out of 6 nodes** is slow or unavailable, **the entire transaction fails**

Aurora **explicitly avoids** this design.

**Memory Hook:**

> “Waiting for everyone makes the system fragile.”

---

## 3. Aurora Storage Architecture Recap

* **6 storage nodes** per protection group
* **3 Availability Zones**
* **2 nodes per AZ**
* Nodes form a **quorum group**

This design balances **durability, availability, and performance**.

---

## 4. Quorum-Based Writes (4-of-6)

### 4.1 Write Quorum Rule

* A write is successful when **4 out of 6 storage nodes** acknowledge it
* Remaining nodes may be:

  * Slow
  * Temporarily unavailable
  * Recovering

> Aurora tolerates **up to 2 storage node failures** for writes

---

## 5. Write Path – Redo (XLog) Processing (Step-by-Step)

This is the **most critical difference** between Aurora and traditional PostgreSQL.

### 5.1 Step-by-Step Write Flow

1. Client sends INSERT / UPDATE / DELETE

2. Database engine parses and executes SQL

3. Database engine generates **Redo / WAL (XLog) records**

   * WAL records are **hundreds of bytes**
   * Traditional PostgreSQL writes **entire 8 KB pages**

4. **Redo (XLog) records are sent to all 6 storage nodes**

5. Database engine enters a **non-blocking wait state**

6. Storage nodes:

   * Apply redo records **directly to data pages**
   * Do **not** write local WAL files

7. As soon as **4 acknowledgements** are received:

   * Database engine returns **COMMIT SUCCESS** to client

<img width="1088" height="588" alt="Aurora Quorum Write Flow" src="https://github.com/user-attachments/assets/3b76e89f-4a0e-4e18-b614-732cc3e0439a" />

**Memory Hook:**

> “Aurora ships redo, not pages — and commits at 4.”

---

## 6. Why Aurora Writes Are Fast

* Redo records are **small** (hundreds of bytes)
* No full-page writes over the network
* No blocking on slow nodes beyond quorum
* Efficient network utilization

> Smaller payloads + quorum = high write throughput

---

## 7. Continuous Redo Application (No WAL Files, No Checkpoints)

### 7.1 Key Difference vs Community PostgreSQL

| Community PostgreSQL | Amazon Aurora             |
| -------------------- | ------------------------- |
| WAL written to disk  | No WAL files              |
| Pages updated later  | Pages updated immediately |
| Requires checkpoints | No checkpoints            |

### 7.2 Continuous Recovery Mode

* Storage nodes apply redo **as soon as it arrives**
* Pages are always crash-consistent
* Aurora storage is **always in recovery mode**

### 7.3 Implications

* No checkpoint overhead
* No WAL replay backlog
* Extremely fast crash recovery

**Memory Hook:**

> “Aurora never needs to catch up — it’s always caught up.”

---

## 8. Quorum Reads (3-of-6) – Used Only When Required

### 8.1 Quorum Read Rule

* Read quorum = **3 of 6 nodes**(Requires 3 of 6 nodes provides consistent data)
* Tolerates **up to 3 node failures**

### 8.2 Why Quorum Reads Are Expensive

1. Request sent to multiple nodes
2. Responses compared for consistency
3. Higher CPU and network overhead

> Correct, but expensive
<img width="1085" height="510" alt="image" src="https://github.com/user-attachments/assets/362ef7ce-b08f-44aa-b4f1-966b3155a974" />

---

## 9. Optimized Read Path Using Internal Bookkeeping

Aurora avoids quorum reads for normal SELECTs.

### 9.1 Internal Bookkeeping Metadata

Aurora tracks:

* Node freshness (LSN progress)
* Node latency
* Node health

### 9.2 Optimized Read Flow

1. SELECT arrives
2. Aurora consults bookkeeping metadata
3. Chooses **best up-to-date node**
4. Reads from **a single node**
5. Returns result

> Minimal overhead, consistent results
<img width="1127" height="612" alt="image" src="https://github.com/user-attachments/assets/c920d5ba-6f96-479f-a5bc-3e1e7b628cec" />

### 9.3 When Quorum Reads Are Used

* Crash recovery
* When node state metadata is unavailable

---

## 10. Crash Recovery Behavior

### 10.1 Why Recovery Is Fast

* No WAL replay required
* Pages already updated
* Storage nodes already consistent

### 10.2 Recovery Read Strategy

* Uses **3-of-6 quorum reads**
* Rebuilds internal state
* Switches back to optimized reads

---

## 11. Continuous, Storage-Level Backups

### 11.1 Backup Flow

* Redo records:

  * Applied to storage pages
  * **Simultaneously streamed to Amazon S3**

### 11.2 Backup Characteristics

* Incremental
* Asynchronous
* Transparent to users
* Not user-accessible

### 11.3 Performance Impact

* Zero CPU usage on DB instance
* No query I/O impact

---

## 12. Fault Tolerance Summary

| Operation        | Quorum | Failure Tolerance     |
| ---------------- | ------ | --------------------- |
| Writes           | 4 of 6 | Up to 2 node failures |
| Reads (recovery) | 3 of 6 | Up to 3 node failures |
| Reads (normal)   | 1 node | Optimized selection   |

---

## 13. Key Takeaways (Critical)

* Aurora avoids traditional synchronous replication
* Uses **4/6 quorum writes** for durability + performance
* Ships **Redo (XLog), not data pages**
* Storage nodes apply redo immediately
* **No WAL files, no checkpoints**
* Storage is always in **continuous recovery mode**
* Crash recovery is fast and predictable
* Backups are incremental and storage-managed

---

### Final Memory Hook

> “Aurora writes redo, commits at 4, reads at 1 — and only reads at 3 when recovering.”
