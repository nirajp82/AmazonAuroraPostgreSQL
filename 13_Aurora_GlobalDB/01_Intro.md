# Amazon Aurora PostgreSQL – Architecture, High Availability, and Global Database

## Overview

Amazon Aurora is a **cloud-native relational database** compatible with PostgreSQL and MySQL, designed to deliver **high performance, high availability, and durability** in cloud environments.

Aurora is not a traditional database running on cloud infrastructure. It is a **re-architected system** that separates compute from storage and assumes that **failures are normal and continuous**.

---

## Part 1: Aurora Architecture Fundamentals

### Traditional Database Model (Pre-Cloud)

Traditional databases tightly couple:

* Compute (query processor, transaction manager, buffer cache)
* Storage (data files and redo logs)

Performance improvements historically relied on:

* Faster disks
* Wider I/O channels

This model assumes long-lived servers and stable infrastructure.

---

## Why This Model Breaks in the Cloud

In cloud environments:

* Instances are **ephemeral** (they fail, scale, and are replaced)
* Storage must outlive compute
* Network bandwidth becomes the primary bottleneck

Because of this, **compute and storage must be decoupled**.

---

## Aurora’s Decoupled Compute & Storage Design

Aurora separates the database into two independent tiers:

### 1. Database (Compute) Tier

* SQL parsing and execution
* Transaction management
* Buffer cache
* Sends **redo log records only** (deltas)

Important:

* Aurora **never writes full data pages** for checkpoints or cache eviction

### 2. Storage Tier

* Distributed, multi-tenant AWS-managed service
* Stores data **six times across three Availability Zones (AZs)**
* Applies redo logs internally
* Materializes data blocks **on demand** or in the background

### Memory Hook

> **In Aurora, the log is the database.**

The compute node sends only changes; storage reconstructs data.

---

## Part 2: High Availability and Durability (Single Region)

### Availability Zones (AZs)

An **Availability Zone** is:

* One or more physically isolated data centers
* Independent power, networking, and cooling

AZ failures can occur due to:

* Power outages
* Network isolation
* Natural disasters

Aurora assumes AZ failures **will happen**.

---

## Storage Replication Model

Aurora stores **six copies of every data block**:

* Across **3 AZs**
* **2 copies per AZ**

This enables survival of:

* Disk failures
* Node failures
* Entire AZ failures

---

## Quorum-Based Writes and Reads

Aurora does not require all replicas to be available.

| Operation          | Requirement   | Purpose                    |
| ------------------ | ------------- | -------------------------- |
| Write              | 4 of 6 copies | Guarantees durability      |
| Read (repair only) | 3 of 6 copies | Used during storage repair |

### Important Clarification

* **Normal reads do NOT use read quorums**
* Reads are served from a **single fastest valid replica**

This avoids read amplification and latency penalties.

---

## Why 3 Copies (One per AZ) Is Not Enough

Failures do not occur in isolation.

### AZ + 1 Failure Scenario

With only 3 copies:

1. Lose one AZ → 1 copy gone
2. Lose one more node → quorum lost
3. Writes stop; data at risk

### Aurora’s Design

With 6 copies:

* Lose an AZ (2 copies) → still 4
* Lose another node → still 3
* Data safe; system self-heals

This is called **AZ + 1 failure tolerance**.

---

## Self-Healing Storage

When a storage node fails:

* Aurora rebuilds it automatically
* Copies data from healthy replicas
* Happens continuously in the background

> Temporary write impact may occur, but **data is never lost**.

---

## Read Performance Optimization

Aurora tracks:

* Replica progress
* Replica latency

Each read is routed to the **lowest-latency up-to-date replica**.

---

## Part 3: Why Single-Region Aurora Is Not Enough

Single-region Aurora:

* Protects against **AZ failures**
* Does **not** protect against **entire region failures**

If a region becomes unavailable:

* Snapshot restore is required
* Infrastructure must be recreated
* **High RTO** (hours)
* **Non-trivial RPO**

---

## Part 4: Amazon Aurora Global Database

### What Is Aurora Global Database

Aurora Global Database runs a **single logical database across multiple AWS regions**.

It addresses:

* **Region-level disaster recovery**
* **Low-latency global reads**

Replication happens at the **storage layer**, not via logical replication.

---

## Core Concepts

| Term              | Meaning                             |
| ----------------- | ----------------------------------- |
| Primary Cluster   | Only cluster that accepts writes    |
| Secondary Cluster | Read-only clusters in other regions |
| Global Database   | One primary + multiple secondaries  |

Only **one primary** exists at any time.

---

## How Global Database Replication Works

* **Asynchronous replication**
* Physical (redo-log based)
* Uses **AWS private backbone**, not public internet
* Offloaded to storage layer (minimal compute overhead)

### Memory Hook

> **One Region writes, many Regions read — storage handles replication.**

Typical outcomes:

* **RPO:** Seconds
* **RTO:** Minutes

---

## Write Forwarding (Feature Overview)

Allows applications connected to a **secondary cluster** to issue writes:

* Writes are forwarded to the primary
* Commit occurs on the primary
* Results returned transparently

> This does NOT create multi-writer behavior.

---

## Disaster Recovery and Failover

### Critical Clarification

Aurora Global Database:

* **Automatically manages database promotion and replication roles**
* **Does NOT automatically reroute application traffic**

Database is managed by AWS. **Traffic routing is your responsibility.**

---

## Types of Cross-Region Failover

### 1. Planned Failover (Switchover)

Used for drills or migrations:

* Manual initiation
* **RPO = 0**
* Replication direction reverses

### 2. Unplanned Failover

Used during regional outages:

* Initiated by user or automation
* **RPO > 0** (async replication)
* Secondary promoted to primary

---

## Step-by-Step: Unplanned Regional Failover

1. Primary region fails
2. Failover initiated
3. Secondary promoted to primary
4. Replication roles updated by AWS
5. Applications reconnect to new writer endpoint

---

## Responsibility Breakdown

| Area                     | Owner       |
| ------------------------ | ----------- |
| Database promotion       | AWS         |
| Replication role changes | AWS         |
| Writer endpoint update   | AWS         |
| DNS / traffic routing    | User        |
| Connection retries       | Application |

---

## Endpoint Management Best Practices

* Use Route 53 or DNS abstraction
* Avoid hardcoded regional endpoints
* Implement connection retry logic

> Aurora manages the database. **You manage how apps find it.**

---

## Memory Hooks (Quick Recall)

* Logs are the database
* 6 copies across 3 AZs
* 4/6 writes, 3/6 repair reads
* AZ failover automatic
* Region failover: DB automatic, traffic manual

---

## FAQ

### Is Aurora Global Database multi-master?

No. Exactly one writer exists at all times.

### Is cross-region failover automatic?

Database role changes are automatic **once initiated**. Traffic routing is not.

### Can zero data loss be guaranteed?

Yes for planned failovers. No for unplanned failovers.

### Does replication use the public internet?

No. AWS private backbone is used.

---

## Validation Query (Post-Failover)

```sql
SELECT 
  sum(heap_blks_read) AS heap_read,
  sum(heap_blks_hit)  AS heap_hit,
  (sum(heap_blks_hit) - sum(heap_blks_read))::float / NULLIF(sum(heap_blks_hit),0) AS hit_ratio
FROM pg_statio_user_tables;
```
