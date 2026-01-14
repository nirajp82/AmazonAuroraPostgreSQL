# PostgreSQL Query Performance Bottlenecks and Wait Events

---

## Why Query Performance Bottlenecks Matter

A poorly performing query:

* Slows down the user-facing application
* Consumes shared resources (CPU, memory, IO)
* Negatively impacts *other* concurrent queries

On a busy database server, **multiple backend processes execute queries in parallel**, all competing for the same shared resources. PostgreSQL serializes access to critical resources to maintain **data consistency and isolation**, which introduces waits.

Understanding *where* a query is waiting is the key to fixing performance issues.

---

## PostgreSQL Backend Process Overview

1. Client connects to PostgreSQL
2. PostgreSQL **forks a backend process** for that connection
3. Client sends SQL query
4. Backend process executes the query
5. Backend process reports execution state and waits to the **stats subsystem**

If activity tracking is enabled, PostgreSQL continuously reports this information to the **pg_stat_activity** view.

---

## pg_stat_activity View

`pg_stat_activity` provides **real-time visibility** into:

* Client sessions
* Query state
* Current SQL
* Wait event type and specific wait event

This view is the foundation for query-level performance analysis.

---

## Backend Process States

Each backend process can be in **one of the following states**:

### 1. Disabled

* Activity tracking is disabled
* Controlled by `track_activities = off`

### 2. Active

* Backend process is **executing a query**
* Primary focus for performance optimization

### 3. Idle

* Backend is waiting for a new query from the client

### 4. Idle in Transaction

* Client issued `BEGIN`
* Backend is waiting for the next statement
* **Dangerous state** if held too long (can block vacuum)

### 5. Idle in Transaction (Aborted)

* One statement failed inside a transaction
* Backend is waiting for `ROLLBACK`

### 6. Fastpath Function Call (Obsolete)

* Legacy mechanism
* Not relevant for modern PostgreSQL usage

---

## Optimization Goal

The primary goal of query optimization is:

> **Minimize the time spent by queries in the `active` state**

This is achieved by identifying *why* a query is waiting during execution.

---

## Shared Resource Contention

On a database server:

* Multiple backend processes run concurrently
* All processes compete for **shared resources**

### Types of Shared Resources

#### Server Resources (OS-managed)

* CPU
* Memory
* Disk IO
* Storage

#### Database Resources (DB-managed)

* Tables
* Indexes
* Tuples (rows)
* Shared buffers
* Internal queues and IPC structures

PostgreSQL **serializes access** to many of these resources using locks.
<img width="1268" height="595" alt="image" src="https://github.com/user-attachments/assets/847a51eb-4e10-4cf2-9eea-bcf68138b870" />

---

## Example: Serialized Access to a Table

* Two backend processes issue `UPDATE` on the same table
* PostgreSQL allows **only one process** to proceed
* Other process waits in a **lock queue**
* Once the first transaction commits, the lock is released
* Waiting process resumes execution
      
This waiting time directly impacts query latency.
<img width="1261" height="563" alt="image" src="https://github.com/user-attachments/assets/3f38f2e8-f8fa-46bf-be1a-159bcb0624de" />

---

## Lock Queues and Waits

Below is a **clean, README-ready section** you can copy-paste directly.
I’ve corrected wording, clarified concepts, and kept the meaning **exactly aligned** with PostgreSQL behavior.

---

## Serialization of Access to Shared Resources

The key takeaway is that **database engines serialize access to shared resources**.

At any point in time, **only one backend process can actively use a shared resource**.
If another backend process needs the same resource, it **must wait** until the resource becomes available.

This waiting backend process enters a **waiting state**, which is represented internally as a **wait event**.
PostgreSQL controls access to shared resources using a **locking mechanism**.

* The backend process that **holds the lock** is allowed to operate on the resource
* Other backend processes requesting the same resource are **blocked**
* Blocked processes are placed into a **lock wait queue**

On a busy server, it is common to see:

* Multiple backend processes waiting for locks
* Long lock queues for heavily contended resources

Because of this, **query performance and overall throughput** depend on more than just physical resources such as:

* CPU
* Memory
* Disk I/O

They also depend heavily on:

