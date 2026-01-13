# Amazon Aurora Storage Internals

## Quorum, Protection Groups, Redo Processing, Memory Hooks, and Crash Recovery (Deep Dive)

This README provides a **deep, system-level explanation** of how Amazon Aurora works internally.
It focuses on **critical design decisions** that differentiate Aurora from traditional PostgreSQL and explain **why Aurora delivers high performance, durability, and fast recovery at scale**.

This is **core knowledge** for production systems, troubleshooting, and architecture decisions.

---

## 1. Aurora Storage Philosophy (High-Level)

Aurora is **not** a traditional database with attached storage.

Instead:

* Compute (DB instance) and storage are **fully decoupled**
* Storage is **distributed, peer-based, and quorum-driven**
* Aurora writes **redo logs, not data pages**
* Durability is achieved through **consensus, not replication**

> Aurora treats storage as a **distributed system**, not a disk.

---

## 2. Storage Node (Smallest Building Block)

A **storage node** is the smallest physical unit in Aurora storage.

Each storage node:

* Stores a **subset of database pages**
* Maintains its **own local copy** of those pages
* Receives **redo / XLog / WAL records** from the DB instance
* Applies redo logs **directly to data pages**
* Does **not** maintain WAL files on disk
* Can fail or slow down **without stopping the database**

Key points:

* Aurora does **not ship 8 KB PostgreSQL pages**
* Only **redo records (hundreds of bytes)** are sent
* Nodes are designed to be **easily rebuilt**

> A storage node is a **mini storage engine** that understands redo, not SQL.
<img width="1478" height="912" alt="image" src="https://github.com/user-attachments/assets/bd779a40-8892-41d7-89b3-d7f11aea72c1" />

---

## 3. Peer (Role of a Storage Node)

In Aurora, **every storage node is a peer**.

### What “Peer” Means

* No primary storage node
* No secondary storage node
* No leader election
* No promotion or failover at storage layer
* All nodes have **equal authority**

Important clarification:

* **Availability Zones are NOT peers**
* **Nodes are peers**
* AZs are only a placement boundary

> Peers are **equals that vote**, not replicas that follow.

---

## 4. Protection Group (Unit of Durability)

A **Protection Group** is a logical grouping of **exactly 6 peer storage nodes** that together protect the **same data segment**.

### Layout

* 6 peers total
* Spread across **3 Availability Zones**
* **2 peers per AZ**

Example:

```
AZ-1: Peer A, Peer B
AZ-2: Peer C, Peer D
AZ-3: Peer E, Peer F
```

Important details:

* All 6 peers store the same data segment
* Data is **striped across many protection groups**
* Each protection group operates **independently**

> A protection group is the **blast-radius boundary** in Aurora.

---

## 5. Quorum Group (How the 6 Peers Act as One)

The 6 peers inside a protection group collectively form a **quorum group**.

### What a Quorum Group Means

* Not all nodes must respond
* Only a **minimum number (quorum)** must agree
* System continues operating during partial failures

Aurora avoids traditional synchronous replication because:

* Waiting for all replicas increases tail latency
* One slow node can block the system
* Network variability makes “all-or-nothing” fragile

> Aurora waits for **enough**, not for **everyone**.

---

## 6. Quorum Rules (Critical)

| Operation        | Quorum     | Purpose                      |
| ---------------- | ---------- | ---------------------------- |
| Writes           | **4 of 6** | Commit durability            |
| Reads (Recovery) | **3 of 6** | Reconstruct consistent state |
| Reads (Normal)   | **1 node** | Optimized performance        |

---

## 7. Memory Hooks & Buffer Cache Interaction

Aurora modifies the PostgreSQL storage engine using **memory hooks**.

### What Changes in Aurora

In traditional PostgreSQL:

* WAL is written to disk
* Pages are written later via checkpoints
* Crash recovery replays WAL

In Aurora:

* WAL is **never written to local disk**
  - * WAL records are still generated (redo/XLog) - “Aurora uses WAL logic, not WAL files”)
  - * But they are:
      - Generated in memory
      - Sent directly to storage nodes
      - Storage nodes apply redo immediately → pages always current
      - Streamed to S3 for backups
      - Checkpoints are not needed
      - Crash recovery is fast
* Redo is shipped directly from memory to storage
* Storage nodes apply redo immediately

### Shared Buffers Role

* Shared buffers still cache data pages in memory
* Pages are marked dirty **logically**, not physically
* Durability is guaranteed **once redo is acknowledged by quorum**

> Aurora’s buffer cache is about **performance**, not durability.

---

## 8. End-to-End Redo (XLog) Flow

