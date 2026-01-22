# Amazon Aurora PostgreSQL – Global Database Internals, Limits, and Design Choices

## Overview

This lesson dives deeper into **how Amazon Aurora Global Database works internally**, how it achieves high availability, and what **limitations and configuration requirements** you must understand before using it in production.

The focus here is not just *what* Global Database does, but *how and why* it behaves the way it does.

---

## Global Database Topology

A Global Database:

* Spans **multiple AWS regions**
* Has **exactly one primary cluster**
* Can have **up to five secondary clusters**
* Can span **a maximum of six regions**

<img width="1092" height="516" alt="Global Database Topology" src="https://github.com/user-attachments/assets/abf04c33-ffa9-4425-b723-8bacf481b593" />

### Cluster Roles

| Cluster Type      | Role                             |
| ----------------- | -------------------------------- |
| Primary cluster   | Accepts **all writes** and reads |
| Secondary cluster | **Read-only** (until promoted)   |

There is **only one writer instance**, and it always resides in the primary cluster.

---

## Replication Model (Under the Covers)

### Physical (Storage-Level) Replication

Aurora Global Database uses **physical replication**:

* Storage volumes from the primary region are replicated to secondary regions
* Replication occurs over **AWS-managed high-speed infrastructure**
* Replication does **not** use logical decoding or SQL-based replication

### Why This Matters

* Replication has **little to no impact** on primary cluster performance
* Typical replication lag is **under one second**
* Enables very high throughput compared to logical replication

---

## Physical vs Logical Replication (Performance Insight)

Independent benchmarking (referenced from AWS blogs) compared:

* **Logical replication** across regions
* **Aurora Global Database physical replication**

### Observations

| Metric             | Logical Replication | Global DB Physical Replication |
| ------------------ | ------------------- | ------------------------------ |
| Peak QPS           | ~35,000             | ~200,000                       |
| Replication lag    | Grows exponentially | < 1 second                     |
| Performance impact | High                | Minimal                        |

**Conclusion:** Physical replication allows **much higher throughput and stability** at scale.

---

## Writer, Readers, and Replicators

### Writer Behavior

* All **writes** are handled by the writer instance in the primary cluster
* Secondary clusters never accept writes unless promoted

### Replication Flow

* Primary cluster uses an **outbound replicator**
* Secondary clusters use an **inbound replicator**
* Data changes are streamed at the storage layer

> This replication mechanism is **proprietary to Amazon Aurora**.

<img width="1072" height="508" alt="Replication Flow" src="https://github.com/user-attachments/assets/2a223704-e4d8-49ae-bdfc-83c4932c5bcf" />

---

## Consistency Model

Because replication is asynchronous:

* Replication lag is **finite (> 0)**
* Secondary clusters are **eventually consistent**

This is a deliberate trade-off to achieve:

* High availability
* High throughput
* Low write latency

---

## Secondary Clusters as Failover Targets

Secondary clusters are **pre-configured promotion targets**.

### Example Scenario

* Primary cluster in **Region 1**
* Secondary clusters in **Region 2** and **Region 3**
* All **writes** are handled by the primary cluster

If the primary cluster fails:

1. Applications connected to the primary writer experience an outage
2. Replication from primary to secondary stops (expected)
3. A user or automation **initiates failover**
4. One secondary cluster is promoted to primary
5. The promoted cluster becomes **read/write**
6. Applications reconnect to the new writer endpoint

> Database promotion is handled by AWS. Applications must reconnect.

---

## RPO and RTO Expectations

With proper automation:

* **RPO:** ~1 second
* **RTO:** ~1 minute

### Important Clarification

Global Database failover is fast, but **applications must be prepared** to:

* Detect connection failures
* Retry connections
* Switch to the new writer endpoint

Automation at the application or DNS layer is critical.

---

## Headless Secondary Clusters

A secondary cluster:

* May have **no database instances** (headless)
* Still maintains a replicated storage volume
* Has replication lag typically under one second

### Use Case

This design is useful when:

* **RPO is critical**
* **Longer RTO is acceptable**
* Cost optimization is important

If the primary fails:

* Instances can be added to the headless cluster
* The cluster is promoted to primary

---

## Independent Cluster Management

Clusters in a Global Database are managed **independently**:

* Replica count can vary per cluster
* Instance classes can differ by region
* Scaling decisions are regional

### Parameter Groups

* Managed independently per cluster
* Some global database-specific restrictions apply

---

## Global Database Limitations

Be aware of the following constraints:

1. **Clusters cannot be stopped** once part of a Global Database
2. Cluster identifiers must be **globally unique**
3. **Automatic minor version upgrades are not supported** (setting has no effect)
4. If the **primary writer instance restarts or fails**, secondary clusters may become unavailable until replication resynchronizes

---

## Configuration Requirements

### Instance Classes

* Must be **memory-optimized**
* Minimum recommended: **db.r5 or higher**

### Structural Requirements

* Minimum: **1 primary + 1 secondary** cluster
* Maximum: **1 primary + 5 secondary** clusters
* Only **one writer instance** (in the primary cluster)

