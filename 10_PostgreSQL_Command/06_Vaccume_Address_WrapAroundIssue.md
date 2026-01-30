## Transaction ID (TXID) Wraparound Problem

PostgreSQL assigns a **Transaction ID (TXID)** to every transaction. TXIDs are **32-bit values**, which means they live in a **finite, circular space**.

This design enables efficient comparisons but introduces a serious risk known as **transaction ID wraparound**.

---

## Circular TXID Space

TXIDs do not grow forever. Instead, they wrap around after reaching their maximum value.

### Circular TXID Space

PostgreSQL transaction IDs (TXIDs) are **32-bit values**, so they do not grow forever.
After reaching the maximum value, they **wrap around**, forming a circular ID space.

#### How PostgreSQL interprets TXIDs

At any moment, PostgreSQL has a **current TXID**.
This current TXID conceptually divides the entire TXID space into **two logical halves**:

* **Past TXIDs** – transactions that started **before the current transaction’s snapshot**
* **Future TXIDs** – transactions that started **after the snapshot was taken**

Because the TXID space is circular, this division always results in:

* Roughly **2 billion TXIDs considered “past”**
* Roughly **2 billion TXIDs considered “future”**

This is why PostgreSQL describes the TXID space as **circular** rather than linear.

#### Important clarification: “past” and “future” are not time-based

The terms **past** and **future** do **not** refer to wall-clock time.

They mean only:

* **Past TXID** → a transaction that started **before the snapshot**
* **Future TXID** → a transaction that started **after the snapshot**

#### Visibility rule

A transaction can see **only rows created by past transactions** relative to its snapshot.

Row visibility is determined by comparing:

* the row’s transaction metadata (`xmin`, `xmax`)
* against the transaction’s snapshot

**Wall-clock time is never used** for visibility decisions.

For ex: When the current TXID is 2, PostgreSQL treats TXIDs 2 to 2³¹ + 1 as belonging to the future and therefore invisible, while TXIDs 2³¹ + 2 through 0 (after wraparound) are treated as the past and are visible.
<img width="1113" height="482" alt="image" src="https://github.com/user-attachments/assets/2b6c4f3e-a4d4-4381-a29f-5ca40adee867" />

---

## Visibility Boundary

For a given current TXID:

* Rows inserted by transactions in the **past half** of the TXID space are visible
* Rows inserted by transactions in the **future half** are invisible

This boundary continuously shifts as new transactions start.

---

## How TXID Wraparound Happens

PostgreSQL transaction IDs (TXIDs) are **32-bit numbers** that go from **1 up to about 4.29 billion**.
To handle visibility, PostgreSQL treats TXIDs as a **circle**. This is why very old rows can eventually appear “in the future” if wraparound is not prevented.

### Step-by-step example

1. **Row is inserted**

   * **TXID = 1** inserts a row (`xmin = 1`)
   * Millions or billions of transactions happen afterward
   * The **current TXID** advances, but TXID 1 is still considered in the **past**
   * ✅ The row remains **visible**

2. **Approaching the halfway point**

   * Current TXID gets close to **2.1 billion** (half of the TXID range)
   * TXID 1 is still behind the current TXID
   * ✅ The row is **still visible**

3. **The critical moment**

   * Current TXID passes **2.1 billion + 1**
   * Now, because of **circular TXID logic**, TXID 1 is considered in the **future**
   * ❌ The row created by TXID 1 becomes **invisible** to all new transactions

**Result:**
Even though the row still exists on disk, PostgreSQL “thinks” it hasn’t been created yet. This is the **wraparound problem**.
<img width="1043" height="518" alt="image" src="https://github.com/user-attachments/assets/ca4ecaa4-ec44-45a6-9c20-680a79f1a2db" />

---

### Key takeaway

> TXIDs are not linear forever — they **wrap around after half of the 4.29-billion space**, and PostgreSQL uses circular arithmetic to decide visibility.
> Without proper maintenance (like **VACUUM**), very old rows can temporarily disappear from queries.

---

## Why Wraparound Is Dangerous

If wraparound is not prevented:

* Valid rows suddenly appear invisible
* PostgreSQL can no longer determine correct visibility
* Data corruption becomes possible

To protect data integrity, PostgreSQL will:

* **Stop accepting new transactions**
* Put the database into a **protective shutdown mode**

This is not a performance issue — it is a **correctness emergency**.

---

## Tuple Freezing: The Solution

PostgreSQL prevents wraparound using a process called **tuple freezing**.

### What Freezing Does