* **How long backend processes wait to acquire locks**
* **Which database and non-database resources are contended**
* **How efficiently locks are acquired and released**

Lock contention can occur on:

* Database objects (tables, rows, indexes)
* Internal shared memory structures
* Inter-process communication (IPC) resources

---

## Implication for Query Performance

When analyzing query performance issues, it is not sufficient to look only at hardware utilization.

A query may be slow because:

* It is waiting in a **lock queue**
* Another backend process is holding a required lock for a long time
* Multiple queries are competing for the same shared resource

Understanding **lock waits and wait events** is critical to identifying query processing bottlenecks.

---

## Transition to Bottleneck Analysis

With this foundation, we can now examine **potential bottlenecks in query processing**, starting from:

* Client connection to the PostgreSQL server
* Backend process creation
* Resource acquisition and lock contention during query execution

---

## Query Processing Bottlenecks

Potential bottlenecks during query execution include:

### 1. Backend Process Creation

* Depends on CPU and memory availability

### 2. Local Processing Resources

* Working memory (`work_mem`)
* Temporary file IO

### 3. Physical Resource Pressure

* High CPU utilization
* Memory starvation
* Slow disk IO

### 4. Shared Resource Locks

* Shared buffers
* IPC queues
* Database object locks

### 5. Extension Execution

* Extensions run **synchronously** inside backend process
* Slow extensions directly slow query execution
<img width="1101" height="567" alt="image" src="https://github.com/user-attachments/assets/75d1abf0-d1a4-485c-8b1b-f306ade16fdf" />

---

## Wait Events in pg_stat_activity

If a session is waiting, PostgreSQL reports:

* `wait_event_type`
* `wait_event`

There are **nine wait event types**.

---

## The Nine Wait Event Types

### 1. Activity

* Backend is in the main execution loop

### 2. BufferPin

* Waiting for exclusive access to a data buffer
* High frequency indicates buffer contention

### 3. Client

* Waiting for client interaction
* Example: client not sending data

### 4. Extension

* Waiting on a condition defined by an extension

### 5. IO

* Waiting for disk IO to complete
* Indicates storage or cache inefficiency

### 6. IPC

* Waiting for inter-process communication
* Message queues, signals, shared memory

### 7. Lock (Heavyweight Lock)

* Waiting for database object locks
* Tables, rows, internal database locks

### 8. LWLock (Lightweight Lock)

* Protects in-memory structures
* Shared memory data structures

### 9. Timeout

* Waiting for a timeout to expire
* Example: `pg_sleep()`

---

## Wait Events vs Wait Event Types

* **wait_event_type**: High-level category
* **wait_event**: Specific event name

Each type includes **multiple specific events**.

Example:

* `wait_event_type = Client`
* `wait_event = ClientRead`

This indicates the backend is waiting for data from the client.

---

## Why Lock-Related Waits Are Critical

Two wait types are especially important:

* `Lock`
* `LWLock`

These waits:

* Serialize access to shared resources
* Directly impact concurrency
* Are required for consistency and isolation

Excessive lock waits are a strong indicator of **contention problems**.

---

## Memory Hook 🧠

> **Active query + long wait = contention problem**

Ask these questions:

1. Is the query active?
2. What resource is it waiting for?
3. Is the wait due to IO, lock, or memory?

The answer points directly to the fix.

---

## Best Practices

* Avoid long-running transactions
* Reduce lock contention
* Optimize indexes and query design
* Monitor `pg_stat_activity` regularly
* Investigate dominant wait events

---

## FAQ

### Q1. Why is my query active but slow?

Because it is likely **waiting on a resource** (lock, IO, buffer, or IPC).

### Q2. What is the most common performance bottleneck?

Lock contention and IO waits on busy systems.

### Q3. Are lock waits always bad?

No. Locks are necessary for consistency. **Excessive lock waits** are the problem.

### Q4. How do extensions affect performance?

Extensions execute synchronously inside backend processes, so slow extensions slow queries.

### Q5. Which view should I check first for query issues?

`pg_stat_activity`

---

## What’s Next

In the next lesson, you will:

* Observe wait events and lock queues in action
* Learn how contention impacts throughput
* Correlate waits with real performance issues
