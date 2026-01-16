## How PostgreSQL Stores Data on Disk

To understand how **VACUUM** fixes table bloat, you must first understand how PostgreSQL stores table and index data on disk.

PostgreSQL data files are made up of **fixed-size 8 KB pages**.

* Files always grow in **8 KB increments**
* Reads and writes always happen in **8 KB chunks**

This design is fundamental to how space reuse and vacuuming work.

---

## Heap Page Structure (Table Pages)

Each table page consists of three logical areas:

1. **Page Header**
2. **Item Pointer Array (Line Pointers)**
3. **Heap Tuples (Actual Rows)**

### Page Header

* Stores metadata about the page
* Tracks free space boundaries

### Item Pointers (Line Pointers)

* A dynamic array in the page header
* Each entry points to a tuple in the heap
* Item pointer #1 points to the first row in the page

> Item pointers do not disappear when a row is deleted.

### Heap Tuples

* Actual row data
* Grows **from the end of the page backward** toward the header

Free space lives between the item pointer array and the heap tuples.

---

## How Inserts Work at Page Level

* New rows are placed in the free space of an existing page
* A new item pointer is added to reference the tuple
* When a page runs out of space:

  * A new 8 KB page is appended to the table file

Over time, a table accumulates many pages as data grows.

---

## Deletes, Updates, and Dead Tuples

Due to MVCC:

* DELETE does **not** remove the row
* UPDATE creates a new row version

Instead:

* Tuples are **marked dead** using row header metadata
* Item pointers continue to reference dead tuples

These dead tuples:

* Occupy disk space
* Are loaded into shared buffers
* Appear during sequential scans

This is known as **table bloat**.

---

## Index Pages and Dead Index Entries

Index files also consist of **8 KB pages**, but their layout differs from heap pages.

Key points:

* Index entries point to **heap tuple locations**
* When a heap tuple becomes dead:

  * The index entry is **not immediately removed**

As dead tuples accumulate:

* Dead index entries accumulate as well
* Index scans become less efficient

VACUUM cleans up **both heap tuples and index entries**.

---

## Why VACUUM Must Be Efficient

Scanning every page of every table repeatedly would be extremely expensive.

PostgreSQL uses **helper data structures** to make vacuuming fast and selective:

* Free Space Map (FSM)
* Visibility Map (VM)

---

## Free Space Map (FSM)

The **Free Space Map** tracks available free space on a **per-page basis**.

* Updated when tuples are deleted
* Helps PostgreSQL find pages with enough space for new rows

### pg_freespacemap Extension

PostgreSQL provides the `pg_freespacemap` extension to inspect FSM data.

This extension allows you to:

* See how much free space exists per page
* Understand fragmentation patterns

---

## Visibility Map (VM)

The **Visibility Map** tracks whether a page contains any dead tuples.

* Maintained **per page**, not per row

### Visibility Map Rules

| Page State              | VM Bit |
| ----------------------- | ------ |
| All tuples visible      | 1      |
| At least one dead tuple | 0      |

Whenever a tuple is deleted or updated:

* VM bit for that page is set to **0**

---

## How VACUUM Uses the Visibility Map

VACUUM checks the VM to decide:

* Pages with VM=1 → **skipped**
* Pages with VM=0 → **scanned and cleaned**

This avoids unnecessary sequential scans and makes vacuuming scalable.

---

## Types of VACUUM

PostgreSQL provides **two vacuuming modes**:

1. Concurrent VACUUM (VACUUM)
2. Full VACUUM (VACUUM FULL)

---

## Concurrent VACUUM (Standard VACUUM)

This is the most commonly used vacuuming mode.

### What It Does

* Removes dead tuples
* Cleans dead index entries
* Freezes old transaction IDs
* Updates FSM and VM

### What It Does NOT Do

* Does not shrink table files
* Does not return disk space to the OS

### Key Properties

* Does **not lock the table**
* Reads and writes continue during vacuum
* Page count before and after vacuum remains the same

Space is reused **inside existing pages**, not released.

---

## Full VACUUM (VACUUM FULL)

VACUUM FULL is a **heavyweight operation**.

### What It Does

* Acquires an **exclusive lock** on the table
* Rewrites the entire table into new files
* Compacts tuples into fewer pages
* Rebuilds all indexes
* Updates system catalogs, FSM, VM, and statistics

### Result

* Table size may shrink
* Disk space may be returned to the OS

---

## Why VACUUM FULL Is Dangerous on Large Tables

* Blocks all reads and writes
* Runtime can be hours or days
* High disk I/O usage (read + write)
* Requires extra temporary disk space

VACUUM FULL should be used:

* Only during maintenance windows
* Only when table bloat is severe

---

## Concurrent VACUUM vs VACUUM FULL

| Feature             | VACUUM | VACUUM FULL     |
| ------------------- | ------ | --------------- |
| Table lock          | No     | Yes (exclusive) |
| Removes dead tuples | Yes    | Yes             |
| Shrinks table       | No     | Yes             |
| Rebuilds indexes    | No     | Yes             |
| Safe for production | Yes    | No              |

---

## Key Takeaways

* PostgreSQL stores data in 8 KB pages
* Dead tuples cause table and index bloat
* FSM tracks reusable space
* VM tracks pages with dead tuples
* VACUUM reclaims space internally
* VACUUM FULL rebuilds tables and frees disk space

---

## Memory Hook 🧠

> **VACUUM cleans inside pages.**
>
> **VACUUM FULL rebuilds the house.**
>
> One is safe and routine — the other is disruptive but powerful.

---

## FAQ

### Q: Why doesn’t VACUUM reduce table size?

Because it only reclaims space inside existing pages, not release pages to the OS.

### Q: Why is VACUUM FULL so slow?

It rewrites the entire table and all indexes under an exclusive lock.

### Q: Does autovacuum run VACUUM FULL?

No. Autovacuum only runs standard VACUUM.

### Q: Can I stop a VACUUM FULL?

Yes, but it will roll back and waste the work already done.

### Q: Is VACUUM required even if I delete little data?

Yes. VACUUM also prevents transaction ID wraparound, not just bloat.
