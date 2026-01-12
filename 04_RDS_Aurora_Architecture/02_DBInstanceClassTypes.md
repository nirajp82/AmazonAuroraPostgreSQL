# Amazon Aurora – Instance Types & Storage Architecture

This README covers **Aurora database instance types** and **Aurora storage architecture**, including:

* Traditional DB storage vs Aurora
* Burstable vs memory-optimized instances
* Instance classes and sizes
* Network & storage bandwidth
* Local storage and I/O performance
* Aurora storage nodes, protection groups, and fault tolerance
* Multi-touch and multi-tenant storage
* How to select the right instance type and understand storage performance

---

## 1. Traditional Database Storage vs Aurora

* **Traditional relational databases** rely on **attached instance storage** or **networked storage** (EBS, SAN)
* Main **performance bottleneck**: I/O operations (IOPS)
* **Performance improvements** can be achieved by:

  1. **Reducing the number of I/O operations**
  2. **Increasing I/O bandwidth**
* **Aurora targets these challenges** by **re-architecting the storage layer from the ground up**, moving away from traditional attached/networked storage to a **distributed, database-aware storage system**

**Memory Hook:**

> “Traditional DB = limited by IOPS; Aurora = distributed, scalable, self-healing storage”

<img width="892" height="603" alt="Traditional vs Aurora Storage" src="https://github.com/user-attachments/assets/554e383f-7289-41df-8f0f-694a2c845d03" />

---

## 2. Burstable vs Memory-Optimized Instances

Aurora provides multiple DB instance types:

* **Burstable Performance (B-class / t types)**

  * Suitable for workloads with **variable CPU usage**
  * Accumulates **CPU credits** during low utilization
  * Spends credits during CPU spikes
  * Cost-efficient for workloads with **intermittent or unpredictable traffic**

* **Memory-Optimized (R/X-class / r & x types)**

  * Designed for **memory-intensive, high-performance workloads**
  * Ideal for analytics, large tables, and high-concurrency queries
  * Provides **consistent CPU and memory performance**

**Memory-Optimized R Family – Detailed Instance Classes:**

| Instance Class | Processor          | Notes                                                                                                                               |
| -------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| db.r8g         | Graviton4          | Up to 3x more vCPUs & memory than db.r7g; ideal for memory-intensive workloads; AWS Nitro System                                    |
| db.r7g         | Graviton3          | Memory-intensive workloads; powered by AWS Nitro System                                                                             |
| db.r7i         | Intel Xeon 4th Gen | SAP-Certified; memory-intensive workloads; AWS Nitro System                                                                         |
| db.r6g         | Graviton2          | Memory-intensive workloads; AWS Nitro System                                                                                        |
| db.r6i         | Intel Xeon 3rd Gen | SAP-Certified; memory-intensive workloads                                                                                           |
| db.r5          | Intel/AMD          | Memory-intensive applications; improved networking & EBS performance; AWS Nitro System                                              |
| db.r4          | Intel/AMD          | Supported only for Aurora MySQL 2.x & Aurora PostgreSQL 11/12; not available for I/O-Optimized cluster storage; upgrade recommended |

* **Memory-Optimized X Family – Detailed Instance Classes:**

| Instance Class | Processor | Notes                                                     |
| -------------- | --------- | --------------------------------------------------------- |
| db.x2g         | Graviton2 | Memory-intensive applications; low cost per GiB of memory |

* **Graviton Instances (suffix `g`)**

  * Uses AWS Graviton processor
  * **Better price/performance** ratio

**Memory Hook:**

> “B-class saves cost with CPU credits; R/X-class gives memory and power; `g` = Graviton”

<img width="1066" height="431" alt="Aurora Instance Types" src="https://github.com/user-attachments/assets/bdc8fa7c-07bc-475b-ae05-1022dc58a26c" />

---

## 3. Database Instance Classes and Sizes