### Replica Limits

* Each secondary cluster: **up to 16 replicas**
* Primary cluster: **15 − number of secondary clusters** replicas

```pgsql
Aurora Global Database
├── Primary Cluster (1 region)
│   ├── 1 Writer instance
│   ├── N Reader instances
│
├── Secondary Cluster (region 2)
│   ├── 0–16 Reader instances
│
├── Secondary Cluster (region 3)
│   ├── 0–16 Reader instances
│
└── (up to 5 secondary clusters)
```

### Aurora Global Database – Replica & Cluster Limits (With Examples)

| Item                                     | Limit                                 | What It Means                                                           | Example                                                                                         |
| ---------------------------------------- | ------------------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Writer instances**                     | Exactly **1**                         | Only one instance in the entire Global DB can accept writes at any time | `us-east-1` has **1 writer**. All tenant DB writes go here                                      |
| **Secondary clusters**                   | Up to **5**                           | You can attach up to 5 additional regional clusters for reads / DR      | Primary in `us-east-1`, secondary in `us-west-2`, `eu-west-1`, `ap-southeast-1` (3 secondaries) |
| **Total replicas per cluster (primary)** | **15 − number of secondary clusters** | Primary cluster loses replica capacity as you add secondary clusters    | 3 secondary clusters → `15 − 3 = 12` readers allowed in primary                                 |
| **Replicas per secondary cluster**       | Up to **16**                          | Each secondary cluster can scale read replicas independently            | `eu-west-1` secondary has **8 readers** to serve EU traffic, and can scale up to 16 readers     |
| **Total clusters in a Global DB**        | **1 primary + up to 5 secondary**     | Maximum of 6 regional clusters per Global DB                            | Regions: `us-east-1` (primary) + 5 others                                                       |

---

## Benefits of Aurora Global Database

* Low-latency reads across regions
* Independent scaling per region
* Fast cross-region replication
* **~1 second RPO** achievable
* Protection against **region-level failures**

---

## Cost and When NOT to Use Global Database

Global Database:

* Adds significant value for critical workloads
* Is **more expensive** than a single-region Aurora cluster

It may be **overkill** if:

* RPO requirements are loose
* Cross-region latency is not a concern
* The workload is non-critical

---

## Decision Guide

Use **Aurora Global Database** when:

* Low RPO is required
* Region-level DR is mandatory
* Global read latency matters

Use a **headless secondary cluster** when:

* RPO is critical
* RTO can be longer
* Cost optimization matters

Use a **regular Aurora cluster** when:

* Cross-region reads are unnecessary
* Region-level DR is not required

---

## Memory Hooks (Quick Recall)

* **1 writer only**
* **Physical replication**
* **< 1 second lag**
* **Secondary = failover target**
* **AWS handles DB roles, apps handle reconnection**
* **Powerful but costly**

---

## Failover Summary Example

Consider a Global Database with:

* Primary cluster in **Region 1**

* Secondary clusters in **Region 2** and **Region 3**

* All **writes** go to the primary cluster

* If primary fails, replication stops

* Failover can be triggered to promote a secondary cluster

* Applications reconnect to the new primary writer endpoint

This ensures continuity with **minimal data loss (RPO ~1 sec)** and **rapid recovery (RTO ~1 min)**.

---

## FAQ – Aurora Global Database

**Q1: Can a secondary cluster accept writes?**
No. Secondary clusters are **read-only** unless promoted to primary. All writes go to the single **writer instance** in the primary cluster.

**Q2: How many secondary clusters can I have?**
Up to **5 secondary clusters**, so a Global Database can span a maximum of **6 regions** including the primary.

**Q3: How many read replicas can each cluster have?**

* **Primary cluster:** `15 − number of secondary clusters` replicas
* **Secondary cluster:** Up to **16 replicas** each

**Q4: What happens if the primary cluster fails?**

* Applications experience an outage until failover occurs
* One of the secondary clusters is promoted to primary by AWS
* Applications must reconnect to the new writer endpoint
* Typical **RPO ~1 second**, **RTO ~1 minute** with proper automation

**Q5: Can clusters be stopped?**
No. Once part of a Global Database, clusters **cannot be stopped**.

**Q6: Are replication and failover automatic?**

* **Replication:** Continuous and automatic at the storage level
* **Failover:** **Cross-region failover is triggered manually** or via automation; AZ-level failover within a region is automatic

**Q7: Can I mix instance classes and replicas across clusters?**
Yes. Each cluster is managed independently:

* Replica count can vary
* Instance classes can differ per region
* Parameter groups are cluster-specific (with some limitations)

**Q8: Can I use automatic minor version upgrades?**
No. The option exists in the console but **has no effect** for Global Database clusters.

**Q9: What is a headless secondary cluster?**
A secondary cluster with **no database instances** but a fully replicated storage volume. Instances can be added later if the cluster is promoted to primary. Useful for:

* RPO-critical workloads
* Cost optimization
* Workloads where longer RTO is acceptable