### DB Memory → Storage → Backup

### Step-by-Step Write Flow

1. Client sends INSERT / UPDATE / DELETE
2. DB engine processes SQL
3. Redo (XLog/WAL) records are generated **in memory**
4. Redo records are sent to **all 6 peers**
5. Each peer:

   * Applies redo directly to data pages
   * Does NOT write WAL files
6. DB engine waits non-blocking
7. As soon as **4 peers acknowledge**:

   * Commit succeeds
   * Client receives success
8. Remaining peers catch up asynchronously

### Backup Integration

* The **same redo stream** is:

  * Applied to storage pages
  * Streamed asynchronously to **Amazon S3**
* Backups are:

  * Continuous
  * Incremental
  * Storage-managed
  * Zero DB CPU overhead

> Aurora backups are a **side effect of redo**, not a separate job.

---

## 9. Read Paths (Normal vs Recovery)

### Normal Reads (Optimized Path)

* Aurora maintains internal metadata:

  * Node freshness
  * Node latency
  * Node health
* For SELECT queries:

  * Aurora chooses **one best node**
  * Reads from a single peer
  * No quorum required

> Most reads hit **one node only**.

---

### Recovery Reads (Quorum Reads)

Used only when:

* DB restart
* Crash recovery
* Metadata is unavailable

Process:

1. DB requests data from peers
2. Reads from **any 3 of 6**
3. Compares responses
4. Reconstructs consistent state

> Quorum reads are expensive — used only when necessary.

---

## 10. Crash Recovery & Fast Restart

### Why Aurora Recovery Is Fast

* No WAL files to replay
* Pages already updated at storage layer
* Storage is always in **continuous recovery mode**

### Recovery Process

1. DB instance restarts
2. Performs **3-of-6 quorum reads**
3. Rebuilds in-memory state
4. Resumes optimized reads immediately

Result:

* Recovery time = seconds
* Independent of database size

---

## 11. Failure Handling & Self-Healing

### Node Failure

* Quorum still holds
* Writes continue
* Reads continue

### AZ Failure

* 2 peers lost
* 4 peers remain
* Write quorum still satisfied

### Healing

1. Failure detected
2. New node created
3. Healthy peers stream:

   * Missing redo
   * Required pages
4. Node rejoins group

All without downtime.

---

## 12. Aurora vs RDS PostgreSQL (Critical Comparison)

| Area        | RDS PostgreSQL    | Aurora PostgreSQL          |
| ----------- | ----------------- | -------------------------- |
| Storage     | Attached EBS      | Distributed                |
| WAL         | Written to disk   | Never written locally      |
| Writes      | Page-based        | Redo-based                 |
| Replication | Physical replicas | Quorum consensus           |
| Failover    | Replica promotion | Storage already consistent |
| Recovery    | WAL replay        | Quorum reads               |
| Backups     | Snapshot-based    | Continuous redo            |

> Aurora is **not PostgreSQL on EBS** — it is a **distributed database system**.

---

## 13. Mental Models (Memory Hooks)

* **Node** → Worker
* **Peer** → Equal voter
* **Protection Group** → Team of 6 guarding data
* **Quorum** → Minimum votes to move forward
* **Redo** → Source of truth
* **Pages** → Derived state

### One-Line Memory Hook

> **“Aurora writes logs, not pages — commits at 4, recovers at 3.”**

---

## 14. Final One-Line Summary

> **Amazon Aurora stores data using peer-based storage nodes grouped into 6-node protection groups. Each protection group forms a quorum group that achieves durability and consistency through redo-based consensus (4/6 for writes, 3/6 for recovery reads), enabling high performance, fast crash recovery, and fault tolerance without traditional replication.**

### FAQ
---

## Q1. What is the smallest durable storage unit in Amazon Aurora?

**Answer:**
A **storage node**.

Each storage node:

* Stores a subset of database pages
* Applies redo (XLog) records directly
* Operates independently
* Participates as a peer in quorum decisions

> Durability is not provided by the DB instance or AZ — it starts at the **storage node level**.

---

## Q2. What does it mean when Aurora says “nodes are peers”?

**Answer:**
It means:

* No node is primary
* No node is secondary
* All nodes have equal authority
* Any node can participate in reads, repair, or recovery

There is **no leader election** and **no promotion** at the storage layer.

**Exam Trap:**
AZs are **not peers** — *nodes* are peers.

---

## Q3. What is a Protection Group?

**Answer:**
A **Protection Group** is a logical group of **exactly 6 peer storage nodes** that together protect the **same data segment**.

Characteristics:

* 6 peers
* Spread across 3 AZs
* 2 peers per AZ
* Independent durability boundary