* **DB Instance Classes** define:

  * Number of vCPUs
  * Memory size
  * Maximum network bandwidth

* **Naming convention:**

  * Always starts with `db`
  * `db.r5` or `db.x1` → Memory-optimized (r/x types)
  * `db.t3` → Burstable performance
  * Suffix `g` indicates **Graviton processor**

* **Instance sizes example:**

  * `r5.8xlarge` = 32 vCPUs, 256 GB RAM, 10 Gbps network
  * `r5.16xlarge` = 64 vCPUs, 512 GB RAM, 20 Gbps network

**Memory Hook:**

> “r5/x1 = power; t3 = burst; g = Graviton”

<img width="1057" height="514" alt="Instance Classes & Sizes" src="https://github.com/user-attachments/assets/d2cb7279-b718-4475-a56d-0a568bb38dda" />

---

## 4. Local Storage for Instances

* Aurora DB instances require **local storage** for:

  * Logs
  * Query processing
  * Temporary database operations

* **EBS volume** is attached automatically based on instance class

* **Users cannot resize EBS** volumes

* Running out of local storage → **query failures or performance issues**

**Best Practice:**

> Monitor local storage and consider **vertical scaling** if CPU or memory pressure occurs.

---

## 5. Instance I/O Performance

* **I/O depends on instance network bandwidth**:

  * **Local storage I/O** → EBS bandwidth
  * **Aurora storage I/O** → network to Aurora cluster volume

* Example:

| Instance Class | vCPUs | RAM    | Max Local Storage IOPS | Network Bandwidth |
| -------------- | ----- | ------ | ---------------------- | ----------------- |
| r5.8xlarge     | 32    | 256 GB | 6,800 MB/s             | 10 Gbps           |
| r5.16xlarge    | 64    | 512 GB | 13,600 MB/s            | 20 Gbps           |

**Memory Hook:**

> “Double the instance → double the CPU & RAM → double the IOPS & bandwidth”

---

## 6. How to Select a DB Instance Type

* **Workload Characteristics**:

  * Mostly steady CPU → B-class is cost-efficient
  * High concurrency / large memory → R/X-class recommended

* **Database Size**:

  * Larger DB → more memory → better caching → fewer I/O bottlenecks

* **Query Characteristics**:

  * Complex joins, filters, analytics → higher CPU & memory required

* **CPU Spikes**:

  * Burstable instances handle temporary spikes
  * Sustained high CPU → upgrade instance class

* **Instance Availability**:

  * Check **Aurora documentation** for supported **PostgreSQL versions** per instance class and region

**Memory Hook:**

> “Match instance type to CPU, memory, network, PostgreSQL version & storage needs”

---

## 8. Key Takeaways

* **Instance type** determines vCPU, memory, local storage, network & Aurora storage I/O bandwidth
* **Burstable instances** = low cost, variable workloads
* **Memory-optimized instances** = high performance, memory-intensive workloads
* **Graviton instances** = cost-efficient, high performance
* **Local storage** = fixed-size EBS attached, monitor usage
* **Aurora storage** = distributed, database-aware, self-healing, multi-tenant, multi-touch
* **Vertical scaling** may be needed for large workloads or storage pressure

| Feature                  | Key Detail                                                           |
| ------------------------ | -------------------------------------------------------------------- |
| Traditional storage      | Attached storage or SAN/EBS, IOPS bottleneck                         |
| Burstable (t3/t4)        | CPU credits, low cost, occasional spikes                             |
| Memory-Optimized (r5/x1) | High memory, consistent CPU, high concurrency                        |
| Graviton (g)             | AWS Graviton processor, better price/performance                     |
| Local storage            | EBS attached, fixed size by instance class                           |
| I/O performance          | Depends on network and instance class                                |
| Aurora storage           | Distributed, database-aware, self-healing, multi-tenant, multi-touch |
| Instance selection       | Based on workload, query type, DB size, CPU & memory                 |

---
