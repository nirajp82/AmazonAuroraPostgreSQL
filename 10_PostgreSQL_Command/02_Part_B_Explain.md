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
* Highly selective predicates: Highly selective predicates: conditions that match only a small fraction of rows in a table, making indexes highly effective. If a query returns 0.01% of the table (like a specific User ID), the predicate is highly selective because it successfully excluded almost everything else.
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

When PostgreSQL executes a query with joins, the **planner considers multiple ways** to execute the join and chooses the one with the **lowest estimated cost**. Choosing the right join strategy is **critical for performance**, especially on large tables.

The three main join strategies are:

| Join Type   | When Used         | Good When                       | Bad When                                 |
| ----------- | ----------------- | ------------------------------- | ---------------------------------------- |
| Nested Loop | Small outer table | Indexed inner table             | Large × large tables without indexes     |
| Hash Join   | Equality joins    | Large unsorted data             | Low memory, non-equality joins           |
| Merge Join  | Sorted inputs     | Already sorted / ordered output | Inputs unsorted and sorting is expensive |

---

### Key Concepts Before Diving In

1. **Outer table vs Inner table**:

   * Outer table → the table scanned **row by row**.
   * Inner table → the table that is **probed/matched** against each row of the outer table.
   * The planner often picks the **smaller table (after filtering) as outer** to reduce work.

2. **Filtered tables can be “small”**:

   * Even if a table has 100,000 rows, a `WHERE` clause can reduce the number of rows drastically.
   * Example: `WHERE a.id < 100` reduces a 100k row table to ~99 rows → planner treats it as small for Nested Loop.

3. **Memory hooks**:

   * Think of **Nested Loop = outer × inner lookups**
   * **Hash Join = build once, probe many**
   * **Merge Join = walk two sorted lists**

---

### Nested Loop Join

```sql
EXPLAIN
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id
WHERE a.id < 100;
```

**How Nested Loop Works**

1. Planner chooses the **filtered table_a** as the **outer table** (small: ~99 rows).
2. table_b is the **inner table** (larger, 100k rows) with an **index on id**.
3. For each row in the outer table (table_a), the executor searches the inner table (table_b) for matching rows, repeating this process for every outer row — hence the term “Nested Loop.”
4. Output of inner table joins is passed up to parent nodes.

**Memory Hook**:

> Nested Loop = outer × inner lookup

**Why it’s good**:

* Outer table is small after filter
* Inner table has an index → fast lookups
* Small result set

**Why it’s bad**:

* Outer table is large
* Inner table has no index
* Performance: O(N × M) → very slow for large tables

---

### Hash Join (Most Common for Large Tables)

```sql
EXPLAIN
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id;
```

**How Hash Join Works**

1. Planner identifies equality join condition (`a.id = b.id`)
2. Builds a **hash table** on the smaller input (could be table_a or table_b depending on row estimates)
3. Probes hash table with rows from the other table
4. Matches are output to parent nodes

**Memory Hook**:

> Hash Join = build once, probe many

**Why it’s good**:

* Large tables with equality joins
* Input order doesn’t matter

**Why it’s bad**:

* Low memory environment → hash may spill to disk
* Non-equality joins

**Example Visualization**:

```
Build hash on table_a (100k rows)
Probe with table_b (100k rows)
Output matching rows
```

---

### Merge Join

```sql
EXPLAIN
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id
ORDER BY a.id;
```

**How Merge Join Works**

1. Planner requires **sorted inputs** (or will sort them first).
2. Executor **walks two sorted lists simultaneously**, matching rows according to join condition.
3. Output passes to parent nodes (aggregates, filters, etc.).

**Memory Hook**:

> Merge Join = walk two sorted lists

**Why it’s good**:

* Large datasets
* Ordered output required (ORDER BY, GROUP BY)
* No additional hash memory needed

**Why it’s bad**:

* Sorting cost is high if inputs are not already sorted
* Non-sorted inputs without indexes → planner may avoid Merge Join

**Example Visualization**:

```
Sorted table_a: 1, 2, 3, 4
Sorted table_b: 2, 3, 5
Walk both lists → output matching rows: 2,3
```

---

### Choosing the Best Join (Planner Trade-offs)

| Factor                        | Effect on Planner Choice                         |
| ----------------------------- | ------------------------------------------------ |
| Outer table size after filter | Small → Nested Loop; Large → Hash/Merge          |
| Index availability            | Indexed inner → Nested Loop is efficient         |
| Equality vs inequality        | Equality → Hash Join; Others → Nested Loop/Merge |
| Sorted inputs / ORDER BY      | Merge Join preferred if sorted                   |
| Memory limits                 | Low memory may prevent Hash Join                 |

---

### Example: Outer Table After Filter

```sql
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id
WHERE a.id < 100;
```

* table_a = **outer** (small after filter)
* table_b = **inner** (large, indexed)
* Result: Nested Loop is ideal here → 99 × index lookups

> Without filtering: 100,000 × 100,000 → Nested Loop would be very slow

---

### Summary

| Join Type   | Key Idea                                  | Memory Hook            |
| ----------- | ----------------------------------------- | ---------------------- |
| Nested Loop | Outer row × inner table lookup            | outer × inner lookup   |
| Hash Join   | Build hash on smaller input, probe larger | build once, probe many |
| Merge Join  | Walk two sorted inputs simultaneously     | walk two sorted lists  |

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