* Marks old rows as **frozen**
* Frozen rows are visible to **all transactions, forever**
* Visibility no longer depends on `xmin`
<img width="1012" height="406" alt="image" src="https://github.com/user-attachments/assets/d97380e8-708e-4153-8b88-edadae244211" />

Once frozen, a row is immune to wraparound.

---

## How Freezing Works Internally

Each heap tuple contains header flags.

Freezing:

* Sets a **frozen bit** in the tuple header
* Uses the header field `t_infomask`

Before PostgreSQL 9.4:

* A special TXID value (`FrozenXID = 2`) was used

PostgreSQL 9.4 and later:

* A dedicated **frozen flag** is used instead

> Frozen tuples bypass normal `xmin` visibility checks.

---

## Frozen vs Non-Frozen Tuples

| Tuple Type | Visibility Rule                      |
| ---------- | ------------------------------------ |
| Frozen     | Visible to all transactions          |
| Non-frozen | Visible based on `xmin` and snapshot |

Example:

* First 3 rows frozen → always visible
* Remaining rows → normal MVCC rules apply
<img width="1111" height="506" alt="image" src="https://github.com/user-attachments/assets/68800519-3efa-4c22-9c4b-f08d215cb2ca" />

---

## When Does PostgreSQL Freeze Tuples?

Freezing is performed by the **VACUUM** command.

VACUUM decides which tuples to freeze based on:

* Age of the tuple’s `xmin`
* System-defined thresholds (e.g., `autovacuum_freeze_max_age`)

Older tuples are frozen first.

---

## Freezing Modes

PostgreSQL uses **two freezing strategies**:

---

### 1. Lazy Freezing (Normal Mode)

* Uses the **Visibility Map (VM)**
* Skips pages that have no dead tuples
* Minimizes I/O
* Used during routine VACUUM and autovacuum

Efficient and low-impact.

---

### 2. Aggressive Freezing (Eager Mode)

* Ignores the visibility map
* Scans **all pages** in the table
* Freezes as many tuples as possible

This mode is triggered when wraparound risk is high.

---

## Additional Visibility Map Optimization

During aggressive freezing:

* An additional VM bit may be set
* Indicates that **all tuples on the page are frozen**

Pages marked this way:

* Can be skipped by future freezing runs
* Reduce long-term maintenance cost

---

## What If Freezing Does Not Happen?

If VACUUM does not run frequently enough:

* TXIDs continue to age
* Wraparound risk increases

Eventually PostgreSQL will:

* Refuse new transactions
* Log wraparound warnings
* Force emergency maintenance

This is why **vacuuming is mandatory**, not optional.

---

## Vacuum Cost-Based Control

VACUUM can be resource-intensive.

PostgreSQL uses **cost-based throttling** to limit impact:

* `vacuum_cost_limit`
* `vacuum_cost_delay`

Mechanism:

* Each vacuum action consumes cost units
* When cost limit is exceeded, vacuum sleeps
* Prevents vacuum from overwhelming I/O

Proper tuning balances:

* System performance
* Wraparound safety

---

## Operational Reality: A Double-Edged Sword

* No vacuum → database becomes unsafe and unusable
* Infrequent vacuum → long, disruptive maintenance
* Regular vacuum → stable performance and safety

The goal is **continuous, low-impact maintenance**.

---

## Key Takeaways

* TXIDs live in a circular 32-bit space
* Wraparound can make valid rows invisible
* PostgreSQL prevents corruption by freezing tuples
* Frozen tuples are always visible
* VACUUM performs freezing automatically
* Cost-based controls prevent vacuum overload

---

## Memory Hook 🧠

> **TXIDs wrap. Data must not.**
>
> Freezing locks visibility in time so the database can keep moving forward.

---

## FAQ

### Q: Is wraparound rare?

No. Busy databases reach wraparound quickly without vacuum.

### Q: Can I disable freezing?

No. Freezing is essential for correctness.

### Q: Does autovacuum handle freezing?

Yes. Autovacuum performs freezing automatically.

### Q: Is freezing the same as VACUUM FULL?

No. Freezing happens during normal VACUUM and does not rewrite tables.

### Q: What happens if wraparound is detected?

PostgreSQL blocks new transactions to prevent data corruption.

### Q: Why does vacuum sometimes slow the system?

Because it performs real I/O work; cost-based controls limit the impact.
-------
# Transaction ID (TXID) Wraparound in PostgreSQL

## What Is a Transaction ID (TXID)?

