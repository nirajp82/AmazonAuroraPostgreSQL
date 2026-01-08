## 1. The Core Conflict: Speed vs. Data Safety

When you run a SQL command (`INSERT`, `UPDATE`, `DELETE`), PostgreSQL **does not write directly to disk**.

Instead, it first modifies data in memory, inside the **Shared Buffer Pool (RAM)**.

### The Problem: Dirty Pages

* After a page is modified in Shared Buffers, it becomes a **dirty page**
* At that point:
  * Shared Buffers contain the newer version
  * The data file on disk contains the older version
  * The disk copy is updated later by BGWriter or Checkpointer
* If PostgreSQL crashes before these pages are written to disk, the changes (Newer version) are lost because it hasn't reached the disk (It was only in RAM Memory)

### The Naive (Slow) Solution

* Write every change immediately to data files on disk
* This causes **random I/O** across many files
* Results in massive **I/O spikes** and very poor performance

#### Example: I/O Spike

If 1,000 users click **Save** at the same time:

* PostgreSQL would need to write to many different files and locations
* Disk heads must constantly seek new positions
* The disk becomes overwhelmed → severe slowdown

### The PostgreSQL Solution: WAL

Instead of writing data files immediately, PostgreSQL:

* Records every change in a **sequential log**
* This log is the **Write-Ahead Log (WAL)**
```sql
LSN 100: T1 - INSERT row A
LSN 101: T2 - UPDATE row B
LSN 102: T1 - COMMIT
LSN 103: T2 - COMMIT
```
Sequential writes are **fast**, predictable, and disk-friendly.
- Note: There is **one logical Write-Ahead Log (WAL)** that records all changes in sequence, and internally it is **physically stored as multiple fixed-size files (e.g., 16 MB segments)** for manageability and recovery purposes.

---

## 2. Write-Ahead Log (WAL)

Think of WAL as a **durable ledger** that records every change **before** it is applied to data files.

### How WAL Works

* **WAL Buffer:** Changes are first written to an in-memory WAL buffer
* **WAL Writer Process:** Flushes WAL buffer contents to disk
* **Commit Rule:**
  A transaction is considered **committed only after its WAL record is safely written to disk**
* **WAL Segments:** WAL is stored in **16 MB files**
* **LSN (Log Sequence Number):**
  Every WAL record has a unique, increasing LSN

> **Important:**
> A transaction is *durable* once its WAL record reaches disk — even if data files are not updated yet.
<img width="999" height="554" alt="image" src="https://github.com/user-attachments/assets/eb754675-710e-47dd-9f0c-dbdd15e8374b" />

---

## 3. Checkpointing

Eventually, PostgreSQL must apply WAL changes to the actual data files.
This process is called **Checkpointing**.
<img width="982" height="474" alt="image" src="https://github.com/user-attachments/assets/39c83638-6346-4a2f-8c31-1bcffa6115e3" />

### What Happens During a Checkpoint

