## Overview

This lesson focuses on the **PostgreSQL VACUUM command**, its commonly used options, and the **database configuration parameters** that control its behavior and impact on the system. A major part of the lesson explains how VACUUM performs **tuple freezing** and how PostgreSQL decides between **lazy freezing** and **eager (aggressive) freezing**.

Understanding these concepts is critical for preventing transaction ID (XID) wraparound and for maintaining consistent query performance.

---

## What the VACUUM Command Does

The VACUUM command serves multiple purposes:

* Reclaims space occupied by dead tuples
* Prevents transaction ID wraparound by freezing old tuples
* Updates visibility information used by the optimizer
* Optionally updates statistics for the query planner

If you run:

```sql
VACUUM;
```

PostgreSQL performs a **concurrent (lazy) vacuum** on **all tables in the database**.

You can also vacuum a specific table:

```sql
VACUUM tablename;
```

> Available VACUUM options depend on the PostgreSQL version.

---

## Common VACUUM Options

### VERBOSE

```sql
VACUUM VERBOSE;
```

* Prints detailed progress information
* Useful for understanding what VACUUM is doing internally
* Disabled by default

---

### FULL

```sql
VACUUM FULL;
```

* Performs **full vacuuming**
* Rewrites the table completely
* Requires an **exclusive lock** on the table
* Reclaims disk space back to the operating system
* Much slower than regular VACUUM

---

### ANALYZE

```sql
VACUUM ANALYZE;
```

* Updates table and column statistics
* Statistics are stored in the system catalog
* Used by the query planner to choose efficient execution plans
* Column-level vacuuming is only possible when ANALYZE is specified

---

### FREEZE

```sql
VACUUM FREEZE;
```

* Forces **aggressive (eager) freezing**
* Freezes qualifying tuples regardless of age thresholds
* Typically used during wraparound risk situations

---

### DISABLE_PAGE_SKIPPING

* Affects freezing behavior
* Prevents VACUUM from skipping pages based on the visibility map
* Forces scanning of all pages for freeze candidates
* Useful when you want maximum freezing coverage

---

### SKIP_LOCKED

* If a table is locked by another transaction, VACUUM skips it instead of waiting
* Prevents VACUUM from blocking

---

### INDEX_CLEANUP

* Controls whether index cleanup is performed
* Disabling index cleanup can speed up VACUUM
* Dead index entries may remain temporarily

---

### TRUNCATE

* Controls whether empty pages at the end of a table are truncated
* If set to FALSE, truncation is skipped

---

## VACUUM and Freezing Modes

VACUUM can perform tuple freezing in **two modes**:

* **Eager (Aggressive) Freezing**
* **Lazy Freezing**

The choice between these modes depends on:

* Database configuration parameters
* System catalog values (`pg_database`, `pg_class`)

---

## System Catalogs Involved

### `pg_database`

* Contains database-wide freezing information
* Key column:

  * `datfrozenxid`: Oldest transaction ID already frozen at the database level

### `pg_class`

* Contains table-level freezing information
* Key column:

  * `relfrozenxid`: Oldest transaction ID frozen in a specific table

---

## Eager (Aggressive) Freezing

Eager freezing is triggered when the database approaches XID wraparound risk.

### Trigger Condition (Conceptual)

Aggressive freezing occurs when:

```
datfrozenxid < (oldest_active_xid - vacuum_freeze_table_age)
```

* `vacuum_freeze_table_age` default: **150 million transactions**

This ensures that after a certain number of transactions, VACUUM aggressively freezes data.

---

### Freeze Limit XID

Before freezing begins, PostgreSQL calculates the **freeze limit XID**:

```
freeze_limit_xid = oldest_active_xid - vacuum_freeze_min_age
```

* `vacuum_freeze_min_age` default: **50 million transactions**

Any tuple with an XID **less than this limit** is eligible for freezing.

---

### Eager Freezing Process

1. VACUUM checks `pg_class.relfrozenxid`
2. If the table qualifies, all pages are scanned
3. For each page:

   * If page is already marked frozen → skip
   * Otherwise, scan tuples
4. Tuples with XID < freeze limit are marked frozen
5. If all tuples in a page are frozen:

   * Page is marked frozen in the visibility map
6. After completion:

   * `pg_class.relfrozenxid` is updated
   * `pg_database.datfrozenxid` may be updated
   * Statistics are refreshed

---

## Lazy Freezing

Lazy freezing happens during **regular VACUUM** (without FULL or FREEZE).

Key characteristics:

* VACUUM only scans pages that contain **dead tuples**
* Page skipping is controlled by the **visibility map**
* Pages without dead tuples are skipped entirely

### Impact

* Some freeze-eligible tuples may be skipped
* Freezing progresses gradually over multiple VACUUM runs

---

### Freeze Limit in Lazy Mode

Freeze limit is calculated similarly:

```
freeze_limit_xid = oldest_active_xid - vacuum_freeze_min_age
```

Example:

* Oldest active XID = 50,000,1000
* `vacuum_freeze_min_age` = 50,000,000
* Freeze limit = 1000

Tuples with XID < 1000 are frozen **only if the page is scanned**.

---

### DISABLE_PAGE_SKIPPING Effect

Using:

```sql
VACUUM DISABLE_PAGE_SKIPPING;
```

* Forces scanning of all pages
* Makes lazy freezing behave closer to eager freezing
* Increases runtime but improves freeze coverage

---

## Key Differences: Eager vs Lazy Freezing

| Aspect              | Eager Freezing  | Lazy Freezing               |
| ------------------- | --------------- | --------------------------- |
| Trigger             | Wraparound risk | Regular VACUUM              |
| Page scanning       | All pages       | Only pages with dead tuples |
| Uses visibility map | Limited         | Heavily                     |
| Freezing coverage   | Complete        | Partial                     |
| Runtime cost        | High            | Lower                       |

---

## Memory Hook 🧠

**Think of freezing like renewing an ID card:**

* Lazy freezing renews IDs only when you visit the office
* Eager freezing forces everyone to renew before IDs expire

PostgreSQL *must* freeze aggressively before XIDs wrap around.

---

## FAQ

### Why is freezing necessary?

Without freezing, old transaction IDs become ambiguous after wraparound, leading to data corruption or incorrect query results.

---

### Why does VACUUM sometimes skip pages?

In lazy mode, VACUUM skips pages without dead tuples using the visibility map to reduce I/O.

---

### When should I use VACUUM FREEZE?

Only during wraparound risk situations or maintenance windows. It is aggressive and resource-intensive.

---

### Does VACUUM FULL prevent wraparound?

Yes, but it is disruptive. Regular VACUUM with freezing is preferred.

---

### Is lazy freezing enough?

Yes, **as long as autovacuum is enabled and runs frequently**. Otherwise, eager freezing becomes mandatory.

---

### Which system catalogs control freezing decisions?

* `pg_database` (database-level)
* `pg_class` (table-level)

---

## Key Takeaway

VACUUM is not just about cleaning dead rows—it is a **critical safety mechanism**. Understanding its options and freezing modes helps prevent catastrophic XID wraparound while balancing performance and availability.
