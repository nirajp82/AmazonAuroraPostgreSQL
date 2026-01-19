## Multi-Version Concurrency Control (MVCC / NBCC)

PostgreSQL uses **Multi-Version Concurrency Control (MVCC)**, sometimes referred to as **NBCC (Non-Blocking Concurrency Control)**, to achieve **consistency and isolation** without serializing transactions.

The core idea is simple but powerful:

* Readers never block writers
* Writers never block readers
* Multiple versions of the same row can exist at the same time

This design allows PostgreSQL to handle high concurrency while still providing strong transactional guarantees.

---

## Why PostgreSQL Uses MVCC

In a multi-user database, many transactions operate concurrently on the same data. Without proper control, this leads to incorrect results and data corruption.

Traditional locking-based systems prevent these issues by blocking access, but that severely hurts performance.

PostgreSQL avoids this by:

* Keeping **multiple versions of rows**
* Allowing each transaction to see a **consistent snapshot** of the database

MVCC ensures:

* **Isolation** without heavy locking
* **High concurrency**
* **Predictable query behavior**

---

## Transaction Isolation and Phenomena

Concurrent transactions can cause classic anomalies such as:

* Dirty reads
* Non-repeatable reads
* Phantom reads

PostgreSQL allows control over isolation behavior using:

```sql
SET TRANSACTION ISOLATION LEVEL ...;
```

### Default Isolation Level

* PostgreSQL default: **READ COMMITTED**
* Dirty reads are **never allowed**

> PostgreSQL intentionally does not support dirty reads at any isolation level.

---

## Core Concept: Row Versions

MVCC works by maintaining **multiple versions of the same row**, called **tuples**.

Each row stores transaction metadata in its **row header**, not in a separate structure.

Key fields in every row:

| Field  | Meaning                                             |
| ------ | --------------------------------------------------- |
| `xmin` | Transaction ID that inserted the row                |
| `xmax` | Transaction ID that deleted (or superseded) the row |

> A row is never physically deleted immediately.

### 1. The Core Mechanics: xmin and xmax

Every row in Postgres has hidden "system columns" that track its lifespan:

* **`xmin`**: The ID of the transaction that **created** (inserted) the row.
* **`xmax`**: The ID of the transaction that **deleted** or **updated** the row. (If it's `0`, the row hasn't been deleted).

---

### 2. How Actions Work

* **INSERT**: Creates a new row with `xmin = MyID` and `xmax = 0`.
* **DELETE**: Doesn't actually erase the row; it just sets `xmax = MyID`.
* **UPDATE**: This is a **DELETE + INSERT**. It sets `xmax` on the old version and creates a brand-new row with a new `xmin`.

---

## Transaction IDs (TXID)

* Every transaction gets a **monotonically increasing transaction ID (TXID)**
* TXID is a **32-bit integer**
* IDs are assigned whether the transaction commits or rolls back

TXID acts as a **version number** for visibility decisions.

You can inspect:

```sql
SELECT xmin, xmax FROM table_name;
SELECT txid_current();
```

---

## Visibility Rules (Simplified)

A row is visible to a transaction if:

* `xmin` is **committed** and **less than or equal to** the transaction’s snapshot
* `xmax` is either:

  * `0`, or
  * Greater than the transaction’s snapshot

These rules allow PostgreSQL to decide visibility **without locking**.

---

## INSERT Under MVCC

When a row is inserted:

* `xmin` = inserting transaction’s TXID
* `xmax` = `0`

Visibility rule:

* Row is visible **only to transactions that start after the inserting transaction commits**
<img width="1088" height="520" alt="image" src="https://github.com/user-attachments/assets/5f2554f9-d39f-43f8-b66d-b7562b8d316f" />

### Example

* TXID 10 inserts a row → `xmin=10`, `xmax=0`
* TXID 11 starts before TXID 10 commits → row is invisible
* TXID 12 starts after commit → row is visible

---

## DELETE Under MVCC

A delete:

* Does **not remove** the row
* Sets `xmax` to the deleting transaction’s TXID

Visibility rule:

* Transactions with TXID **less than `xmax`** still see the row
* Transactions with TXID **greater than `xmax`** do not

### Example
* TXID 13 → row visible
* TXID 14 deletes a row → `xmax=14`
* TXID 15 → row invisible
<img width="1100" height="496" alt="image" src="https://github.com/user-attachments/assets/d26b89c4-cf63-4bbc-86cd-a081b8d1a997" />

---

## UPDATE Under MVCC

An UPDATE is internally:

* A **DELETE** of the old row
* Followed by an **INSERT** of a new row

This means:

* Old row gets `xmax = updater TXID`
* New row gets `xmin = updater TXID`

> Updates require a **row-level lock** to avoid conflicting updates.

### Example

* Original row: `xmin=16, xmax=0`
* TXID 17 updates it

  * Old row → `xmax=17`
  * New row → `xmin=17, xmax=0`

Visibility:

* TXID 16 → sees old value that is id = 1
* TXID 18 → sees new value that is id = 2
<img width="1095" height="507" alt="image" src="https://github.com/user-attachments/assets/861f35c3-f128-4150-a8eb-746abdf2dc15" />

---

## Non-Blocking Reads and Writes

MVCC enables:

* Reads without blocking writes
* Writes without blocking reads

Locks are only used for:

* Preventing concurrent updates to the same row
* Structural changes (DDL)

This is why PostgreSQL scales well under read-heavy workloads.

---

## Challenges Introduced by MVCC

### 1. Dead Tuples (Row Bloat)

* Deleted or updated rows remain as **dead tuples**
* Dead tuples:

  * Consume disk space
  * Pollute buffer cache
  * Increase index size
  * Slow down scans

This condition is known as **table bloat**.

---

### 2. Transaction ID Wraparound

* TXID is 32-bit → ~4 billion values
* TXID space is **circular**
* After exhaustion, IDs wrap back to zero

If unmanaged:

* New transactions may appear **older** than existing rows
* Rows can become invisible to all new transactions
* Database becomes read-only to protect data integrity

This is called the **transaction wraparound problem**.

---

## VACUUM: The Solution

The `VACUUM` command addresses both MVCC challenges:

### What VACUUM Does

* Removes dead tuples
* Reclaims disk space
* Updates visibility maps
* Freezes old transaction IDs to prevent wraparound

Autovacuum usually handles this automatically, but understanding VACUUM is critical for performance and reliability.

---

## Key Takeaways

* PostgreSQL uses MVCC (NBCC) for concurrency control
* Rows are versioned, not overwritten or deleted immediately
* `xmin` and `xmax` control visibility
* Readers and writers do not block each other
* MVCC introduces dead tuples and wraparound risks
* VACUUM is essential for correctness and performance

---

## Memory Hook 🧠

> **PostgreSQL never overwrites rows.**
>
> It creates new versions, hides old ones, and cleans up later.
>
> MVCC gives speed — VACUUM pays the bill.

---

## FAQ

### Q: Why doesn’t PostgreSQL physically delete rows immediately?

Because doing so would break consistent snapshots for concurrent transactions.

### Q: Why are updates more expensive than inserts?

Updates create a new row version and mark the old one dead, doubling work.

### Q: Can MVCC cause performance issues?

Yes. Without regular vacuuming, dead tuples cause table and index bloat.

### Q: Is wraparound just theoretical?

No. Databases that ignore vacuuming can hit wraparound and become unusable.

### Q: Does MVCC eliminate all locking?

No. Row-level locks are still used for updates and deletes to prevent conflicts.

### Q: Who runs VACUUM?

Usually autovacuum. Manual VACUUM may be needed for heavily updated tables.
