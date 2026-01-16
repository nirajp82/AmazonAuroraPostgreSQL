## Objective

Understand **how to interpret PostgreSQL EXPLAIN output in depth** by:

* Creating sample tables
* Running EXPLAIN on progressively complex queries
* Understanding **scan nodes, operation nodes, and join strategies**
* Knowing **when a plan is good vs bad**
* Learning **how to reason like the PostgreSQL planner**

This README is meant for **deep learning and long-term reference**, not quick tips.

---

## Test Environment Setup

### Tables

We create two tables with identical structure:

```sql
CREATE TABLE table_a (
    id   INT,
    text TEXT
);

CREATE TABLE table_b (
    id   INT,
    text TEXT
);
```

### Populate Data (100,000 rows each)

```sql
INSERT INTO table_a
SELECT i, 'text_' || i
FROM generate_series(1, 100000) AS i;

INSERT INTO table_b
SELECT i, 'text_' || i
FROM generate_series(1, 100000) AS i;
```

### Index (Only on table_a)

```sql
CREATE INDEX idx_table_a_id ON table_a(id);
```

This asymmetry is intentional so we can **observe planner behavior differences**.

---

## How PostgreSQL Executes a Query Plan

* **EXPLAIN shows the plan tree produced by the planner**: This tree represents the planner’s chosen execution strategy, including scan methods, join algorithms, and operation order, but not the actual execution results.
* **Execution happens bottom-up**: The executor begins with the lowest-level nodes (leaf nodes) that fetch actual rows from tables or indexes, then passes their output to parent nodes for operations like filtering, aggregation, or joining. For example, in a query joining table_a and table_b, the executor first reads rows from each table (leaf nodes), then the join node combines them, and finally the root node returns the result to the client. Understanding leaf nodes is critical because inefficient scans here can slow the entire execution tree.
* **Leaf nodes fetch data**: These nodes (Seq Scan, Index Scan, Index Only Scan, Bitmap Scan) are responsible for physically retrieving rows from tables or indexes.
* **Parent nodes perform operations (filter, join, aggregate)**: Parent nodes consume rows from child nodes and apply logical operations such as WHERE filtering, JOIN conditions, sorting, grouping, and aggregation.
* **Root node returns results to the client**: The topmost node represents the final step where fully processed rows are returned to the client connection.

### Cost Format

```
cost = startup_cost .. total_cost
```

* **Startup cost**: work before first row is returned
* **Total cost**: work to return all rows

🧠 Memory Hook:

> *Startup = before first row, Total = before last row*

---

## Scan Nodes (Leaf Nodes)

Scan nodes fetch rows from storage. They dominate performance.

### Sequential Scan (Seq Scan)

```sql
EXPLAIN
SELECT * FROM table_b WHERE id = 10;
```

**Why Seq Scan?**

* No index exists on `table_b.id`
* Planner must scan entire table

**When Seq Scan is GOOD**

* Small tables
* Queries returning large % of rows
* No usable index

**When Seq Scan is BAD**

* Large tables
* Highly selective predicates: conditions that match only a small fraction of rows in a table, making indexes highly effective.  If a query returns 0.01% of the table (like a specific User ID), the predicate is highly selective because it successfully excluded almost everything else.
* Index exists but not used

🧠 Memory Hook:

> *Seq Scan = read everything, decide later*

---

### Index Scan

```sql
EXPLAIN
SELECT * FROM table_a WHERE id = 10;
```

**Why Index Scan?**

* Index exists on `table_a.id`
* Fetch row pointers from index, then fetch table rows

**When Index Scan is GOOD**

* Highly selective queries
* Small result sets

**When Index Scan is BAD**

* Query returns most rows
* Random I/O dominates

🧠 Memory Hook:

> *Index Scan = index first, table later*

---

### Index Only Scan

```sql
EXPLAIN
SELECT id FROM table_a WHERE id = 10;
```

**Why Index Only Scan?**

* Query only needs indexed column
* Table access avoided if visibility map allows

**When Index Only Scan is GOOD**

* Read-heavy workloads
* Covering indexes

**Why It Sometimes Falls Back**

* Rows not marked all-visible

🧠 Memory Hook:

> *Index Only Scan = table never touched*

---

## Aggregation Nodes

Aggregation nodes compute values like MIN, MAX, SUM, COUNT.

