# Amazon Aurora PostgreSQL – Global Database

## Overview

**Amazon Aurora Global Database** is designed to run a single logical database across multiple AWS regions. It is the premier solution for:

* **Region-level disaster recovery** (protection against entire AWS region outages).
* **Low-latency global reads** by keeping data geographically close to users.

Unlike a standard Aurora cluster (limited to one region), a Global Database manages replication at the **storage layer**, allowing for high-performance global scaling and rapid recovery without manual backup-and-restore automation.

---

## Why Global Database Is Needed

### The Problem with Single-Region Aurora

A standard Aurora cluster protects against Availability Zone (AZ) failures but **cannot** survive a full region failure. In a regional outage:

1. **Manual Recovery:** You must restore from snapshots or use cross-region clones.
2. **Complexity:** Recreating infrastructure in a new region is slow and error-prone.
3. **RTO/RPO:** Recovery Time (RTO) is high (hours), and Recovery Point (RPO) depends on the age of the last backup.

### The Global Database Solution

Aurora Global Database solves this by:

* Continuously replicating data to up to **5 secondary regions**.
* Using dedicated infrastructure that doesn't impact primary cluster performance.
* Achieving a typical **RPO of < 1 second** and **RTO of < 1 minute**.

---

## How Global Database Works

### Core Concepts

| Term | Meaning |
| --- | --- |
| **Primary Cluster** | The main cluster (one per Global DB) that accepts read and write traffic. |
| **Secondary Cluster** | Read-only clusters in other regions (up to 5 regions). |
| **Global Database** | The logical container for the primary and all secondary clusters. |

### Write Forwarding (New Feature)

Previously, applications in secondary regions had to send writes to the primary region's endpoint manually.

* **Local Write Forwarding:** You can now issue write statements (DML) directly to a secondary cluster. Aurora automatically "forwards" the write to the primary and returns the result once committed.

---

## Disaster Recovery (The Failover Process)

### ⚠️ Important Correction on Automation

While AZ-failover within a region is automatic, **Cross-Region Failover is NOT automatic by default.** AWS requires a manual trigger (or your own automation like Route 53 ARC) to initiate a regional failover. This prevents accidental data loss during transient network "blips" between regions.

### Types of Failover

1. **Managed Planned Failover (Switchover):** Used for DR drills or changing your primary region.
* **RPO = 0** (Zero data loss).
* Replication direction is automatically reversed.


2. **Unplanned Failover (Failover):** Used during a real regional outage.
* **RPO > 0** (Small data loss possible depending on lag).
* The secondary is detached and promoted to a standalone primary.



### Step-by-Step Flow (Unplanned)

1. **Region Failure Detected:** Your monitoring (CloudWatch/Health Dashboard) alerts you to the outage.
2. **Initiate Failover:** You trigger the "Failover" command via the AWS Console or CLI.
3. **Secondary Promotion:** AWS promotes the secondary cluster to a primary. It transitions from Read-Only to Read/Write.
4. **Application Reconnection:** The **Global Cluster Endpoint** is updated. Applications must be designed to retry and reconnect.

---

## Who Handles What?

| Responsibility | Who Handles It |
| --- | --- |
| Detecting regional outage | AWS (Monitoring) / User (Decision) |
| Promoting secondary to primary | **AWS** (Once triggered) |
| Updating Global Endpoints | **AWS** |
| Retrying failed queries | **Application / DB Driver** |
| Reconnecting to the new writer | **Application** |

---

## FAQ

**Q1: Is Aurora Global Database multi-master?**
No. Only one cluster is the Primary "Writer." However, "Write Forwarding" allows secondary regions to handle write requests by forwarding them to the Primary.

**Q2: How do I guarantee zero data loss?**
You can use the **Managed RPO** feature. You set a maximum lag (e.g., 20 seconds). If the lag exceeds this, Aurora throttles writes on the primary to ensure the secondary stays within the RPO window.

**Q3: Does replication happen over the public internet?**
No. It uses the **AWS-managed high-speed backbone**, ensuring low latency and high security.

---

## Memory Hooks (Quick Recall)

* **AZ-Failover** = Automatic. **Region-Failover** = Triggered by you.
* **Primary** = Writer. **Secondary** = Reader (until promoted).
* **Storage-based** = No performance hit on the main DB.
* **RPO** = Seconds. **RTO** = Minutes.

---

> **Tip:** You can use the following SQL script to check your local cache performance once your application starts reading from a secondary region:
> ```sql
> SELECT 
>   sum(heap_blks_read) as heap_read,
>   sum(heap_blks_hit)  as heap_hit,
>   (sum(heap_blks_hit) - sum(heap_blks_read))::float / sum(heap_blks_hit) as hit_ratio
> FROM pg_statio_user_tables;
> 
> ```
