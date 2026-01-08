## 1. The Core Conflict: Speed vs. Data Safety

When you run a SQL command (`INSERT`, `UPDATE`, `DELETE`), PostgreSQL **does not write directly to disk**.

Instead, it first modifies data in memory, inside the **Shared Buffer Pool (RAM)**.

### The Problem: Dirty Pages

* Modified memory pages are called **dirty pages**
* If PostgreSQL crashes before these pages are written to disk, the changes are lost because it hasn't reached the disk (It was only in RAM Memory)

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

Sequential writes are **fast**, predictable, and disk-friendly.

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

   * PostgreSQL identifies the last checkpoint’s ending LSN
2. **Checkpoint Record**

   * A new checkpoint record is written to the WAL
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

   * Identify the last checkpoint and redo LSN
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

## 7. Summary

* **WAL** ensures durability using fast sequential writes
* **Commit** happens only after WAL is flushed to disk
* **Checkpoints** sync memory with data files
* **BgWriter** prevents sudden I/O pressure
* **Archiving** enables long-term recovery options
