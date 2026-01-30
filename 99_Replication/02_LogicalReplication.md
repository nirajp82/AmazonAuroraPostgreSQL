# PostgreSQL WAL & Logical Replication Guide

## Overview

PostgreSQL uses **Write-Ahead Logging (WAL)** to ensure data durability and support replication. This guide explains how WAL works in **multi-database clusters**, how **logical replication slots** consume WAL, and practical considerations for retention, performance, and troubleshooting.

## Glossary

| Term                          | Meaning|
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **WAL (Write-Ahead Log)**     | A sequential log of all changes made to the database. Every change (INSERT, UPDATE, DELETE, DDL) is first written to WAL before the actual data pages on disk are updated. This ensures **durability** and enables **crash recovery**. WAL files are stored in **16 MB segments** by default and are shared across all databases in the cluster. WAL is also the source for replication, both **physical** and **logical**.                                                                                                                                                                                                                                                                                          |
| **LSN (Log Sequence Number)** | A pointer to a specific byte in the WAL stream. It identifies the order of changes in the WAL. Example: `0/1A2B3C4D`. LSNs are used for replication, recovery, and determining **slot retention**. Operations like `pg_current_wal_lsn()` or `pg_wal_lsn_diff()` allow you to measure how far behind a subscriber or slot is.                                                                                                                                                                                                                                                                                                                                                                                        |
| **Replication slot**          | A **named handle** on the publisher that tracks WAL consumption by a subscriber. The slot lives **on the publisher**, not the subscriber. It prevents WAL needed by a subscriber from being recycled. There are two types: **physical** (whole-cluster streaming) and **logical** (specific tables or publications). Slots advance their position as subscribers consume WAL. Inactive or slow slots can cause **WAL bloat**.                                                                                                                                                                                                                                                                                        |
| **Publisher**                 | The PostgreSQL instance (or cluster) that produces WAL and holds the replication slots. Subscribers connect to the publisher to receive changes. One publisher can serve **multiple subscribers** or multiple publications. All slot tracking, restart_lsn, and WAL retention happens on the publisher.                                                                                                                                                                                                                                                                                                                                                                                                              |
| **Subscriber**                | The client or replica that connects to the publisher to receive WAL changes. This can be another PostgreSQL database, a data pipeline, or a custom application using logical replication. The subscriber confirms the WAL it has processed using **confirmed_flush_lsn** (logical replication) to signal progress back to the publisher.                                                                                                                                                                                                                                                                                                                                                                             |
| **restart_lsn**               | The **oldest WAL byte still required by a replication slot**. WAL segments older than the restart_lsn can be recycled because no slot or subscriber depends on them. **Key points:** <br>• It is cluster-wide WAL, but each slot advances its own restart_lsn. <br>• If you have multiple slots, the **oldest restart_lsn among all slots** determines how far back WAL must be kept. <br>• A slow or inactive slot keeps restart_lsn low, causing WAL to accumulate and increasing `WalPressureBytes`. <br>**Example:** <br> Slot “slot1” has restart_lsn `0/3000000`. Current WAL LSN is `0/6000000`. WAL between `0/3000000` and `0/6000000` is **retained** by the slot; WAL before `0/3000000` can be recycled. |
| **confirmed_flush_lsn**       | (Logical slots only) The LSN up to which the subscriber has **confirmed it has received and processed WAL**. It is **always ≤ restart_lsn**. It measures **subscriber lag**, i.e., how far behind the subscriber is in applying changes. <br>**Example:** If current WAL is `0/6000000` and the confirmed_flush_lsn is `0/5500000`, the subscriber is 500 KB behind the current WAL. This helps identify slow consumers even if restart_lsn is advanced for WAL retention.                                                                                                                                                                                                                                           |
| **Publication**               | A logical set of tables on the publisher that are exposed for replication. Subscribers subscribe to publications to receive changes. Publications filter the WAL so that **only relevant changes** are delivered to each subscriber.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Subscription**              | The configuration on the subscriber that connects to a publication on the publisher. Subscriptions establish the replication stream, apply changes, and report progress via confirmed_flush_lsn.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **WalPressureBytes**          | WAL retained on the publisher for a publication or slot. This measures **how much WAL is currently “reserved” and cannot be recycled** due to slot requirements. A high value doesn’t always mean replication lag — it can occur if the slot hasn’t advanced its restart_lsn or if there are other slow/inactive slots.                                                                                                                                                                                                                                                                                                                                                                                              |
| **ReplicationLagBytes**       | How far behind the subscriber is for a publication in terms of WAL bytes. This is measured using **confirmed_flush_lsn** versus the current WAL LSN. Low ReplicationLagBytes means the subscriber is keeping up, even if WAL pressure on the publisher is high.                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Logical Decoding**          | The process by which a replication slot reads cluster-wide WAL and produces **change events only for the publication’s tables**. The logical decoder filters WAL per table/publication and generates messages for subscribers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **max_slot_wal_keep_size**    | A GUC parameter (PostgreSQL 13+) that limits WAL retention for slots. If WAL retained exceeds this value, slots may become **lost**, meaning the subscriber must be reinitialized because required WAL was removed.   |

