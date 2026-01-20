## How PostgreSQL Stores Data on Disk

To understand how **VACUUM** fixes table bloat, you must first understand how PostgreSQL stores table and index data on disk.

### PostgreSQL Data File and Page Layout

PostgreSQL stores **relations** (tables, indexes, free space maps, and visibility maps) as **physical data files on disk**. Each relation is divided into **pages (blocks)** of **8192 bytes (8 KB)** by default.

* Pages are **numbered sequentially from 0**, called **block numbers**.
* When all existing pages in a data file are full, PostgreSQL **adds a new empty page to that file**.
* **File extension details:**

  * Each data file can grow up to **1 GB**.
  * If the relation exceeds this limit, PostgreSQL **creates additional files** with numeric suffixes (e.g., `tablename.1`, `tablename.2`) to continue storing pages.

#### Page Layout in Heap Tables

Each page contains **three main components**:

1. **Heap Tuples**

   * Store the actual **record data**.
   * Stacked **from the bottom of the page upward**.
   * Internal structure depends on **concurrency control (CC)** and **write-ahead logging (WAL)** (covered in Sections 5.2 and 9).

2. **Line Pointers (Item Pointers)**

   * Each is **4 bytes**, pointing to a heap tuple.
   * Form a **sequential array**, acting as an **index to the tuples**.
   * Numbered sequentially from **1**, called **offset numbers**.
   * Adding a new tuple automatically adds a **new line pointer** in the array.

3. **Header Data (`PageHeaderData`)**

   * Located at the **beginning of the page**, **24 bytes** long.
   * Stores general page information, including:

     * **pd_lsn** – LSN of the last XLOG change (8 bytes, WAL-related).
     * **pd_checksum** – Page checksum (PostgreSQL 9.3+; earlier versions stored `pd_tli`).
     * **pd_lower** – Points to the **end of line pointers**.
     * **pd_upper** – Points to the **start of newest tuple**.
     * **pd_special** – Reserved for indexes; in tables, points to **end of page**; in indexes, points to **special data area** (e.g., B-tree, GiST, GIN).

* **Free Space / Hole** – Space between the end of line pointers (`pd_lower`) and the start of the newest tuple (`pd_upper`).

#### Tuple Identification

* Each tuple is identified by a **Tuple Identifier (TID)**, which consists of:

  * **Block number** – Page containing the tuple.
  * **Offset number** – Line pointer pointing to the tuple.
* TID is commonly used in **indexes** for tuple referencing.
<img width="1897" height="645" alt="image" src="https://github.com/user-attachments/assets/54eea615-a797-4f5e-a255-195e3966932f" />

- Reference: http://interdb.jp/pg/pgsql01/03.html

---

## How Inserts Work at Page Level

* New rows are placed in the free space of an existing page
* A new item pointer is added to reference the tuple
* When a page runs out of space:

  * A new 8 KB page is appended to the table file

Over time, a table accumulates many pages as data grows.
<img width="673" height="230" alt="image" src="https://github.com/user-attachments/assets/6e4555dd-bc63-4ad1-90f7-4efaae7606bb" />

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
- In the below data page, item 3, 6 and 16 marked as deleted. 
<img width="677" height="232" alt="image" src="https://github.com/user-attachments/assets/edf0141a-a2a6-4023-9738-cdcbf3e7e9cd" />

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

The **Visibility Map** tracks whether a page contains any dead tuples. The basic concept of the VM is simple. Each table has an individual visibility map that holds the visibility of each page in the table file. The visibility of pages determines whether or not each page has dead tuples. Vacuum processing can skip a page that does not have dead tuples by using the corresponding visibility map (VM).

* Maintained **per page**, not per row

### Visibility Map Rules

| Page State              | VM Bit |
| ----------------------- | ------ |
| All tuples visible      | 1      |
| At least one dead tuple | 0      |

Whenever a tuple is deleted or updated:

* VM bit for that page is set to **0**

-Suppose that the table consists of three pages, and the 0th and 2nd pages contain dead tuples and the 1st page does not. The VM of this table holds information about which pages contain dead tuples. In this case, vacuum processing skips the 1st page by referring to the VM’s information.

- <img width="1865" height="798" alt="image" src="https://github.com/user-attachments/assets/6a8c2130-34fd-4129-a5f9-fd7531cb008b" />
  
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
<img width="1850" height="855" alt="image" src="https://github.com/user-attachments/assets/d9d22fc2-b9ac-45de-8389-de38ae713b9c" />

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