### MIN / MAX with Index (Very Efficient)

```sql
EXPLAIN
SELECT MIN(id) FROM table_a;
```

* Uses Index Only Scan
* Uses LIMIT internally

**Why Fast?**

* Index already sorted
* Stops after first row

🧠 Memory Hook:

> *MIN/MAX + index = instant*

---

### SUM / AVG (Index Not Helpful)

```sql
EXPLAIN
SELECT SUM(id) FROM table_a;
```

* Sequential Scan
* Aggregate node

**Why Index Not Used?**

* All rows must be read

🧠 Memory Hook:

> *SUM/AVG = touch every row*

---

## Sorting and Grouping Nodes

### ORDER BY Using Index

```sql
EXPLAIN
SELECT * FROM table_a ORDER BY id;
```

* Index Scan
* No Sort node

### ORDER BY Without Index

```sql
EXPLAIN
SELECT * FROM table_b ORDER BY id;
```

* Seq Scan
* Explicit Sort node

🧠 Memory Hook:

> *ORDER BY loves indexes*

---

### GROUP BY

```sql
EXPLAIN
SELECT id, COUNT(*) FROM table_a GROUP BY id;
```

* Seq Scan
* Hash Aggregate or Group Aggregate

**Important:**

* Index may not help unless order matches grouping

---

## Join Strategies (Critical for Performance)

Planner evaluates **multiple join paths** and selects the cheapest.

### Join Strategy Summary

| Join Type   | When Used      | Good When           | Bad When            |
| ----------- | -------------- | ------------------- | ------------------- |
| Nested Loop | Small inputs   | Indexed inner table | Large × large joins |
| Hash Join   | Equality joins | Large unsorted data | Low memory          |
| Merge Join  | Sorted inputs  | Ordered data        | Sorting required    |

---

### Nested Loop Join

```sql
EXPLAIN
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id
WHERE a.id < 100;
```

**Why Nested Loop?**

* Small outer input
* Index on inner side

**When GOOD**

* Small result sets
* Index available

**When BAD**

* Large joins without index
* O(N × M) behavior

🧠 Memory Hook:

> *Nested Loop = outer × inner lookup*

---

### Hash Join (Most Common)

```sql
EXPLAIN
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id;
```

**Why Hash Join?**

* Equality condition
* No ordering required

**How It Works**

1. Build hash table on smaller input
2. Probe with larger input

**When GOOD**

* Large equality joins

**When BAD**

* Memory constrained systems
* Non-equality joins

🧠 Memory Hook:

> *Hash Join = build once, probe many*

---

### Merge Join

```sql
EXPLAIN
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id
ORDER BY a.id;
```

**Why Merge Join?**

* Inputs already sorted or sortable

**When GOOD**

* Large datasets
* Ordered output required

**When BAD**

* Sorting cost dominates

🧠 Memory Hook:

> *Merge Join = walk two sorted lists*

---

## How to Judge a Plan Like a Pro

### Good Signs

* Index scans for selective queries
* Hash joins for large equality joins
* Estimated rows close to actual rows

### Red Flags

* Seq Scan on large filtered tables
* Nested Loop on large joins
* Row misestimation by 10× or more

---

## What to Do When a Plan Looks Bad

| Issue                     | Action                     |
| ------------------------- | -------------------------- |
| Seq Scan instead of Index | Create index               |
| Wrong join type           | ANALYZE tables             |
| Big row misestimate       | Increase statistics target |
| Hash spill to disk        | Increase work_mem          |
| Slow aggregation          | Materialized view          |

---

## Memory Hooks (Quick Recall)

* Seq Scan → read everything
* Index Scan → index then table
* Index Only Scan → index only
* MIN/MAX → instant with index
* SUM/AVG → full scan
* Nested Loop → small first
* Hash Join → equality joins
* Merge Join → sorted inputs

---

## FAQ

### Why does PostgreSQL ignore my index?

Planner estimates index is more expensive due to data distribution.

### Is Seq Scan always bad?

No. It is optimal for large result sets.

### Which join is fastest?

Depends on data size, indexes, and memory.

### When should I use EXPLAIN ANALYZE?

Always when performance matters—it shows actual execution.

### What separates a beginner from an expert?

Understanding planner trade-offs, not memorizing node names.

---