1. **Redo Point**

   * PostgreSQL identifies the last checkpoint’s ending LSN. (In this case LSN# 99)
2. **Checkpoint Record**

   * A new checkpoint record is written to the WAL (LSN# 102, Changes after #102 are new changes by other transaction)
3. **Candidate Pages**

   * All dirty pages modified between the redo point and checkpoint record become candidates for flushing
4. **Flush to Disk**

   * The **Checkpointer** writes these dirty pages to data files
   * This is where **I/O spikes** usually occur
5. **Control File Update**

   * Metadata is stored in `pg_control`

---

### When Do Checkpoints Occur?

Checkpoints prevent WAL from growing indefinitely.

1. **Time-Based**

   * Controlled by `checkpoint_timeout`
   * **Default:** 5 minutes

2. **Size-Based**

   * Controlled by `max_wal_size`
   * **Default:** 1 GB
   * Forces a checkpoint when WAL grows too large

3. **Manual**

   * Triggered using:

     ```sql
     CHECKPOINT;
     ```

> **Note:**
> I/O spikes are most noticeable during checkpoints because large volumes of dirty pages are written to disk.

---

## 4. Crash Recovery (Redo Process)

If PostgreSQL crashes (PANIC), the **Postmaster** restarts the system and begins recovery.
<img width="980" height="433" alt="image" src="https://github.com/user-attachments/assets/0c4b08a1-1712-4d0e-a6ca-6901255877b2" />

### Recovery Steps

1. **Read `pg_control`**

   * Identify the last checkpoint and redo LSN (Uses PG_Control file)
2. **Locate WAL Records**

   * Find WAL entries written after the last checkpoint
3. **Redo Changes**

   * Replay WAL records into data pages
4. **Finalize**

   * Write a new checkpoint
   * Database becomes consistent again

> **Recovery Time Depends On:**
> Amount of WAL since the last checkpoint
> More frequent checkpoints = faster recovery

---

## 5. Background Processes

### Background Writer (BgWriter)

**Purpose:** Reduce checkpoint pressure and user-visible latency

* Shared Buffers are limited
* Pages must be evicted to load new data
* Evicting dirty pages is expensive
<img width="967" height="444" alt="image" src="https://github.com/user-attachments/assets/2255384e-2000-44ff-a174-be594edc7e62" />

**What BgWriter Does**

* Proactively writes dirty pages to disk
* Targets pages likely to be evicted soon
* Keeps buffers clean ahead of time

**Benefit**

* User queries don’t stall waiting for disk writes

---

### WAL Archiver

**Purpose:** Enable Point-In-Time Recovery (PITR)

* Completed WAL segments are archived
* Stored in external storage (NFS, S3, etc.)

**Benefits**

* Frees primary disk space
* Allows database recovery to any moment in time

---

## 6. Critical PostgreSQL Background Processes

| Process               | Responsibility                                           |
| --------------------- | -------------------------------------------------------- |
| **WAL Writer**        | Flushes WAL buffer to disk so transactions can commit    |
| **Checkpointer**      | Writes dirty pages to data files and controls WAL growth |
| **Background Writer** | Smooths I/O by pre-cleaning dirty buffers                |
| **WAL Archiver**      | Copies completed WAL segments to archive storage         |

---

### Q: When a record is inserted, how do Shared Buffers and the Write-Ahead Log (WAL) store the change?

**Example:**

```sql
INSERT INTO Customer (Id, Name)
VALUES (1, 'JDoe');
```

### A: Shared Buffers and WAL store **different representations** of the same operation, each for a specific purpose.

---

## High-level view

### Shared Buffers (Actual Data)

Think of Shared Buffers as an **in-memory copy of table pages**.

After the insert, a data page in Shared Buffers looks like:

```
Customer Table – Data Page (in memory)
-------------------------------------
| Id | Name |
-------------------------------------
| 1  | JDoe |
-------------------------------------
```

* Contains the **real row data**
* Page is marked **dirty**
* Written to disk later by BGWriter or Checkpointer

---

### WAL (Write-Ahead Log

Think of WAL as a **sequential instruction log**.

A WAL record looks like (conceptual):

```
WAL Record
----------
Action  : INSERT
Table   : Customer
PageId  : 123
Slot    : 4
Values  : (1, 'JDoe')
LSN     : 0/16B6A90
```

* Stores **how to reapply the change**
* Append-only and sequential
* Flushed on commit to guarantee durability

---

## Why they are different

* Shared Buffers are optimized for **fast reads and writes**
* WAL is optimized for **crash recovery**
* WAL does **not** represent table data directly

---

## What happens step-by-step

```
INSERT statement
   ↓
Modify data page in Shared Buffers
   ↓
Write corresponding WAL record
   ↓
Commit → WAL flushed to disk
   ↓
Later → Data page written to data file
```

---

## Important note (edge case)

* After a checkpoint, the first change to a page may log the **entire page image** in WAL for safety.
* Even then, Shared Buffers still hold the authoritative row data.

---

## Summary

> Shared Buffers contain the actual table rows, while WAL contains a sequential description of changes that allows the database to recover those rows after a crash.

**Q:** If BGWriter writes modified pages to the data files, does that mean the corresponding WAL records are no longer needed for data files and are therefore ready for archival?

### Clarified Answer

**A:** Not exactly.
Even after BGWriter writes modified pages to the data files, the corresponding WAL records may still be required. BGWriter’s role is only to **gradually flush dirty pages** to reduce I/O pressure; it does **not** define a recovery boundary.

WAL records become safe for **archival or reuse only after a checkpoint** is completed. A checkpoint ensures that all data pages up to a specific WAL position are persisted to disk and records this state in the WAL itself. Only then are earlier WAL segments no longer needed for crash recovery.

**In summary:**

> WAL records are not made archivable by BGWriter writes; they become archivable only after a checkpoint establishes a recovery point, at which time those WAL segments are used for archival and recovery purposes—not for writing data files.


## 7. Summary

* **WAL** ensures durability using fast sequential writes
* **Commit** happens only after WAL is flushed to disk
* **Checkpoints** sync memory with data files
* **BgWriter** prevents sudden I/O pressure
* **Archiving** enables long-term recovery options
