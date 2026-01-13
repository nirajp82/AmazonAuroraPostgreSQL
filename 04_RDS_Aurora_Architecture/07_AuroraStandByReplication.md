# Amazon Aurora Replication & Failure Handling

## Storage, Buffer Cache, Quorum, and Availability Explained

---

## Introduction: Why Fault Tolerance Is Fundamental

Like any enterprise-grade database or database cluster, **Amazon Aurora is designed with the assumption that failures happen all the time**.

Aurora follows the same design philosophy as other AWS services:

> **Assume failure is constant, not exceptional.**

Failures may be:

* Isolated (single storage node)
* Independent and simultaneous (multiple nodes across AZs)
* Correlated (entire Availability Zone failure)
* Prolonged (natural disasters, long-term AZ outages)

The goal of Aurora’s architecture is to **remain available and durable even under these conditions**.

---

## Aurora Storage Architecture Recap (Critical Foundation)

Aurora storage is built from:

* **Storage Nodes** (smallest unit)
* Grouped into **6-node Protection Groups**
* Spread across **3 Availability Zones (2 nodes per AZ)**
* All nodes are **peers**
* Data consistency is enforced via **quorum**, not replication

### Quorum Rules

| Operation              | Required Nodes |
| ---------------------- | -------------- |
| Write quorum           | **4 of 6**     |
| Read quorum (recovery) | **3 of 6**     |
| Normal reads           | **1 node**     |

This design allows Aurora to **ignore failed or slow nodes** without blocking progress.

---

## Why Aurora Still Needs Replication (Despite Shared Storage)

Aurora storage is **shared across all DB instances**, so:

* There is **no need for disk-level replication**
* Storage nodes already replicate data internally
* WAL / redo logs are applied directly at the storage layer

So why replicate at all?

### The Real Problem: Buffer Cache Inconsistency
<img width="1097" height="506" alt="image" src="https://github.com/user-attachments/assets/948fa63f-b160-4cd6-9ade-bfc013ad4c35" />

Each Aurora DB instance (primary and replicas):

* Has its **own private buffer cache**
* Does **not share memory** with other instances

This means:

* Storage may be correct
* But replicas may still hold **stale data in memory**

---

## Aurora Replication = Buffer Cache Replication

Aurora replication exists **only** to keep **buffer caches consistent**.

### What Aurora Replication Does

* Propagates **page-level changes**
* From primary buffer cache
* To replica buffer caches
* Without disk I/O

### What It Does NOT Do

* ❌ No WAL replay on replicas
* ❌ No disk writes on replicas
* ❌ No storage replication

> **Aurora replicates memory state, not storage state.**

---

## End-to-End Write Flow (With Replication)

1. Client sends `INSERT / UPDATE / DELETE`
2. Primary DB instance:

   * Generates redo (XLog/WAL)
3. Redo is sent to **all 6 storage nodes**
4. Storage commits using **4-of-6 quorum**
5. Primary updates its buffer cache
6. Aurora replication:

   * Sends page changes to replicas
7. Replica updates its buffer cache
8. Replicas become **eventually consistent**

---

## Replication Lag in Aurora

Aurora replication is **asynchronous**, so lag is possible.

### Typical Characteristics

* Usually **< 100 ms**
* Can increase during:

  * Heavy write load
  * Network latency between AZs
  * Long-running transactions
  * Smaller replica instance sizes

### Important Behavior

* If lag becomes excessive:

  * Aurora may **restart the replica automatically**
* This is **expected behavior**, not a failure

---

## Best Practices to Control Replication Lag

* Use **same instance class** for primary and replicas
* Avoid long-running transactions
* Write in smaller batches
* Monitor replica lag continuously
* Define acceptable lag (e.g., 500 ms)

---

# Failure + Replication Interaction Scenarios

This section explains **how quorum and replication behave together** during failures.

---

## Scenario 1: Independent Node Failures (Up to 2 Nodes)

**Failure:**

* 1–2 storage nodes fail
* Nodes may be in same or different AZs