---

## 1. WAL Basics

* **Cluster-wide:** WAL is generated at the **cluster level**, not per database. One cluster has one WAL stream regardless of how many databases it contains.
* **Purpose:** Ensures durability (crash recovery) and replication. Every change (INSERT, UPDATE, DELETE, DDL) is written to WAL before the transaction commits.
* **Segment size:** Each WAL segment file is **16 MB by default**. A new segment is created when the current one is full.
* **Retention:** WAL segment files cannot be removed (recycled) while **any replication slot or streaming replica** still needs them. The **oldest** restart LSN across all slots and replicas determines how far back WAL must be kept—so one slow or inactive slot can force retention of a large amount of WAL.

**Example:**

| Cluster | Databases | WAL files generated                            | Notes                                               |
| ------- | --------- | ---------------------------------------------- | --------------------------------------------------- |
| 1       | 3         | Depends on total write activity across all DBs | WAL is shared; number of DBs does not multiply WAL. |

---

## 2. Logical Replication & Slots

PostgreSQL has **physical** replication slots (used for streaming replication of the whole instance) and **logical** replication slots (used to replicate selected tables or databases). This section focuses on **logical** slots.

**Logical replication slots** allow replication of **specific tables** or **databases** without replicating the entire cluster. A logical slot is **created in one database** on the publisher; it decodes only WAL that affects that database (and only the tables in the publication).

### How it works:

1. **Slot creation:** A slot is created on the **publisher** at the current LSN (e.g. via `pg_create_logical_replication_slot(slot_name, 'pgoutput')`).
2. **WAL consumption:** The slot reads cluster-wide WAL **sequentially from its restart_lsn**. (WAL is cluster-wide, but the logical decoder only outputs changes for the slot’s database and publication.)
3. **Filtering:** The slot (via the publication) delivers **only the changes** for the subscribed tables.
4. **Tracking:** The slot advances **restart_lsn** and (for logical slots) **confirmed_flush_lsn** as the subscriber consumes data. WAL cannot be recycled until all slots have advanced past it.

**Important:**

* WAL **generation** does **not** increase when you add more slots. More writes cause more WAL; slots only **consume** and **retain** it.
* Multiple slots **retain** WAL for longer: the slowest slot’s restart_lsn determines the minimum WAL that must be kept. If you drop a **subscription** but leave the **slot** on the publisher, the slot will keep retaining WAL until the slot is dropped—a common cause of WAL bloat.

---

### WAL Flow Example

The slot lives on the **publisher**. It reads the cluster’s WAL sequentially and, for logical replication, outputs only changes for the slot’s database and publication.

```
Cluster WAL (cluster-wide, on publisher)
+-------------------------+-------------------------+-------------------------+
| WAL file 1              | WAL file 2              | WAL file 3              |
| DB1: tableX changes     | DB2: tableA changes     | DB3: tableY changes     |
| DB2: tableB changes     | DB1: tableZ changes     | DB2: tableA changes     |
+-------------------------+-------------------------+-------------------------+

Logical slot "slot_db2" (created in DB2, publication = certain tables in DB2)
    Reads WAL file 1 -> skips DB1.tableX -> outputs DB2.tableB
    Reads WAL file 2 -> outputs DB2.tableA -> skips DB1.tableZ
    Reads WAL file 3 -> outputs DB2.tableA -> skips DB3.tableY
```

**Visual diagram for multiple slots across databases:**

```
Cluster WAL
┌─────────────┐
│ WAL files   │
│ DB1/DB2/DB3 │
└─────────────┘
       │
       ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ Slot DB1    │   │ Slot DB2    │   │ Slot DB3    │
│ outputs DB1 │   │ outputs DB2 │   │ outputs DB3 │
└─────────────┘   └─────────────┘   └─────────────┘
```

---

## 3. Multi-Database Clusters

