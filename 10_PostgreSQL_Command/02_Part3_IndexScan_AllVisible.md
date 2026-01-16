# PostgreSQL Index-Only Scan Explained

**Goal:** Understand why Index-Only Scans sometimes fall back to heap fetches, how the **Visibility Map** works, and how to tune your table for maximum performance.

---

## 🔹 What is an Index-Only Scan?

An **Index-Only Scan** is the “holy grail” of query performance:

* PostgreSQL retrieves all required data from the **index** without touching the **heap** (the table).
* This dramatically reduces I/O and speeds up queries.

However, you may notice `Heap Fetches` in your execution plan even when using an Index-Only Scan. Why?

---

## 🔹 The "All-Visible" Requirement

PostgreSQL uses **MVCC (Multi-Version Concurrency Control)**, which tracks row visibility for each transaction.

### Problem:

* **Indexes** only store the indexed columns and a pointer to the row in the heap.
* Indexes **don’t know** if the row is “alive” for your transaction.

### Solution: The **Visibility Map (VM)**

* A **tiny bitmask**, one bit per page in the table, tells PostgreSQL if all rows on a page are visible to all transactions.

| Bit Value | Meaning                                                       |
| --------- | ------------------------------------------------------------- |
| 1         | All rows on the page are visible (“all-visible”)              |
| 0         | Some rows are recently updated or deleted; visibility unknown |

---

## 🔹 Why Heap Fetches Happen

During an Index-Only Scan:

* **Page is all-visible:** Postgres trusts the index → skips heap → fast!
* **Page is NOT all-visible:** Postgres must fetch the row from the heap to confirm visibility → results in `Heap Fetches`.

---

## 🔹 How to Fix "Heap Fetches"

1. **Run `VACUUM`**

   * Cleans up dead rows and updates the Visibility Map.

2. **Tune autovacuum**

   * Ensure `autovacuum` runs frequently enough for tables with heavy updates.

3. **Avoid constantly updated indexed columns**

   * Pages with frequent updates rarely become all-visible, negating index-only benefits.

---

## 🔹 Check Your Table’s Visibility

```sql
SELECT 
    relname, 
    relpages, 
    relallvisible, 
    (relallvisible::float / CASE WHEN relpages = 0 THEN 1 ELSE relpages END) * 100 AS percent_visible
FROM pg_class 
WHERE relname = 'your_table_name';
```

*Low `percent_visible` → Index-Only Scans will continue to fetch from the heap.*

---

## 🔹 Memory Hook / Analogy

Think of the **Visibility Map** as a **traffic light for table pages**:

* **Green light (1 / all-visible):** Postgres trusts the index → zoom past the heap.
* **Red light (0 / not all-visible):** Postgres must knock on the door (heap) to check row visibility.

---

## 🔹 Visual Diagram

```
          +-----------------+
          |      Index      |
          |----------------|
          | Row Pointers -> |
          +--------+--------+
                   |
                   v
          +-----------------+       +-------------------+
          |      Heap       | <---> | Visibility Map    |
          |  (table data)   |       |  (All-Visible?)   |
          +-----------------+       +-------------------+
```

*Index points to heap rows, but the VM tells PostgreSQL whether it can skip the heap.*

---

## 🔹 FAQ

**Q1: Why don’t indexes store visibility info themselves?**
A: MVCC requires row-level visibility per transaction. Storing that in the index would be too costly and constantly updated.

**Q2: How often should I run `VACUUM` manually?**
A: Usually rely on **autovacuum**. Manual `VACUUM` is mainly for bulk imports or if low all-visible pages are causing heap fetches.

**Q3: Can Index-Only Scan work on small tables without a Visibility Map?**
A: Yes. Small tables may be scanned quickly directly from the heap, so index-only benefit is minimal. VM is more critical for large tables.

**Q4: What if my index includes frequently updated columns?**
A: Pages with frequently updated columns rarely become all-visible, so index-only scans may still hit the heap often. Consider excluding such columns from the index if possible.

**Q5: How to check if autovacuum is keeping up?**
A: Query `pg_stat_all_tables` for `last_autovacuum` and `n_dead_tup` to see if autovacuum is running frequently enough for your workload.

---
