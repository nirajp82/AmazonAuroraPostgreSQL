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

### Nested Loop Join (Best when the outer table is small and the inner table has an index)

```sql
EXPLAIN
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id
WHERE a.id < 100;
```

**How Nested Loop Works**

1. Planner chooses the **filtered table_a** as the **outer table** (small: ~99 rows).
2. table_b is the **inner table** (larger, 100k rows) with an **index on id**.(PostgreSQL does NOT blindly use Nested Loop just because the outer table is small. It uses Nested Loop only if the inner side can be accessed efficiently (usually via an index). Otherwise, it will often choose a Hash Join, even when one side is small.)
    - When b.Id is indexed: 99 outer rows × 1 index lookup each ≈ 99 operations (For each outer row (a.id), PostgreSQL does one index lookup in table_b)
    - When b.Id is not indexed: 99 outer rows × 100,000 inner rows = 9,900,000 comparisons (For each outer row, PostgreSQL must scan all 100,000 rows of table_b)
4. For each row in the outer table (table_a), the executor searches the inner table (table_b) for matching rows, repeating this process for every outer row — hence the term “Nested Loop.”
5. Output of inner table joins is passed up to parent nodes.

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
### Hash Join (Mostly common for Large tables + equality join, no index required)

Query:

```sql
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id;
```

### How Hash Join Works

1. **Planner detects equality condition**

   ```
   a.id = b.id
   ```

2. **Planner chooses the smaller input to build the hash table**
   Assume (for this example):

   * `table_b` is chosen as the **build side**
   * `table_b` → 100,000 rows
   * `table_a` → 100,000 rows (probe side)

3. **Build phase (done once)**

   PostgreSQL scans **table_b once** and builds an in-memory hash table.

   For each row in `table_b`, it stores:

   ```
   hash(id) → { id, row pointer }
   ```

   Work done:

   ```
   100,000 rows scanned once
   ```

4. **Probe phase (done once per probe row)**

   PostgreSQL scans `table_a` row by row.

   For each row in `table_a`:

   * Compute `hash(a.id)`
   * Look up matching entries in hash table
   * Verify actual value (to handle collisions)
   * Output matching rows

   Work done:

   ```
   100,000 probe rows × O(1) hash lookup
   ```

5. **Total work**

   ```
   Build: 100,000
   Probe: 100,000
   ----------------
   Total ≈ 200,000 operations
   ```

### Why Hash Join Is Chosen Here

* Both tables are large
* Equality join (`=`)
* No need for indexes
* Avoids N × M behavior


**Why PostgreSQL builds the hash on the smaller table and probes the larger table:**

* PostgreSQL builds the hash table on the **smaller table** and probes the **larger table**.
* **Why smaller table?**

  * Total operations (build + probe) would be roughly the same either way.
  * Using fewer rows in memory is faster and safer — e.g., better to hash 100 rows than 100,000 rows.
  * Keeps **RAM usage low**, storing 100 rows in memory is much cheaper than 100,000 — avoids spilling the hash to disk, and ensures efficient execution.
* **Probing larger table against smaller hash table** minimizes memory pressure while producing the same result.

🧠 Memory Hook:

> *Hash Join = build small once, probe large many; RAM efficiency matters more than operation count*

### When Hash Join Is BAD

* Hash table does not fit in memory
* Hash spills to disk → slow
* Non-equality joins (`>`, `<`, `BETWEEN`)

🧠 **Memory Hook**

> Hash Join = build once (100k), probe many (100k)

---

## Merge Join (Both sides sorted on join key or ordered output)

Query:

```sql
SELECT *
FROM table_a a
JOIN table_b b ON a.id = b.id
ORDER BY a.id;
```

### How Merge Join Works

1. **Planner requires sorted inputs**
   Merge Join works only if **both sides are sorted on join key**.

2. **Prepare sorted inputs**

   * `table_a`

     * Has index on `a.id`
     * Rows already sorted → **no sort needed**

   * `table_b`

     * No index on `b.id`
     * PostgreSQL adds a **Sort node**

   Work done:

   ```
   table_b → 100,000 rows sorted
   ```

3. **Merge phase (single pass)**

   PostgreSQL walks both sorted inputs **at the same time**:

   ```
   pointer_a → table_a
   pointer_b → table_b
   ```

   Logic:

   * If `a.id == b.id` → output row
   * If `a.id < b.id` → advance pointer_a
   * If `a.id > b.id` → advance pointer_b

4. **Total work**

   ```
   Sort table_b: 100,000 log N
   Merge walk: 100,000 + 100,000
   ```

### Why Merge Join Is Chosen

* Join condition is equality
* Sorted output required (`ORDER BY`)
* One side already sorted via index
* Efficient streaming, no random access

### When Merge Join Is BAD

* Both sides unsorted → expensive sorting
* No `ORDER BY`, Hash Join usually cheaper
* Small tables → Nested Loop faster
* Sorting cost dominates if table has no index (like `table_b` in this case)


🧠 **Memory Hook**

> Merge Join = sort once, walk both lists once

## Side-by-Side Comparison (Numbers)

| Join Type   | Core Cost Pattern          |
| ----------- | -------------------------- |
| Nested Loop | outer × inner lookups      |
| Hash Join   | build once + probe once    |
| Merge Join  | sort + single linear merge |


### Final Intuition 

* **Nested Loop** → “Do I have very few outer rows AND fast inner lookup?”
* **Hash Join** → “Do I have large tables with equality join?”
* **Merge Join** → “Are my inputs already sorted or do I need ordered output?”

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

* **Nested Loop** → small outer table + indexed inner table, works best when the outer table has few rows (after filtering) and the inner table has an index. Since the outer table has few rows, the executor only performs a small number of iterations. Each outer row looks up matching rows in the inner table using the index, which is very fast. Fewer outer rows → fewer lookups → efficient execution.
Perfect! Here’s the detailed rephrasing for both, in the same style as your Nested Loop one:

* **Hash Join** → large equality joins, works best when both tables are large and the join condition is an equality (like `a.id = b.id`). The executor builds a hash table on the smaller table and then probes it with rows from the larger table. This avoids repeated scans, so even with many rows, matches are found efficiently. Large tables → build once, probe many → fast equality matching.

* **Merge Join** → sorted inputs or ordered output, works best when one or both tables are already sorted on the join key or when an ordered result is required. The executor walks through both sorted lists simultaneously, comparing join keys and outputting matches. Indexed or pre-sorted table → no additional sorting needed → fast merge. Sorted tables → sequential walk → minimal random I/O → efficient execution.

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