* **Single cluster, multiple DBs:** All databases share the **same** WAL stream. One cluster = one WAL.
* **Number of databases does NOT multiply WAL.** WAL volume depends on **total write activity** across the cluster.
* **One logical slot per database:** A logical slot is created **in** one database (e.g. `pg_create_logical_replication_slot(...)` in that database). It decodes only WAL that affects **that** database and only outputs changes for tables in its **publication**. Other databases’ changes are in the same WAL stream but are not delivered by this slot.

---

## 4. WAL Retention & Slot Behavior

* WAL cannot be recycled until **all** replication slots (and streaming replicas) have advanced past it. The **oldest** restart_lsn among all slots is the effective “retention point.”
* Slow or inactive slots **increase WAL retention** (more segment files on disk) and can lead to **WAL bloat** and storage pressure.
* If **max_slot_wal_keep_size** is set (PostgreSQL 13+), the server may **drop WAL** beyond that size even if a slot has not consumed it. The slot then becomes **lost** and is no longer usable until recreated. Use this parameter to cap WAL retention at the cost of possibly invalidating lagging slots.
* Regular monitoring is essential to avoid WAL bloat and to catch unhealthy slots early.

### Slot health: wal_status

| wal_status     | Meaning                                                                                                                                                                | Color Hint              |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| **reserved**   | Required WAL is within normal limits (`max_wal_size`).                                                                                                                 | 🟢 Green = healthy      |
| **extended**   | More WAL is retained than usual (e.g. slot is lagging); files are still kept. Indicates elevated WAL retention.                                                        | 🟡 Yellow = warning     |
| **unreserved** | Some required WAL is scheduled for removal at the next checkpoint (e.g. due to `max_slot_wal_keep_size`). Slot may soon become **lost**.                               | 🟠 Orange = action soon |
| **lost**       | Required WAL has been removed; the slot is **no longer usable**. The slot must be dropped and recreated; the subscriber must be re-set up from a new consistent state. | 🔴 Red = critical       |

**Useful Queries (color-friendly for Confluence with code macros):**

```sql
-- Highlight slots with extended or lost WAL
SELECT slot_name,
       wal_status,
       CASE
           WHEN wal_status = 'lost' THEN ' Critical'
           WHEN wal_status = 'unreserved' THEN 'Warning'
           WHEN wal_status = 'extended' THEN 'Warning'
           ELSE 'Healthy'
       END AS health_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots;
```


**Useful Queries:**

```sql
-- List all replication slots with key columns (logical slots have database set)
SELECT slot_name, slot_type, database, active, restart_lsn, confirmed_flush_lsn, wal_status
FROM pg_replication_slots;

-- Current WAL position (LSN) on this instance
SELECT pg_current_wal_lsn();

-- Per slot: how much WAL this slot prevents from being recycled (bytes between restart_lsn and current LSN)
-- High values = slot is lagging or inactive; WAL is piling up because of this slot
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots
WHERE restart_lsn IS NOT NULL;

-- Logical slots only: how far behind the consumer is (lag in bytes)
-- confirmed_flush_lsn = LSN up to which the subscriber confirmed receipt
SELECT slot_name, restart_lsn, confirmed_flush_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_bytes
FROM pg_replication_slots
WHERE slot_type = 'logical' AND confirmed_flush_lsn IS NOT NULL;

-- Warn if any slot retains > 1GB WAL (monitoring example)
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots
WHERE pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 1024*1024*1024;
```

---

## 5. Memory & Performance Hints

* **Slot memory:** Each logical replication slot uses memory on the **publisher** to track progress and (when active) to decode WAL. Heavy write activity with many active slots can increase memory and CPU use on the publisher.
* **WAL retention:** Ensure **sufficient disk space** for WAL. In multi-slot environments, the slowest slot dictates retention; one inactive or slow slot can cause WAL to grow large.
* **Consume quickly:** Subscribers should **consume WAL promptly**. Slow apply on the subscriber keeps the slot’s restart_lsn (and confirmed_flush_lsn) behind, which increases WAL retention on the publisher.
* **Drop slot when removing replication:** When tearing down logical replication, **drop the subscription on the subscriber first**, then **drop the slot on the publisher** (e.g. `pg_drop_replication_slot('slot_name')`). If you only drop the subscription and leave the slot, the slot will retain WAL indefinitely until the slot is dropped.
* **Cluster sizing:** For many databases and heavy write workloads, consider **partitioning write-heavy tables**, batching writes, or using **multiple clusters** for isolation.

**Best Practices:**

* Monitor slot lag regularly using `pg_replication_slots`.
* Drop unused slots immediately.
* Set `max_slot_wal_keep_size` to prevent uncontrolled WAL growth.
* Prefer batching writes for high-volume tables to reduce WAL pressure.

---

## 6. FAQs

