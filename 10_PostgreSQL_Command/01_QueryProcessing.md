# PostgreSQL Query Processing: Deep Dive

This README explains the detailed flow of query processing in PostgreSQL, including the roles of each component in the query processor, how query plans are generated and executed, and the cost-based optimization mechanism used by the planner.

---

## 1. Overview of Query Processing

When a client sends a SQL query to PostgreSQL, the backend process handles it through several stages:

1. **Parser** – Checks syntax and generates a parse tree (PASSTREE).
2. **Analyzer** – Performs semantic analysis and transforms the parse tree into a query tree.
3. **Rewriter** – Applies rules (e.g., for views) to transform the query tree into a form referencing physical tables.
4. **Planner / Optimizer** – Generates multiple query execution plans and selects the optimal one using a **cost-based optimizer**.
5. **Executor** – Executes the instructions in the plan tree and returns the result set to the client.

**Memory Hook:** “Think of PostgreSQL query processing as an assembly line: each stage refines the query until the final result emerges.”

---

## 2. Stage-by-Stage Details

### 2.1 Parser

* **Responsibility:** Syntax analysis of the SQL query.
* **Output:** Parse Tree (PASSTREE) – a tree-like in-memory representation of the query.
* **Notes:**

  * Does **not** check semantic validity (e.g., whether tables/columns exist).
  * If syntax is invalid, an error is returned immediately.
* **Memory Hook:** “Parser is the grammar teacher—it only cares if your query speaks SQL correctly.”
<img width="1081" height="592" alt="image" src="https://github.com/user-attachments/assets/2d0e1308-e1de-4bd6-9715-9a59c852ee58" />

---

### 2.2 Analyzer

* **Actions:**

  * Checks the database catalog for referenced objects (tables, columns).
  * Assigns **object IDs** for valid references.
  * Returns errors if the query references missing objects.(It make sure that table and column exists in the database and they are valid)
* **Memory Hook:** “Analyzer is the fact-checker—ensures the query refers to real database objects.”
<img width="1092" height="515" alt="image" src="https://github.com/user-attachments/assets/e5193b87-6da8-4fd3-89c4-fe063ff6f262" />

---

### 2.3 Rewriter

Got it! Here’s the revised **Rewriter section** with your requested clarifications:

---

## PostgreSQL Query Rewriter

The **Rewriter** component in PostgreSQL is part of the **Query Rewriting System**. Its main responsibility is to **transform queries that reference virtual objects (views) or rules** into queries that operate on the underlying physical tables, applying all predefined rules from the catalog.

### How It Works

1. **Views (Virtual Objects):**

   * When a query references a **view** (a virtual table), the Rewriter expands the view so that the query references the underlying physical tables instead.
   * This ensures the Planner and Executor see a query that can actually access real data.

2. **Rules:**

   * Rules allow queries to be **automatically modified or extended** before execution.

   * Example:

     ```sql
     -- Rule: Any insert into employee should also insert into audit_table
     CREATE RULE employee_insert AS
     ON INSERT TO employee
     DO ALSO INSERT INTO audit_table(user_id, action) VALUES (NEW.id, 'INSERT');
     ```

     * Original Query:

       ```sql
       INSERT INTO employee(id, name) VALUES (1, 'John Doe');
       ```
     * Rewritten by Rewriter:

       ```sql
       INSERT INTO employee(id, name) VALUES (1, 'John Doe');
       INSERT INTO audit_table(user_id, action) VALUES (1, 'INSERT');
       ```

   * The Rewriter applies this **at query rewrite time**, producing a **Rewritten Query Tree** that is passed to the Planner.

### Note on Rules vs Triggers

* **Rules**: Rewrite the query **before execution**; the Planner sees the modified query. They operate at the **query level**.
* **Triggers**: Fire **during execution** (row or statement level) and run procedural code. Triggers react to events; rules rewrite queries.

**Memory Hook:**

* “The Rewriter is the translator and rule enforcer—it converts views into physical table queries and applies catalog-defined rules before execution.”
* “Rules rewrite the query; triggers react to events.”

---

### 2.4 Planner / Optimizer

* **Responsibility:** Generate multiple execution plans and select the most efficient using a **cost-based optimizer**.
* **Query Paths:** Different execution strategies for the same query (e.g., sequential scan vs index scan).
* **Plan Tree:**

  * Nodes represent operations (scan, join, sort, filter).
  * Execution starts at **leaf nodes** and moves up to the **root node**.

#### Cost-Based Optimization

* Each operation is assigned a **cost**:

  * Default sequential page cost = 1.0
  * Cost per row processed = 0.01 (example)
* Uses database statistics (e.g., table size, index presence) to estimate the total cost for each plan.
* Selects the plan with the **lowest estimated cost**.

**Memory Hook:** “Planner is the strategist—calculates the cheapest path to query victory.”

---

### 2.5 Executor

* **Responsibility:** Executes instructions in the plan tree.
* **Flow:** From leaf nodes → intermediate nodes → root node.
* **Output:** Result set sent to the client.
* **Memory Hook:** “Executor is the worker—follows the plan to deliver results.”

---

## 3. Example Query Plans

### 3.1 Simple Select with Sort

```sql
SELECT name FROM TableA ORDER BY id;
```

* Leaf Node → Sequential scan of TableA
* Next Node → Sort by `id`
* Root Node → Return results

### 3.2 Join Query

```sql
SELECT * FROM TableA JOIN TableB ON TableA.id = TableB.id;
```

* Leaf Nodes → Scan TableA and TableB
* Join Node → Merge or nested loop join
* Root Node → Return joined results

### 3.3 Join with View

* View expands into underlying tables
* Filter Node → Apply view conditions
* Join Node → Combine data
* Root Node → Return results

---

## 4. Summary of Key Tasks by Component

| Component | Responsibilities                                                             |
| --------- | ---------------------------------------------------------------------------- |
| Parser    | Syntax validation, generate parse tree                                       |
| Analyzer  | Semantic validation, transform parse tree → query tree                       |
| Rewriter  | Apply rules (views) to generate rewritten query tree                         |
| Planner   | Generate multiple plan trees, select optimal plan using cost-based optimizer |
| Executor  | Execute plan instructions from leaf → root, return result set                |

---

## 5. Memory Hooks

* **Parser:** “Grammar teacher” – checks syntax only.
* **Analyzer:** “Fact-checker” – validates real database objects.
* **Rewriter:** “Translator” – converts virtual queries to physical tables.
* **Planner:** “Strategist” – picks the cheapest execution path.
* **Executor:** “Worker” – carries out the plan.

---

## 6. FAQ

**Q1: Can a query have multiple plans?**

* Yes, PostgreSQL generates multiple query paths; the planner selects the one with the lowest estimated cost.

**Q2: Does the planner always pick the best plan?**

* It picks the plan with the lowest estimated cost based on available statistics; accuracy depends on up-to-date stats.

**Q3: How does the planner use indexes?**

* If an index exists, the planner may choose an index scan over a sequential scan if it’s cheaper.

**Q4: What’s the difference between parse tree and query tree?**

* Parse Tree → Syntax representation (from parser)
* Query Tree → Semantic and object-aware tree (from analyzer)

**Q5: Why is cost-based optimization important?**

* Efficient plan reduces execution time, resource usage, and improves overall database performance.

---