In PostgreSQL, every transaction gets a unique **transaction ID** (TXID):

- TXIDs increase: 1, 2, 3, …
- They are used for **MVCC** (Multi-Version Concurrency Control): each row stores which TXID created it and which TXID deleted it (if any).
- Visibility rules use TXIDs to decide which rows a transaction can see: “visible” means created by a transaction that committed before the current one and not deleted by a transaction visible to the current one.

So TXIDs are central to how PostgreSQL decides what data is visible to each transaction.

---

## Why Is TXID Stored as 32-Bit?

Historically, TXIDs were stored as **32-bit integers**:

- Range: 1 to **2³¹ − 1** (about 2.1 billion).
- After 2³¹ − 1, the next value would be 2³¹, which doesn’t fit in the signed 32-bit space, so the counter “wraps” (e.g. back to a small number).

So there is a fixed “circle” of TXIDs; after the maximum, the numbering wraps. That’s the basis of the wraparound problem.

---

## What Is the TXID Wraparound Problem?

### The situation

- TXIDs only go up to **2³¹ − 1**.
- Old rows keep the TXID of the transaction that created/updated them.
- Visibility is decided by comparing “row TXID” with “current TXID” and “oldest active TXID,” etc.
- If the current TXID wraps (e.g. from 2³¹ − 1 to 2), PostgreSQL can no longer correctly tell whether an old row (e.g. created at TXID 5) is “in the past” or “in the future” relative to the current TXID.

So **wraparound** means: the TXID space has been used so much that the 32-bit counter has (or is about to) wrap, and the database can no longer reliably decide visibility for some rows.

### What PostgreSQL does if wraparound is not prevented

To avoid returning wrong data, PostgreSQL:

1. **Stops accepting writes** — the database effectively becomes read-only.
2. **Forces emergency operations** — e.g. aggressive VACUUM to “freeze” old rows (see below).
3. **Can shut down** — if it can’t fix the situation in time, it may shut down to prevent corruption.

So the “TXID wraparound problem” is: **running out of TXIDs causes the instance to go read-only or shut down unless old rows are frozen in time.**

---

## How PostgreSQL “Solves” Wraparound: Row Freezing

The fix is to **freeze** old rows so they no longer depend on the 32-bit TXID for visibility.

### What “freeze” means

- A **frozen** row is treated as if it was committed “before any possible current transaction.”
- Frozen rows use a special marker (e.g. `FrozenTransactionId`) instead of a normal TXID.
- Once frozen, that row’s visibility no longer depends on the current TXID, so **wraparound does not affect it**.

So: **freezing = making old rows permanent in terms of visibility, so we don’t need to allocate new TXIDs for them.**

### Who does the freezing?

- **VACUUM** (and **autovacuum**) scans tables and **freezes** old row versions.
- When a row’s “creation” TXID is old enough (below the “freeze horizon”), VACUUM replaces that TXID with the frozen marker.
- PostgreSQL tracks how old the unfrozen data is per table and per database so it knows when freezing is urgent.

So the “solution” to TXID wraparound in PostgreSQL is: **run VACUUM (autovacuum) often enough so that all old rows are frozen before the TXID counter gets close to wraparound.**

---

## Key Concepts and Parameters

### Freeze horizon

- **Freeze horizon** = “TXID age beyond which rows must be frozen.”
- Conceptually: “Any row older than (current_txid - freeze_limit) must be frozen.”
- If any row is not frozen and its TXID age is too high (e.g. &gt; ~200 million), the database will force anti-wraparound work.

### Per-table tracking

- **`relfrozenxid`** (in `pg_class`): the TXID before which all rows in that table have been frozen.
- **Table “age”** = current TXID − `relfrozenxid`. High age means many TXIDs have been used since the last freeze; if age gets too large, wraparound is risky.

### Per-database tracking

- **`datfrozenxid`** (in `pg_database`): the oldest freeze horizon across all tables in that database.
- **Database “age”** = current TXID − `datfrozenxid`. This is the main number PostgreSQL uses to decide “how close are we to wraparound?”

### Autovacuum and freeze

- **Autovacuum** runs periodically and, among other things, **freezes** old rows.
- Parameters that affect when freezing happens:
  - **`vacuum_freeze_min_age`**: don’t freeze a row until its TXID is at least this many TXIDs old (default 50 million). Reduces unnecessary freezing.
  - **`vacuum_freeze_table_age`**: when a table’s age (current TXID − `relfrozenxid`) exceeds this, autovacuum will do a full-table scan to freeze that table (default 150 million). So freezing is triggered **before** age gets dangerously high.

