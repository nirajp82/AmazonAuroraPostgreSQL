# PostgreSQL Hands-On: VACUUM vs VACUUM FULL

## What this exercise demonstrates

This hands-on exercise demonstrates the **practical difference between regular (lazy) VACUUM and VACUUM FULL** by observing:

* Table size on disk
* Number of heap pages
* Free space inside pages

You will clearly see **why VACUUM does not shrink tables** and **why VACUUM FULL does**, along with the trade-offs involved.

This is a critical concept for understanding table bloat, disk usage, and maintenance strategies in PostgreSQL.

---

## Tools and extensions used

Two PostgreSQL extensions are used to inspect internal storage behavior:

### 1. `pg_freespacemap`

* Shows free space available inside each heap page
* Helps visualize internal fragmentation caused by dead tuples

### 2. `pgstattuple`

* Provides statistics such as:

  * Live tuples
  * Dead tuples
  * Table size
  * Free space percentage

> These extensions allow us to *measure* the effect of VACUUM operations instead of relying on theory alone.

---

## Exercise structure

The exercise is divided into **three parts**:

1. Test environment setup
2. Regular (lazy) VACUUM
3. VACUUM FULL

The goal is to compare **table size and page count before and after each step**.

---

## Part 1 – Test environment setup

### Step 1: Create required extensions

```sql
CREATE EXTENSION pg_freespacemap;
CREATE EXTENSION pgstattuple;
```

> If the extensions already exist, PostgreSQL will raise an error — this can be safely ignored.

---

### Step 2: Truncate the test table

```sql
TRUNCATE TABLE test;
```

Effects of TRUNCATE:

* Deletes all rows instantly
* Releases storage back to the operating system
* Resets the table to **zero pages**

At this point:

* Table size = 0
* Number of pages = 0

---

### Step 3: Insert test data

Insert a large number of rows to force PostgreSQL to allocate many heap pages.

```sql
INSERT INTO test (id)
SELECT generate_series(1, 1000000);
```

What this does:

* Inserts **1 million rows** into the table
* Forces allocation of thousands of 8 KB heap pages
* Creates a realistic dataset to observe bloat and vacuum behavior

### Step 3a: Create index on test table

```sql
CREATE INDEX idx_test_id ON test(id);
```

Why this matters:

* Ensures index maintenance work during VACUUM
* Demonstrates how dead tuples also create **dead index entries**
* Makes the VACUUM vs VACUUM FULL difference more realistic

> If the index already exists, PostgreSQL will raise an error. You can safely ignore it or use `IF NOT EXISTS`.

---

### Step 4: Delete a large portion of rows

```sql
DELETE FROM test WHERE id > 1000 AND id < 600000;
```

Important behavior:

* Rows are **not physically removed**
* They are marked as **dead tuples**
* Storage remains allocated

This creates **table bloat**, which VACUUM is designed to manage.

---

### Step 5: Measure table size and page count

At this stage:

* Table size ≈ **35 MB**
* Number of heap pages ≈ **4425**

These values are obtained using:

* `pgstattuple` for size
* `pg_freespacemap` for page count

---

## Part 2 – Regular (lazy) VACUUM

### Step 6: Run VACUUM

```sql
VACUUM test;
```

What regular VACUUM does:

* Removes dead tuples logically
* Makes space reusable **inside the table**
* Updates visibility map and statistics
* **Does NOT release disk space to the OS**

---

### Step 7: Measure again

After VACUUM:

* Table size ≈ **35 MB** (unchanged)
* Number of pages ≈ **4425** (unchanged)

Why nothing changed:

* VACUUM is **concurrent and non-blocking**
* It does not rewrite the table
* It only prepares internal free space for reuse

This behavior is intentional and expected.

---

## Part 3 – VACUUM FULL

### Step 8: Delete more rows

```sql
DELETE FROM test WHERE id > 900000;
```

This creates additional dead tuples and increases internal fragmentation.

---

### Step 9: Run VACUUM FULL

```sql
VACUUM (FULL) test;
```

What VACUUM FULL does:

* Rewrites the entire table
* Physically removes dead tuples
* Compacts data pages
* Releases unused disk space to the OS
* Requires an **exclusive lock** on the table

---

### Step 10: Measure final results

After VACUUM FULL:

* Table size ≈ **10 MB**
* Number of pages ≈ **2132**

This confirms:

* Dead space has been removed
* Table storage has been compacted
* Disk space has been reclaimed

---

## Key observations

| Operation   | Table Size Shrinks | Page Count Reduces | Locks Table |
| ----------- | ------------------ | ------------------ | ----------- |
| VACUUM      | ❌ No               | ❌ No               | ❌ No        |
| VACUUM FULL | ✅ Yes              | ✅ Yes              | ✅ Yes       |

---

## Memory hook

**VACUUM cleans inside the table. VACUUM FULL rebuilds the table.**

If disk space matters → VACUUM FULL
If availability matters → VACUUM

---

## Common misconceptions

### ❓ Why didn’t VACUUM reduce table size?

Because VACUUM does not rewrite the table. It only marks space as reusable.

### ❓ Why does VACUUM FULL need a lock?

Because it creates a new physical copy of the table and swaps it in.

### ❓ Should VACUUM FULL be used regularly?

No. It is expensive and blocks access. Use it sparingly.

---

## When to use each

* **VACUUM**

  * Routine maintenance
  * Autovacuum
  * High-availability systems

* **VACUUM FULL**

  * Severe table bloat
  * Disk pressure
  * Maintenance windows

---

## What to try next

* Compare autovacuum vs manual VACUUM
* Observe index size changes
* Monitor `pg_stat_user_tables`
* Experiment with `vacuum_cost_delay`
