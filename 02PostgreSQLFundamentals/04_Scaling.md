# PostgreSQL Scalability 

## 1. What is Scalability?

**Scalability** refers to the ability of a database to handle increased load by adding resources.

Two main approaches:

1. **Vertical Scaling (Scaling Up)**

   * Increase resources of a single database instance: CPU, memory, storage, or network bandwidth.
   * Simple to implement, but limited by hardware capacity.

2. **Horizontal Scaling (Scaling Out)**

   * Add additional database instances to distribute load.
   * Load is shared across multiple servers in a **database cluster**.
   * Focus of this lesson is on **horizontal scaling**.
<img width="760" height="340" alt="image" src="https://github.com/user-attachments/assets/ec27f0d2-0db4-4c0c-be54-49ccfe8af6f7" />

https://www.youtube.com/watch?v=dvRFHG2-uYs 

---

## 2. Horizontal Scaling for Read-Heavy Workloads

Many databases are **read-heavy**, meaning **SELECT statements far outnumber INSERT/UPDATE/DELETE statements**.

**Example:**

* An e-commerce system where the web application writes orders to the database.
* Multiple reporting or analytics systems query the same database heavily.

**Problem:**

* Concurrent read and write operations on a single server can degrade performance.
* Example: Heavy reporting can delay inserts for order creation, affecting **user experience**.

**Solution: Add Hot Standbys**

* Add one or more **hot standby servers** to the cluster.
* **Writes** go to the primary instance.
* **Reads** from all applications are routed to standby servers.
* Typically combined with a **network load balancer** to evenly distribute read traffic.

**Replication Considerations:**

* **Asynchronous replication:**

  * Standby may lag behind primary.
  * Some stale data may be served.
  * Acceptable if eventual consistency is tolerable.
  <img width="1653" height="757" alt="image" src="https://github.com/user-attachments/assets/acb3a67c-a4fd-478a-8599-cb2f183569ce" />

* **Synchronous replication:**

  * Primary waits for standby confirmation for each write.
  * Ensures no data loss, but **reduces write performance**.
  <img width="1664" height="769" alt="image" src="https://github.com/user-attachments/assets/6a6659e6-3918-43d2-9766-5b5734222cc9" />

**Key Trade-off:** Performance vs. data freshness.

---

## 3. Horizontal Scaling for Write-Heavy Workloads

Scaling writes is more complex:

* All database servers in the cluster must process write operations.
* This requires mechanisms to maintain **data integrity** and **consistency** across servers.

**Challenges in PostgreSQL:**

* PostgreSQL is **not inherently a distributed database**.
* No out-of-the-box multi-master (multi-write) replication exists.

**Common Approaches for Scaling Writes:**

1. **Sharding (Partitioning)**

   * Split data into partitions across multiple PostgreSQL instances.
   * Each shard handles a subset of the data.
   * Can use **foreign data wrappers** to access data across shards.
   * Not a true multi-master solution, but effective for distributing write load.

2. **Third-Party / Commercial Solutions**

   * Products leveraging replication to achieve multi-master or multi-node clusters.
   * Examples:

     * **EDB BDR (Bi-Directional Replication)**: Multi-master cluster using asynchronous replication.
     * **Citus**: Open-source solution for distributed PostgreSQL with scaling support.
   * Not all tools support true multi-master; some provide master-slave or read-replica clusters.

---

## 4. Summary of Scaling Strategies

| Scaling Type        | Approach                           | Use Case                  | Notes                                                            |
| ------------------- | ---------------------------------- | ------------------------- | ---------------------------------------------------------------- |
| Vertical (Scale Up) | Add CPU, memory, storage, network  | General performance boost | Simple but limited by hardware                                   |
| Horizontal - Reads  | Add hot standby servers            | Read-heavy workloads      | Use load balancer; replication can be async or sync              |
| Horizontal - Writes | Sharding or multi-master solutions | Write-heavy workloads     | PostgreSQL has no native multi-master; use third-party solutions |

**Key Points:**

* For read-heavy systems, **hot standby + load balancer** is a simple horizontal scaling solution.
* **Replication type** affects data freshness and performance:

  * **Asynchronous:** Low overhead, eventual consistency.
  * **Synchronous:** No data loss, but slower writes.
* For write-heavy scaling, consider **sharding** or **commercial multi-master solutions**.
* PostgreSQL does not provide a native out-of-the-box mechanism for scaling writes across multiple servers.