> A protection group is the **unit of fault isolation and durability**.

---

## Q4. Do all 6 nodes need to acknowledge a write?

**Answer:**
No.

Aurora uses **quorum-based writes**.

* **4 of 6 acknowledgements** = commit success
* Remaining nodes can be slow or unavailable

This avoids tail latency and blocking writes.

---

## Q5. How many node failures can Aurora tolerate?

**Answer:**

| Operation      | Failure Tolerance      |
| -------------- | ---------------------- |
| Writes         | Up to **2 nodes**      |
| Recovery Reads | Up to **3 nodes**      |
| Normal Reads   | Depends on node health |

> Aurora can lose an entire AZ and still continue writes.

---

## Q6. What exactly is written during an Aurora write?

**Answer:**
Only **redo (XLog/WAL) records**, not data pages.

* Redo records are typically **hundreds of bytes**
* PostgreSQL pages are **8 KB**
* Redo is shipped directly from memory

> Aurora writes **logs, not pages**.

---

## Q7. Where are WAL files stored in Aurora?

**Answer:**
They are **not stored at all**.

Key differences:

* No local WAL files
* No WAL replay
* No checkpoints

Redo is:

* Generated in memory
* Shipped to storage nodes
* Applied immediately

---

## Q8. Why does Aurora not need checkpoints?

**Answer:**
Because:

* Redo is applied immediately to data pages
* There is no backlog of WAL to flush
* Storage is always in a **continuous recovery state**

> Checkpoints exist to flush dirty pages — Aurora never accumulates them.

---

## Q9. How does Aurora perform backups?

**Answer:**
Backups are a **side effect of redo processing**.

* Redo records are:

  * Applied to storage pages
  * Asynchronously streamed to Amazon S3
* Backups are:

  * Incremental
  * Continuous
  * Storage-level
  * Zero DB CPU cost

---

## Q10. Does Aurora use quorum reads for every SELECT?

**Answer:**
No.

### Normal SELECT:

* Uses **internal bookkeeping**
* Reads from **one best node**

### Quorum Reads (3 of 6):

* Used only during:

  * Crash recovery
  * Metadata rebuild
  * Uncertain node state

> Quorum reads are expensive and avoided during normal operation.

---

## Q11. What is Aurora’s internal bookkeeping used for?

**Answer:**
To track:

* Node freshness
* Node latency
* Node health

This allows Aurora to:

* Select the best node for reads
* Avoid cross-node comparison
* Optimize performance

---

## Q12. How does crash recovery work in Aurora?

**Answer:**

1. DB instance restarts
2. Uses **3-of-6 quorum reads**
3. Reconstructs in-memory state
4. Resumes optimized read path

Why it’s fast:

* No WAL replay
* Pages already updated
* Storage already consistent

---

## Q13. What happens if a storage node fails?

**Answer:**

* Quorum still holds
* Writes continue
* Reads continue
* A new node is automatically created
* Healthy peers stream missing data

No downtime. No user action.

---

## Q14. How is Aurora different from PostgreSQL replication?

**Answer:**

| PostgreSQL         | Aurora              |
| ------------------ | ------------------- |
| Primary + replicas | Peer-based          |
| Page shipping      | Redo shipping       |
| Replica lag        | Quorum commit       |
| Promotion required | No promotion        |
| WAL replay         | Continuous recovery |

> Aurora does not replicate databases — it coordinates **consensus**.

---

## Q15. Can Aurora survive a full AZ failure?

**Answer:**
Yes.

Each AZ contains **only 2 of the 6 peers**.

If one AZ fails:

* 2 peers lost
* 4 peers remain
* Write quorum still satisfied

---

## Q16. Why is Aurora write latency low despite networked storage?

**Answer:**

* Redo records are small
* Quorum avoids waiting for slow nodes
* Non-blocking write path
* No page shipping

> Aurora removes **disk latency** and **tail latency** from the critical path.

---

## Q17. Where does durability actually come from in Aurora?

**Answer:**
From:

* Quorum consensus (4/6)
* Multiple AZ placement
* Storage-level redo persistence

Not from:

* DB instance
* Buffer cache
* WAL files

---

## Q18. True or False: Aurora storage nodes replicate from a primary node.

**Answer:**
❌ **False**

Nodes are peers.
There is no primary storage node.

---

## Q19. True or False: Aurora backups increase database CPU usage.

**Answer:**
❌ **False**

Backups happen at the storage layer using redo streams.

---

## Q20. One-Line Exam Memory Hook

> **“Aurora achieves durability through quorum consensus on redo logs, not through page replication or synchronous storage writes.”**