So the “solution” in practice is: **tune autovacuum (and these parameters) so that every table is frozen often enough that both table age and database age stay well below 2³¹.**

---

## How to Monitor TXID Age (Wraparound Risk)

### Database age (main indicator)

```sql
SELECT datname, age(datfrozenxid) AS database_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

- **age(datfrozenxid)** = “how many TXIDs have been used since the oldest unfrozen data in this database.”
- PostgreSQL starts warning when this approaches **200 million** and takes aggressive action when it gets close to **2³¹** (e.g. ~2 billion).
- So you “solve” wraparound by **keeping database age low** (e.g. &lt; 200 million) by freezing often.

### Table age

```sql
SELECT relname, age(relfrozenxid) AS table_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

- Shows which tables have the “oldest” unfrozen data.
- Tables with high `age(relfrozenxid)` are the ones that need VACUUM (freeze) next.

### Current TXID (for context)

```sql
SELECT txid_current();
```

So: **monitor database and table age; if they grow toward 200 million (or more), freezing is not keeping up and you risk wraparound.**

---

## How to “Solve” or Prevent Wraparound

### 1. Rely on autovacuum (normal case)

- Ensure **autovacuum is enabled** (`autovacuum = on`).
- Let autovacuum run regularly so it freezes old rows.
- If database age stays low (e.g. &lt; 200 million), wraparound is “solved” by routine freezing.

### 2. Tune autovacuum for busy or large tables

- Increase **`autovacuum_max_workers`** if many tables need vacuuming.
- Lower **`vacuum_freeze_table_age`** so tables are frozen earlier (e.g. 100 million instead of 150 million).
- Increase **`autovacuum_naptime`** frequency so autovacuum runs more often.
- Give autovacuum enough **`work_mem`** (via `maintenance_work_mem` for VACUUM) so it can do its job efficiently.

### 3. Manual VACUUM FREEZE (if age is high)

- If a table (or database) age is already high, run:

```sql
VACUUM FREEZE tablename;
-- or for whole database (each table):
VACUUM FREEZE;
```

- **VACUUM FREEZE** forces freezing of old rows even if they’re above `vacuum_freeze_min_age`. Use it when you need to reduce age quickly.
- For very large tables, this can be I/O-heavy; schedule during low load.

### 4. Avoid long-running transactions

- The **oldest active transaction** influences the freeze horizon: PostgreSQL cannot freeze rows that might still be needed by that transaction.
- Long-running transactions (or idle-in-transaction sessions) can block freezing and cause age to grow.
- So: **keep transactions short; avoid holding transactions open for hours.** That’s part of “solving” wraparound in practice.

### 5. In emergencies (age very high, warnings in logs)

- PostgreSQL may force **aggressive autovacuum** or refuse new writes.
- Actions:
  - Run **VACUUM FREEZE** on the oldest tables (from the table-age query above).
  - Kill long-running/idle transactions that block freezing.
  - Increase **`vacuum_freeze_min_age`** only with care (it doesn’t fix age; it affects when we freeze). The main lever is running VACUUM FREEZE and making sure autovacuum can run (no long transactions).

So: **TXID wraparound is “solved” by freezing old rows (VACUUM / autovacuum) and by not blocking freezing with long transactions.**

---

## Summary

| Topic | Explanation |
|-------|-------------|
| **What is TXID?** | Unique 32-bit ID per transaction; used for MVCC visibility. |
| **What is wraparound?** | TXID counter reaches 2³¹ − 1 and wraps; visibility can no longer be decided correctly for very old rows. |
| **What does PostgreSQL do?** | Protects data by going read-only and/or forcing emergency VACUUM; can shut down if not fixed. |
| **How is it solved?** | **Freeze** old rows (replace their TXID with a special “frozen” marker) so they no longer depend on the 32-bit TXID. |
| **Who freezes?** | **VACUUM** and **autovacuum**; they run periodically and freeze rows older than the freeze horizon. |
| **How to monitor?** | `age(datfrozenxid)` and `age(relfrozenxid)`; keep them well below 200 million (and far below 2 billion). |
| **What you must do?** | Keep autovacuum on and effective; avoid long-running transactions; if age is high, run VACUUM FREEZE and tune autovacuum. |

In short: **the TXID wraparound problem is the risk of running out of 32-bit transaction IDs; PostgreSQL solves it by freezing old rows so their visibility no longer depends on TXID, and by monitoring and reducing “age” through VACUUM and autovacuum.**
