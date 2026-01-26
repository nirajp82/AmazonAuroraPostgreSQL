# Amazon Aurora PostgreSQL – Global Database

**Amazon Aurora Global Database** is designed to run a single logical database across multiple AWS regions. It primarily addresses two needs:

* **Region-level disaster recovery** (protection against entire AWS region outages)
* **Low-latency global reads** by keeping data close to users

Unlike a standard Aurora cluster (which is confined to a single region), Aurora Global Database replicates data at the **storage layer** across regions, enabling fast recovery and global scale without custom backup-and-restore automation.

---

## Why Global Database Is Needed

### The Problem with Single-Region Aurora

A standard Aurora cluster:

* Protects against **Availability Zone (AZ)** failures
* Does **not** protect against **entire region failures**

If a region becomes unavailable:

1. **Manual recovery** is required using snapshots or cross-region restores
2. **Infrastructure recreation** is complex and error-prone
3. **High RTO** (often hours)
4. **Non-trivial RPO**, depending on backup frequency

---

## What Global Database Solves

Aurora Global Database mitigates these issues by:

* Continuously replicating data **from one primary Region to up to 10 read-only secondary Regions** using **asynchronous replication** (typically sub-second lag)
* Using **AWS-managed, private high-speed global backbone networking**, avoiding the public internet
* **Offloading replication to the distributed storage layer**, reducing CPU and write overhead on the primary DB compute and improving write performance

**Memory Hook:** *One Region writes, many Regions read — storage handles replication, not the database engine.*


Typical outcomes (workload dependent):

* **RPO:** Seconds
* **RTO:** Minutes

---

## How Global Database Works

### Core Concepts

| Term                  | Meaning                                                    |
| --------------------- | ---------------------------------------------------------- |
| **Primary Cluster**   | The only cluster that accepts **writes** (and reads)       |
| **Secondary Cluster** | Read-only clusters in other regions                        |
| **Global Database**   | A logical grouping of one primary and multiple secondaries |

Only **one** primary exists at any time.

---

### Data Replication Model

* Replication is **asynchronous**
* Occurs at the **storage layer**, not via logical replication or read replicas
* Uses the **AWS-managed backbone**, not the public internet

This design minimizes replication lag while avoiding performance impact on the primary cluster.

---

### Write Forwarding (Feature Overview)

Write Forwarding allows applications connected to a **secondary cluster** to issue write operations:

* The write is transparently forwarded to the primary cluster
* The transaction commits on the primary
* Results are returned to the caller

This feature improves developer ergonomics but **does not change the single-writer model**.

---

## Disaster Recovery and Failover

### Key Clarification (Very Important)

Aurora Global Database provides **automatic database-level failover**, but **application traffic failover is not automatic by default**.

Specifically:

* **AWS automatically handles database promotion and replication role changes**
* **Applications must handle reconnection and endpoint usage**

The phrase “no user intervention” applies strictly to **replication and database role management**, not DNS or application routing.

---

### Secondary Clusters as Failover Targets

Secondary clusters are **pre-configured promotion targets**.

If the primary region fails:

* One secondary cluster is promoted to primary
* It transitions from **read-only → read/write**
* Replication topology is updated by AWS

These database-level changes are fully managed by AWS.

---

## Types of Cross-Region Failover

### 1. Managed Planned Failover (Switchover)

Used for:

* Disaster recovery drills
* Planned region migration

Characteristics:

* Initiated manually (Console / CLI / API)
* **RPO = 0** (no data loss)
* AWS reverses replication direction
* Writer endpoint changes

---

### 2. Unplanned Failover (Regional Outage)

Used when:

* The primary region becomes unavailable

Characteristics:

* Triggered by user action or automation
* **RPO > 0** (small data loss possible due to async replication)
* A surviving secondary is promoted
* New writer endpoint is created

> Aurora handles promotion and replication. Applications must reconnect.

---

### Step-by-Step: Unplanned Regional Failure

1. **Primary Region Failure Occurs**
   AWS detects the outage through internal health monitoring.

2. **Failover Is Initiated**
   The failover is triggered (manually or via automation).

3. **Secondary Promotion**
   AWS promotes a secondary cluster to primary and enables writes.

4. **Replication Roles Updated**
   The promoted cluster becomes the new replication source.

5. **Application Reconnection**
   Existing connections fail. Applications must retry and reconnect using the new writer endpoint.

---

## Who Handles What

| Responsibility                 | Owner                           |
| ------------------------------ | ------------------------------- |
| Detecting regional failure     | AWS (signals) / User (decision) |
| Promoting secondary to primary | **AWS**                         |
| Replication role changes       | **AWS**                         |
| Writer endpoint change         | **AWS**                         |
| DNS / endpoint abstraction     | **User / Architecture**         |
| Connection retries             | **Application / Driver**        |

---

## Endpoint Management (Why Automation Is Often Needed)

After a failover:

* The **writer endpoint changes**
* Applications using hardcoded endpoints will fail

Best practices:

* Use **DNS abstraction** (for example, Route 53 CNAME)
* Design applications to **retry connections**
* Avoid embedding regional endpoints directly in application configuration

Aurora manages the database. **You manage how applications find it.**

---

## FAQ

### Q1: Is Aurora Global Database multi-master?

No. There is always exactly **one writer**. Write Forwarding does not change this model.

---

### Q2: Is cross-region failover fully automatic?

Database promotion and replication changes are automatic **once failover is initiated**. Application routing and DNS updates are not automatic by default.

---

### Q3: Can zero data loss be guaranteed?

Only during **managed planned failover**. Unplanned failover may lose a small amount of data due to asynchronous replication.

---

### Q4: Does replication use the public internet?

No. Replication uses the **AWS-managed high-speed backbone network**.

---

## Memory Hooks (Quick Recall)

* **AZ failover** = automatic
* **Region failover** = database automatic, traffic is not
* **Primary** writes, **secondary** reads
* **Failover target = secondary cluster**
* **AWS manages DB roles; apps manage reconnection**

---

> **Tip:** After switching reads to a secondary region, use the following query to validate cache efficiency:
>
> ```sql
> SELECT 
>   sum(heap_blks_read) AS heap_read,
>   sum(heap_blks_hit)  AS heap_hit,
>   (sum(heap_blks_hit) - sum(heap_blks_read))::float / NULLIF(sum(heap_blks_hit),0) AS hit_ratio
> FROM pg_statio_user_tables;
> ```
