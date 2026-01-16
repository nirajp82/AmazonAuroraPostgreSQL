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

**How Hash Join Works (Step by Step)**

1. **Planner identifies equality join condition:**
   In this query, `a.id = b.id`. Hash Join only works efficiently for equality joins.

2. **Build phase (smaller table):**
   PostgreSQL builds a hash table in memory using all rows from the **smaller table** (in this example, assume **table_b**).

   * Each **hash entry** contains:

     1. **Hash key** – result of hashing `b.id`
     2. **Join key / ID** – the actual `b.id` value
     3. **Reference / pointer** – points to the full row in `table_b`

3. **Probe phase (larger table):**
   PostgreSQL reads each row from the **larger table** (`table_a`).

   * Computes the hash of `a.id`
   * Looks up matching hash entries in the hash table
   * Compares actual values to confirm matches (resolves hash collisions)
   * Combines the matching rows from `table_a` and `table_b`

4. **Output:**
   Matching rows are passed to the parent nodes for further processing (filtering, aggregation, etc.)

**Memory Hook:**

> *Hash Join = build hash table once on smaller table, probe many rows from larger table using hash + pointer*

---

**Why Hash Join is Good:**

* Large tables with equality join conditions
* Input order doesn’t matter (no sorting needed)

**Why Hash Join is Bad:**

* Low memory environment → hash may spill to disk
* Non-equality joins

**Example Visualization:**

| Step | Table           | Action                                                     |
| ---- | --------------- | ---------------------------------------------------------- |
| 1    | table_b (small) | Build hash table with [hash(id), id, pointer]              |
| 2    | table_a (large) | For each row, compute hash(a.id), probe table_b hash table |
| 3    | Both tables     | If match found, combine rows and send to parent node       |

---

Exactly! Let me clarify that in **Merge Join** and rewrite the section with that extra detail about indexed vs non-indexed tables.

---

### Merge Join (Efficient for Sorted Inputs)

```sql
EXPLAIN
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id
ORDER BY a.id;
```

**How Merge Join Works (Step by Step)**

1. **Planner identifies join condition:**
   `a.id = b.id` in this query. Merge Join works best when **inputs are already sorted** or can be sorted efficiently.

2. **Prepare sorted inputs:**

   * **table_a:** Already sorted because `a.id` has an index → no explicit sort needed
   * **table_b:** No index on `b.id` → PostgreSQL will use a **Sort node** to sort rows in memory before merge

3. **Merge phase:**

   * PostgreSQL walks through both sorted lists (table_a and table_b) **simultaneously**:

     1. Start with the first row of each table
     2. Compare `a.id` and `b.id`
     3. If they match, combine rows and send to parent node
     4. If not, advance the pointer of the smaller value to the next row
   * Repeat until all rows are processed

4. **Output:**
   Matched rows are sent to parent nodes for further operations (filtering, aggregation, etc.)

**Memory Hook:**

> *Merge Join = walk two sorted lists, combining matching rows efficiently*

---

**Why Merge Join is Good:**

* Large datasets where one or both tables are sorted on join keys
* Indexed table → no sort needed (saves time)
* Ordered output required (e.g., `ORDER BY`)
* Minimal random I/O, streaming merge

**Why Merge Join is Bad:**

* Sorting cost dominates if table has no index (like `table_b` in this case)
* Non-equality joins cannot use merge join
* Small tables with indexes → Nested Loop may be faster

**Example Visualization (Assume table_a has index, table_b has no index):**

| Step | Table        | Action                                        |
| ---- | ------------ | --------------------------------------------- |
| 1    | table_a      | Already sorted via index → used directly      |
| 2    | table_b      | Sort node sorts rows by `id`                  |
| 3    | Both tables  | Walk through rows in order, compare join keys |
| 4    | Matched rows | Send to parent nodes for further processing   |

---

**Key Insight:**

Merge Join is **streaming and memory-efficient**, especially when at least one table is indexed. The unindexed table is sorted once, then the merge can efficiently combine rows without repeated scans.

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