**Result:**

* ✅ Write quorum (4/6) still met
* ✅ Read quorum (3/6) still met
* ✅ Replication continues normally
* ✅ Database fully available (read + write)

---

## Scenario 2: Complete AZ Failure (2 Nodes Lost)

**Failure:**

* Entire Availability Zone goes down
* 2 storage nodes lost simultaneously

**Result:**

* ✅ Write quorum still met (4/6)
* ✅ Read quorum still met
* ✅ Replication continues
* ✅ Database fully available

---

## Scenario 3: AZ Failure + One Additional Node (AZ + 1 Model)

This is the **maximum fault tolerance model** for Aurora storage.

**Failure:**

* 1 Availability Zone lost (2 nodes)
* 1 additional node lost in another AZ
* Total nodes available = **3**

**Result:**

* ❌ Write quorum (4/6) NOT met
* ✅ Read quorum (3/6) still met
* 🔶 Database becomes **read-only**
* 🔶 Replicas remain usable for reads
* 🔶 Writes are blocked

> This is the **worst failure Aurora can survive without total outage**.

---

## Scenario 4: Loss of Fourth Node

**Failure:**

* More than AZ + 1 failure
* Fewer than 3 nodes remain

**Result:**

* ❌ Write quorum lost
* ❌ Read quorum lost
* ❌ Database becomes unavailable

At this point:

* Application must rely on **DR strategy**
* Example:

  * Cross-region backups
  * Aurora Global Database

---

## Long-Term AZ Failure & Dictator Mode

AWS has introduced a **dictator mode** for rare long-term AZ outages:

* Temporarily relaxes quorum rules
* Allows continued availability
* Used only during **extended AZ loss**
* Managed internally by AWS

> This reduces downtime risk but **does not replace DR planning**.

---

## Disaster Recovery Responsibility

Failures beyond **AZ + 1** are rare, but possible.

Users must plan for:

* Cross-region backups
* Aurora Global Database
* Defined RTO / RPO objectives

---

# Exam-Style Q&A (Aurora Replication & Failures)

### Q1: Does Aurora need WAL?

**Yes.**
Aurora still uses WAL/redo logs, but:

* WAL is sent directly to storage nodes
* WAL is **not replayed on replicas**

---

### Q2: Why doesn’t Aurora use PostgreSQL streaming replication?

Because:

* Storage is shared
* Disk-level replication is unnecessary
* Only buffer cache needs synchronization

---

### Q3: What does Aurora replication actually replicate?

**Buffer cache page changes**, not WAL or storage blocks.

---

### Q4: Can Aurora replicas return stale data?

**Yes.**
Aurora replicas are **eventually consistent**, but lag is typically <100 ms.

---

### Q5: What happens if replication lag becomes too high?

Aurora may:

* Restart the replica automatically
* Allow it to catch up cleanly

---

### Q6: How many storage nodes can fail without downtime?

* Up to **2 nodes** → full read/write
* Up to **3 nodes** → read-only
* 4+ nodes → unavailable

---

### Q7: What is the AZ + 1 fault model?

Aurora remains available with:

* 1 full AZ failure
* Plus 1 additional node failure in another AZ

---

### Q8: Is Aurora replication synchronous?

**No.**
Aurora replication is **asynchronous**, but extremely fast.

---

### Q9: Does Aurora replicate data pages to disk on replicas?

**No.**
Replication is **memory-to-memory only**.

---

### Q10: Who is responsible for disaster recovery beyond AZ + 1?

**The customer.**
Aurora provides durability, not cross-region DR by default.

---

## Final Memory Hooks (Critical)

> **“Aurora storage is replicated by quorum; Aurora replication fixes memory.”**

> **“Failures are expected — quorum absorbs them.”**

> **“Aurora doesn’t wait for everyone, only enough.”**

---

## One-Line Final Summary

> **Aurora achieves high availability by combining quorum-based distributed storage with memory-level replication, allowing it to tolerate multiple node and Availability Zone failures while maintaining low replication lag and fast recovery.**