**Q1: Does each database have its own WAL?**
**A:** No. WAL is cluster-wide. All changes from all databases write to the same WAL stream and segment files.

**Q2: Does adding more replication slots increase WAL generation?**
**A:** No. Slots only **consume** and **retain** WAL; they do not cause more WAL to be written.

**Q3: Can a replication slot ignore changes from other databases?**
**A:** Yes. A logical slot is created **in one database** and decodes only WAL that affects that database (and only tables in its publication). Changes in other databases are not delivered to that slot’s subscriber.

**Q4: How do I monitor WAL retention by slots?**
**A:** Query `pg_replication_slots` and use `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)` per slot to see how much WAL each slot is retaining. Check `wal_status` for slot health (Section 4).

**Q5: How many WAL files will 3 databases generate?**
**A:** It depends on total **WAL written** across the whole cluster, not on the number of databases. Each 16 MB of WAL written produces one segment file. More databases do not multiply WAL; more writes do.

**Q6: How do I prevent WAL bloat from slow or inactive slots?**

* Monitor slot lag and `wal_status` regularly (Section 4).
* Remove or fix lagging subscribers so slots can advance.
* When removing replication: drop the subscription, then **drop the slot** on the publisher. Do not leave unused slots.
* Optionally set `max_slot_wal_keep_size` to cap WAL retention (at the risk of slots becoming **lost** if they fall too far behind).

**Q7: Where does the replication slot live—publisher or subscriber?**
**A:** On the **publisher**. The subscriber connects to the publisher and consumes from the slot. Slot state (restart_lsn, confirmed_flush_lsn) is stored on the publisher.

**Q8: What if wal_status is "lost"?**
**A:** The slot is no longer usable because required WAL has been removed (e.g. due to `max_slot_wal_keep_size` or manual removal). Drop the slot (`pg_drop_replication_slot`). Recreate the subscription and slot from a consistent state (e.g. re-setup replication with a fresh initial copy if needed).

**Q9: What is the difference between restart_lsn and confirmed_flush_lsn?**
**A:** **restart_lsn** is the oldest WAL the slot might still need; it drives WAL retention. **confirmed_flush_lsn** (logical slots only) is the LSN up to which the subscriber has confirmed receipt—it reflects consumer progress. Lag in bytes is `pg_current_wal_lsn() - confirmed_flush_lsn`.

**Common Misconceptions:**

* More databases ≠ more WAL files.
* More slots ≠ more WAL generation.
* Logical slots filter WAL, but WAL is **always cluster-wide**.

---

## 7. Troubleshooting Checklist

| Symptom                                | What to check                                                              | Action                                                                                                                                   |
| -------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| WAL disk usage growing                 | `pg_replication_slots`: which slot has the smallest (oldest) restart_lsn?  | That slot is holding WAL. Fix or remove the lagging subscriber; drop unused slots.                                                       |
| Slot not advancing                     | Is the subscriber connected and applying? Check `active` and `active_pid`. | Ensure subscriber is running and not blocked; check apply lag and errors on the subscriber.                                              |
| wal_status = **extended**              | Slot is lagging; more WAL is retained than normal.                         | Reduce write load, speed up the subscriber, or drop the slot if replication is no longer needed.                                         |
| wal_status = **lost**                  | Required WAL was removed; slot is unusable.                                | Drop the slot; re-create replication (subscription + slot) from a consistent state.                                                      |
| Unused slot after removing replication | Subscription was dropped but slot left on publisher.                       | Connect to the **publisher** and run `SELECT pg_drop_replication_slot('slot_name');` (after terminating any backend using it if needed). |

---

## 8. References

* [PostgreSQL Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
* [WAL (Write-Ahead Logging)](https://www.postgresql.org/docs/current/wal-intro.html)
* [Streaming replication / Replication slots](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION-SLOTS)
* [pg_replication_slots view](https://www.postgresql.org/docs/current/view-pg-replication-slots.html)
* [max_slot_wal_keep_size](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-MAX-SLOT-WAL-KEEP-SIZE) (limits WAL retained for slots; can cause slots to become **lost**)

---

This guide complements [DATABASE_REPLICATION.md](DATABASE_REPLICATION.md), which describes how CRDR uses logical replication for tenant and global databases on Aurora PostgreSQL.

---

✅ **Additions I made without removing anything:**

1. Multi-slot diagram for cluster WAL consumption.
2. Best practices section inside Memory & Performance hints.
3. Monitoring query for WAL >1GB as a practical alert example.
4. Common misconceptions section in FAQs for clarity.
5. Slight clarification notes in WAL flow and slot explanation for newcomers.

