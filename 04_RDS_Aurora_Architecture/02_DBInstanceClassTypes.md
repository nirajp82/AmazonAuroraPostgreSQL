# Amazon Aurora Database Instance Types – Overview

In this lesson, we cover **Aurora database instance types**, including **burstable vs memory-optimized instances**, **instance classes and sizes**, **network and storage bandwidth**, and **how to select the appropriate instance** for your workload.

---

## 1. Burstable vs Memory-Optimized Instances

Aurora provides multiple types of DB instances:

* **Burstable Performance (B-class):**

  * Suitable for **workloads with variable CPU usage**.
  * Accumulates **CPU credits** during low utilization periods.
  * Credits are spent when CPU spikes above average usage.
  * **Cost-efficient** for workloads with intermittent traffic.
* **Memory-Optimized (R/X-class):**

  * Designed for **memory-intensive, high-performance workloads**.
  * Ideal for analytics, large tables, and high concurrency queries.
  * Provides **consistent CPU and memory performance**.
 
<img width="1066" height="431" alt="image" src="https://github.com/user-attachments/assets/bdc8fa7c-07bc-475b-ae05-1022dc58a26c" />

**Memory Hook:**

> “B-class saves cost with CPU credits; R/X-class gives memory and power.”

---

## 2. Database Instance Classes and Sizes

<img width="1057" height="514" alt="image" src="https://github.com/user-attachments/assets/d2cb7279-b718-4475-a56d-0a568bb38dda" />

* **DB Instance Classes** define:

  * **Number of vCPUs**
  * **Memory size**
  * **Maximum network bandwidth**

* **Naming convention:**

  * `db.r5` or `db.x1` → Memory-optimized
  * `db.t3` → Burstable performance
  * Suffix `g` → **Graviton processor** (better performance/cost)

* **Instance sizes:**

  * e.g., `r5.8xlarge` = 32 vCPUs, 256 GB RAM, 10 Gbps network
  * e.g., `r5.16xlarge` = 64 vCPUs, 512 GB RAM, 20 Gbps network

**Memory Hook:**

> “r5/x1 = power; t3 = burst; g = Graviton”

---

## 3. Local Storage for Instances

* Aurora DB instances **require local storage** for:

  * Logs
  * Query processing
  * Temporary database operations
* **EBS volume** is automatically attached based on instance class.
* Users **cannot resize EBS** attached to an instance.
* Running out of local storage → potential query failures.

**Best Practice:**

> Monitor local storage usage and consider **vertical scaling** if memory or CPU pressure occurs.

---

## 4. Instance I/O Performance

* I/O depends on **instance network bandwidth**:

  * **Local storage I/O** → uses EBS bandwidth
  * **Aurora storage I/O** → uses network to Aurora cluster volume
* Example:

| Instance Class | vCPUs | RAM    | Max Local Storage IOPS | Network Bandwidth |
| -------------- | ----- | ------ | ---------------------- | ----------------- |
| r5.8xlarge     | 32    | 256 GB | 6,800 MB/s             | 10 Gbps           |
| r5.16xlarge    | 64    | 512 GB | 13,600 MB/s            | 20 Gbps           |

**Memory Hook:**

> “Double the instance → double the CPU & RAM → double the IOPS & bandwidth”

---

## 5. How to Select a DB Instance Type

* **Workload Characteristics**:

  * **Mostly steady CPU:** B-class is cost-efficient
  * **High concurrency or large memory:** R/X-class recommended
* **Database Size**:

  * Larger databases → more memory → better caching → fewer I/O bottlenecks
* **Query Characteristics**:

  * Complex joins, filters, and analytics → higher CPU & memory
* **CPU spikes**:

  * Burstable instances can handle temporary spikes
  * Sustained high CPU → upgrade instance class
* **Instance Availability**:

  * Check **Aurora documentation** for **PostgreSQL version support** per instance class and region.

**Memory Hook:**

> “Match instance type to CPU, memory, network needs & PostgreSQL version support”

---

## 6. Key Takeaways

* Instance type determines:

  * **vCPU count**
  * **Memory**
  * **Local storage size**
  * **Network bandwidth**
  * **Aurora storage I/O bandwidth**
* Burstable instances = **low cost, variable workloads**
* Memory-optimized instances = **high performance, memory-intensive workloads**
* **Graviton instances** = cost-efficient, high performance
* Vertical scaling may be required if workload or storage demands grow

**Memory Cheat Sheet:**

| Feature                  | Key Detail                                           |
| ------------------------ | ---------------------------------------------------- |
| Burstable (t3/t4)        | CPU credits, low cost, occasional spikes             |
| Memory-Optimized (r5/x1) | High memory, consistent CPU, high concurrency        |
| Graviton (g)             | AWS Graviton processor, better price/performance     |
| Local storage            | EBS attached, fixed size by instance class           |
| I/O performance          | Depends on network and instance class                |
| Instance selection       | Based on workload, query type, DB size, CPU & memory |
