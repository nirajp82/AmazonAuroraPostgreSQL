# Amazon Aurora – Primary, Replicas, and Endpoints

This lesson explains the **roles of primary and standby instances** in an Amazon Aurora database cluster and the **different types of endpoints** used by applications.

It includes:

* Primary (writer) and replica (reader) instances
* Aurora replication behavior
* Cluster, reader, instance, and custom endpoints
* Failover and DNS behavior
* Best practices for application connectivity
* Q&A, diagrams, and a memory cheat sheet

---

## 1. Aurora Database Cluster Overview

* An **Aurora database cluster** can contain **multiple database instances**
* The **first instance** added becomes the:

  * **Primary instance**
  * Also called **master** or **writer**
* All additional instances are added as:

  * **Hot standbys**
  * Also called **read replicas**, **reader instances**, or **replicas**

### Important Facts

* Aurora PostgreSQL supports **single-writer clusters only**
* Aurora **automatically manages replication** between primary and replicas
* Replication setup requires **no manual configuration**

---

## 2. Primary (Writer) vs Replica (Reader) Instances

### Primary Instance (Writer)

* Can process:

  * `SELECT`
  * `INSERT`
  * `UPDATE`
  * `DELETE`
* **Only instance that accepts write traffic**
* Also known as:

  * Primary
  * Master
  * Writer

### Replica Instances (Readers)

* Can process **only read queries** (`SELECT`)
* Write queries **fail** on replicas
* Used for:

  * Read scaling
  * Offloading traffic from primary

### Limits & Best Practices

* Up to **15 replicas per cluster**
* Replicas must be in the **same AWS region**
* Spread replicas across **multiple Availability Zones**
* Use **same or larger instance class** for replicas

---

## 3. Aurora Endpoints – Overview

Aurora provides multiple endpoint types for application connectivity:

1. **Cluster Endpoint (Writer Endpoint)** – automatic
2. **Reader Endpoint** – automatic
3. **Instance Endpoint** – automatic
4. **Custom Endpoint** – user-defined

---

## 4. Cluster Endpoint (Writer Endpoint)

* A **DNS CNAME** created automatically by Aurora
* Always points to the **current primary instance**
* Used for **all write traffic**

### Failover Behavior

* If the primary fails:

  * A replica is **promoted to primary**
  * DNS CNAME is **updated automatically**
  * Applications reconnect without code changes

### Best Practice

> Always use the **cluster (writer) endpoint** for write operations.

---

## 5. Reader Endpoint

* Automatically created by Aurora
* **Load-balances read traffic** across replicas
* Used for scaling read-only workloads

### Single-Instance Cluster Behavior

* If only one instance exists:

  * Both cluster and reader endpoints point to the primary

### Critical Best Practice

> Always use reader and writer endpoints, even for single-instance clusters.
> This makes the application **future-proof**.

---

## 6. Instance Endpoints

* One **instance endpoint per DB instance**
* Points to a **specific instance**
* Does **not change during failover**

### Use Cases

* Troubleshooting instance-level issues
* Applications needing **direct load control**
* Faster recovery (avoids DNS propagation delay)

---

## 7. Replica Deletion & Grace Period

* When a replica is removed:

  * Instance endpoint is removed immediately
  * Removed from reader endpoint
  * **3-minute grace period** for running queries
* Long-running queries beyond 3 minutes **will fail**

---

## 8. Custom Endpoints

Custom endpoints provide **predictable and controlled load balancing**.

### Characteristics

* User-defined
* Can include:

  * Readers only
  * Writers and readers
* Grouped by:

  * Instance type
  * Parameter group
  * Custom selection

### Limits

* Max **5 custom endpoints per cluster**
* Max endpoint name length: **63 characters**

### Endpoint Naming Format

```
<name>.cluster-custom-<account-id>.<region>.rds.amazonaws.com
```

> Endpoint does **not include cluster name**, allowing reassignment to another cluster.

---

## 9. Reader Endpoint vs Custom Endpoint

| Feature                 | Reader Endpoint | Custom Endpoint |
| ----------------------- | --------------- | --------------- |
| Created automatically   | Yes             | No              |
| Load balanced           | Yes             | Yes             |
| Predictable performance | No              | Yes             |
| Workload isolation      | No              | Yes             |

> If custom endpoints are used, reader endpoint is typically not used.

---

## 10. Custom Endpoint – Real-World Example

### Scenario

* One Aurora cluster
* Two applications:

  * Web application
  * Reporting/analytics application

### Problem

* Reporting queries are heavy
* Both apps share reader endpoint
* Web app performance degrades

### Solution

* Create two custom endpoints:

  * **Web endpoint** → smaller instances
  * **Reporting endpoint** → larger instances (e.g., `db.r5.16xlarge`)

### Result

* Isolated workloads
* Predictable query performance

---

## 11. ASCII Diagrams

### Endpoint Routing

```
Application
   |
   |-- Writer (Cluster Endpoint) ---> Primary Instance
   |
   |-- Reader Endpoint ------------> Replica 1
   |                                Replica 2
   |                                Replica 3
```

### Failover Flow

```
Primary Fails
     |
Replica Promoted
     |
DNS Updated
     |
Application Reconnects
```

---

## 12. Q&A – Exam & Interview Friendly

**Q: Which instance accepts writes?**
A: Only the primary (writer) instance.

**Q: What endpoint should be used for writes?**
A: Cluster (writer) endpoint.

**Q: How many replicas can Aurora have?**
A: Up to 15 per cluster.

**Q: When should instance endpoints be used?**
A: Troubleshooting or fast recovery scenarios.

**Q: Why use custom endpoints?**
A: To isolate workloads and achieve predictable performance.

---

## 13. Memory Cheat Sheet

| Concept           | Remember This            |
| ----------------- | ------------------------ |
| Primary           | Reads + Writes           |
| Replica           | Reads only               |
| Cluster Endpoint  | Always points to primary |
| Reader Endpoint   | Load balances reads      |
| Instance Endpoint | Direct instance access   |
| Custom Endpoint   | Predictable performance  |
| Replicas          | Up to 15                 |
| Grace period      | 3 minutes                |

---

## 14. Final Takeaways

* Aurora is **single-writer, multi-reader**
* Use endpoints correctly to avoid code changes later
* Custom endpoints = performance control
* Instance endpoints = troubleshooting & fast recovery
* Right endpoint strategy = stable, scalable architecture