Got it! I can expand your FAQ section with more practical questions and answers for beginners and intermediate users, covering indexes, joins, performance issues, planner behavior, and EXPLAIN intricacies. Here’s the extended FAQ section you can append to your README:

---

## FAQ

### Why does PostgreSQL ignore my index?

* The planner might think a **Seq Scan is cheaper** based on row statistics or small table size.
* Highly selective predicates favor indexes; low selectivity might not.

### Is Seq Scan always bad?

* No. Seq Scan is efficient for:

  * Small tables
  * Queries returning a large portion of rows
  * Tables without useful indexes

### Which join is fastest?

* Depends on:

  * Table sizes (outer vs inner)
  * Index availability
  * Join condition (equality vs inequality)
  * Memory settings

* **Nested Loop** → small outer table + indexed inner table, works best when the outer table has few rows (after filtering) and the inner table has an index.
* **Hash Join** → large equality joins, when both tables are large and the join condition is an equality (like `a.id = b.id`).
* **Merge Join** → sorted inputs or ordered output, works best when one or both tables are already sorted on the join key or when an ordered result is required.

### When should I use EXPLAIN vs EXPLAIN ANALYZE?

* **EXPLAIN** → shows estimated plan, costs, and node types
* **EXPLAIN ANALYZE** → executes query, shows actual times and row counts
* Use EXPLAIN ANALYZE to validate planner estimates and find bottlenecks

### Why does my query sometimes use Seq Scan even with an index?

* Low table size → planner chooses simpler plan
* Statistics outdated → run `ANALYZE table_name`
* Highly non-selective query → index may not help
* Multiple conditions → planner may prefer a different scan path

### How do I know if a Nested Loop is a problem?

* Check estimated vs actual rows
* Large tables with no index → O(N×M) behavior
* Small filtered outer table + indexed inner table → good

### How does Hash Join handle large tables?

* Builds hash on smaller input
* Probes with larger input
* Low memory → hash may spill to disk (slows query)

### How does Merge Join handle unsorted tables?

* Sort nodes may be added for unindexed tables
* Sorting can be expensive for very large datasets
* Merge Join still efficient for large sorted data

### What is a highly selective predicate?

* Condition that matches **very few rows**
* Example: `WHERE id = 12345` in a table with 1 million rows
* High selectivity → index scans are effective

### Can aggregation use indexes?

* MIN/MAX → yes, if index on column
* SUM/AVG → no, must read all rows
* COUNT(*) → Seq Scan unless partial indexes or materialized views used

### How can I reduce planner misestimation?

* Regularly run `ANALYZE` on tables
* Ensure statistics target is sufficient for large tables
* Consider `EXPLAIN (ANALYZE, BUFFERS)` to see actual I/O

### How do I choose the right join type?

* **Nested Loop** → small outer table, indexed inner table
* **Hash Join** → equality joins, large unsorted tables
* **Merge Join** → sorted inputs or required ordered output
* Check EXPLAIN output to verify planner choice

### Why does Hash Join sometimes spill to disk?

* Insufficient `work_mem`
* Large table + low memory → temporary file created
* Increase `work_mem` for faster hash joins

### Should I always add indexes to speed up queries?

* Not always; indexes help selective queries
* Too many indexes → slow inserts/updates
* Use EXPLAIN to verify if a query would benefit

### How do I debug slow queries?

1. Run `EXPLAIN ANALYZE`
2. Check node types (Seq Scan vs Index Scan)
3. Check estimated vs actual rows
4. Check join types and memory usage
5. Optimize with indexes, rewritten queries, or materialized views

### How can I remember scan nodes and join types?

* 🧠 **Memory Hook:**

  * Seq Scan → read everything
  * Index Scan → index first, table later
  * Index Only Scan → table never touched
  * Nested Loop → outer × inner lookup
  * Hash Join → build once, probe many
  * Merge Join → walk two sorted lists

---
