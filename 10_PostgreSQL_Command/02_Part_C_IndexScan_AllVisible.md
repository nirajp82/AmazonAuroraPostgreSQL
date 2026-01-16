# PostgreSQL Index-Only Scan & MVCC Explained

**Goal:** Understand why Index-Only Scans sometimes fall back to heap fetches, how the **Visibility Map (VM)** works, how **MVCC** manages row versions, and how to tune your table for maximum performance.

---

## 🔹 What is an Index-Only Scan?

An **Index-Only Scan** retrieves all required data from the **index** without touching the **heap**, reducing I/O.

However, `Heap Fetches` may still appear in your execution plan. Why?

---

## 🔹 How MVCC Works (Multi-Version Concurrency Control)

PostgreSQL allows multiple readers and writers to operate **without blocking each other**.

### How Versions Are Created

When you perform an `UPDATE`, PostgreSQL **does not modify the row in place**. It creates a **new physical row (tuple)**:

1. **Original State:** `[ID: 1, Balance: 100]` → **Version A**
2. **UPDATE:** `UPDATE accounts SET balance = 200 WHERE id = 1;`
3. **Result:** `[ID: 1, Balance: 200]` → **Version B**

Both rows exist simultaneously.

### Hidden Row Metadata (`xmin` / `xmax`)

| Column   | Meaning                                     |
| -------- | ------------------------------------------- |
| **xmin** | Transaction ID that created the row         |
| **xmax** | Transaction ID that updated/deleted the row |

---

## 🔹 Example: Two Users, Same Index, Different Snapshots

| Row Version | Data | xmin | xmax | Status                            |
| ----------- | ---- | ---- | ---- | --------------------------------- |
| Version A   | $100 | 90   | 100  | Being replaced by Transaction 100 |
| Version B   | $200 | 100  | 0    | Created by Transaction 100        |

Two users scan the same index **concurrently**:

| Transaction | Snapshot Timing                  | Legal Row | Heap Fetch Needed? |
| ----------- | -------------------------------- | --------- | ------------------ |
| User 1      | Before commit of Transaction 100 | Version A | Yes (VM = 0)       |
| User 2      | After commit of Transaction 100  | Version B | Yes (VM = 0)       |

### How It Works:

1. **User 1 scans the index** and sees both Version A and Version B pointers.
2. **Visibility Map shows page is not all-visible**, so Postgres fetches from heap.
3. Heap check shows **Version A** is legal for User 1 → returned.
4. **User 2 scans the same index** after Transaction 100 commits.
5. Heap check shows **Version B** is legal for User 2 → returned.

> 🔑 **Key Point:** The **Visibility Map** is **page-level and global**, indicating “some rows may be invisible,” but **each transaction independently verifies which row is legal** based on its snapshot.

---

## 🔹 Why Indexes Fall Back to Heap

* Indexes contain **data and row pointers only**, not xmin/xmax.
* To return the correct **legal row** for a transaction, Postgres must fetch the heap if:

  * The page is not marked **all-visible**
  * Multiple row versions exist

```markdown
### 🛠️ MVCC Mechanics (xmin/xmax)
Indexes store only the indexed values, not xmin/xmax. To ensure a transaction 
sees its "legal row," PostgreSQL must verify visibility via the Visibility Map 
or fetch xmin/xmax from the heap.
```

---

## 🔹 The "All-Visible" Requirement

| Bit Value | Meaning                                                   |
| --------- | --------------------------------------------------------- |
| 1         | All rows on the page are visible to all transactions      |
| 0         | Some rows have recent updates/deletes; visibility unknown |

* **All-Visible (1):** Index-only scan can skip the heap → fast for all transactions.
* **Not All-Visible (0):** Each transaction checks the heap individually → ensures correct **legal row**.

---

## 🔹 Why Heap Fetches Happen

* **Page all-visible:** Index-only scan skips heap → fast.
* **Page not all-visible:** Heap fetch required to determine **legal row** for each transaction → results in `Heap Fetches`.

---

## 🔹 How to Fix "Heap Fetches"

1. **Run `VACUUM`** → cleans dead rows and updates the VM.
2. **Tune autovacuum** → frequent updates keep pages all-visible.
3. **Avoid frequently updated indexed columns** → reduces pages marked not all-visible.

---

## 🔹 Check Table Visibility

```sql
SELECT 
    relname, 
    relpages, 
    relallvisible, 
    (relallvisible::float / CASE WHEN relpages = 0 THEN 1 ELSE relpages END) * 100 AS percent_visible
FROM pg_class 
WHERE relname = 'your_table_name';
```

---

## 🔹 Memory Hook / Analogy

**Visibility Map = traffic light for table pages:**

* **Green (1 / all-visible):** Skip heap.
* **Red (0 / not all-visible):** Check heap for each transaction’s **legal row**.

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

---

## 🔹 FAQ

**Q1: What is a “legal row”?**
A: The row version visible to your transaction snapshot. Other versions exist physically but are invisible (illegal) for your transaction.

**Q2: How does the VM work with multiple users?**
A: VM is **page-level and global**. If a page is **not all-visible**, each transaction still verifies the **legal row** independently.

**Q3: Why don’t indexes store visibility info themselves?**
A: MVCC requires row-level visibility per transaction. Storing it in the index would be too costly.

**Q4: How often should I run `VACUUM` manually?**
A: Rely on **autovacuum**, except for bulk imports or low all-visible pages.

**Q5: Can Index-Only Scan work on small tables without a VM?**
A: Yes. Small tables are fast to scan from heap.

**Q6: What if indexed columns are frequently updated?**
A: Heap fetches are likely because pages rarely become all-visible.

**Q7: How to check autovacuum activity?**
A: Query `pg_stat_all_tables` for `last_autovacuum` and `n_dead_tup`.

**Q8: Can I peek at `xmin`/`xmax` values?**

```sql
SELECT ctid, xmin, xmax, * FROM your_table_name;
```
