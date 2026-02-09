# Aurora Serverless v2 DB Instance

## Overview: Aurora Serverless v2

Aurora Serverless v2 is a **production-ready, scalable relational database** that automatically adjusts compute capacity based on load. It addresses the limitations of Aurora Serverless v1 by enabling:

* **Instant scaling** of compute resources.
* **High availability** for production workloads.
* **Flexible cluster configurations** (all serverless or mixed with provisioned instances).

**Memory Hook:** Think of v2 as a **smart elevator**: it automatically goes up or down depending on how many people (load) are in the building, without you pushing buttons.

---

## Key Concepts

### 1. Separation of Compute and Storage

* **Storage:** Scales automatically based on data usage.
* **Compute:** In **provisioned instances**, capacity is static and requires manual resizing.
* **Serverless v2 instances:** Compute capacity scales automatically **vertically** based on load.

---

### 2. Capacity Specification

* Compute capacity is specified in **Aurora Capacity Units (ACUs or "Aurora units")**.
* **1 ACU ≈ 2 GB memory, equivalent CPU, and network bandwidth.**
* **Cluster-level capacity range:**

  * Minimum ACU (e.g., 0.5 ACU = ~1 GB memory)
  * Maximum ACU (up to 128 ACU = ~256 GB memory)
* **All instances in the cluster share the same capacity range**.
* If `min = max`, the instance behaves like a **provisioned instance** (no scaling).

**Memory Hook:** ACU is like a "unit of work" – think of it as a **slice of server power** that can grow or shrink automatically.

#### What 1 ACU Means

* **ACU** = Aurora Capacity Unit
* **1 ACU ≈ 2 GB memory + equivalent CPU + network bandwidth**
* **All three scale together automatically** — you do not configure them separately.

**Memory Hook:** Think of 1 ACU as a **unit of “server power”**: it bundles memory, CPU, and network together.

#### Breakdown of ACU Components

| Component             | Role                                                                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Memory (RAM)**      | Stores active data, caches query results, temporary buffers for inserts/updates, working space for aggregations and joins |
| **CPU**               | Processes queries, calculations, sorting, joins, aggregations, indexes, triggers, constraints                             |
| **Network Bandwidth** | Transfers data between client apps and DB, handles replication traffic, moves bulk data during inserts/updates            |

#### Why More CPU & Network are Needed

Memory alone cannot handle high workloads. CPU and network scale proportionally to:

1. **Bulk Inserts / Updates**

   * Example: Insert **1 million rows** into a table
   * Memory: buffers data before commit
   * CPU: validates constraints, updates indexes, executes triggers efficiently
   * Network: transfers large data streams from application or ETL pipelines

2. **Complex Queries / Aggregations**

   * Example: `SELECT SUM(amount), COUNT(*) FROM orders WHERE created_at > now() - interval '1 month';`
   * CPU: scans rows, evaluates filters, performs aggregation
   * Memory: caches temporary results
   * Network: returns large datasets to application clients

3. **Replication / Read Traffic**

   * Example: Read replicas serving thousands of queries per second
   * CPU: executes queries for each request
   * Memory: caches results to speed repeated queries
   * Network: transfers data to replicas or application servers

#### Real-World Example: Bulk Insert Scenario

Assume a **Serverless v2 instance**:

* Cluster capacity: 4 ACU → 8 GB memory
* Table: `orders` with 1 million rows

| Situation                                   | Memory | CPU      | Network | Result                                                                                   |
| ------------------------------------------- | ------ | -------- | ------- | ---------------------------------------------------------------------------------------- |
| **Low ACU / insufficient resources**        | 8 GB   | ~4 vCPUs | ~4 Gbps | Slow inserts; CPU bottleneck; network slows bulk transfer                                |
| **High ACU / sufficient resources (8 ACU)** | 16 GB  | ~8 vCPUs | ~8 Gbps | Inserts execute faster; CPU and network scale with memory; bulk data handled efficiently |

**Memory Hook:** Memory buffers data, CPU crunches it, and network carries it. If any of the three is too low, the workload slows down.

#### Example: Serverless Instance Scaling

Cluster capacity range: 1–4 ACUs

| Load Situation  | ACU Allocated | Memory | CPU      | Network |
| --------------- | ------------- | ------ | -------- | ------- |
| Initial load    | 1 ACU         | 2 GB   | ~1 vCPU  | ~1 Gbps |
| High load spike | 3 ACU         | 6 GB   | ~3 vCPUs | ~3 Gbps |
| Load subsides   | 1 ACU         | 2 GB   | ~1 vCPU  | ~1 Gbps |

* **Scaling up:** almost instantaneous
* **Scaling down:** slower, depends on load and cluster size
* Each instance in a cluster **scales independently**

**Memory Hook:** ACU acts like a **combo pack**: more ACUs → more memory, CPU, and network proportionally.

#### Summary Table: ACU vs Resources vs Use Case

| ACU     | Memory | CPU       | Network  | Use Case Example                                        |
| ------- | ------ | --------- | -------- | ------------------------------------------------------- |
| 1 ACU   | 2 GB   | ~1 vCPU   | ~1 Gbps  | Small queries, occasional inserts                       |
| 4 ACU   | 8 GB   | ~4 vCPUs  | ~4 Gbps  | Medium workload, bulk insert 100k rows                  |
| 8 ACU   | 16 GB  | ~8 vCPUs  | ~8 Gbps  | Large workload, bulk insert 1M+ rows, heavy analytics   |
| 128 ACU | 256 GB | 128 vCPUs | 128 Gbps | Enterprise production, very large analytics / batch ETL |

#### Key Takeaways

1. **ACU bundles memory, CPU, and network**; they scale together.
2. **Higher ACU → faster processing, more parallelism, and faster data transfer.**
3. **Serverless v2 instances scale automatically** based on load.
4. CPU, memory, and network must scale together; otherwise, one becomes a bottleneck.
5. Serverless v2 instances are **drop-in replacements** for provisioned instances.
---

### 3. Scaling Behavior

* Aurora **monitors load** on each serverless instance and adjusts capacity dynamically in increments of 0.5 ACU or more.
* **Scaling up:** almost instantaneous.
* **Scaling down:** slower, depends on workload and cluster capacity range.
* **Independent scaling:** each serverless instance scales independently, even in multi-instance clusters.

**Example:**

* Cluster capacity range: 1–4 ACUs
* Initial load: 1 ACU
* Load spikes → scales to 3 ACUs
* Load subsides → scales back to 1 ACU

**Memory Hook:** Think of it like a **rubber band**: stretches quickly when needed, slowly returns to original size.

---

### 4. Cluster Configurations

**All Serverless Cluster:**

* All instances are serverless.
* Each instance scales independently.

**Mixed Cluster:**

1. **Provisioned writer + serverless readers**

   * Writer is static
   * Readers scale based on read load
2. **Serverless writer + provisioned readers**

   * Writer scales dynamically
   * Reader capacity remains constant

---

### 5. Estimating CPU & Network

* CPU and network bandwidth can be approximated using **equivalent provisioned instance classes (R5/R6 series)**.
* **Example:**

  * 3 ACU → ~64 GB memory → equivalent to `db.r5.2xlarge`
  * 128 ACU → ~256 GB memory → equivalent to `db.r5.8xlarge`

---

### 6. Cost Considerations

* **Cost is based on ACU usage per second**.
* **Savings come only from compute**, storage and I/O costs remain unchanged.
* **Per-second billing:**

  * Track ACU usage over time intervals.
  * Multiply ACU × seconds × cost per ACU per second.

**Example:**

* Cluster range: 4–8 ACU
* First 10 seconds: load = 4 ACU → cost = 4 × 10 × ACU/sec
* Next 20 seconds: load = 6 ACU → cost = 6 × 20 × ACU/sec
* Total cost = sum of all intervals
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/e1911dc4-dba9-42b9-ab0a-9bf0526a68a1" />

**Memory Hook:** Think of it like a **pay-as-you-go taxi**: pay only for the power (ACU) used per second.

---

## Key Takeaways

1. Aurora Serverless v2 dynamically adjusts capacity for serverless instances.
2. Specify capacity range (min and max ACUs) at cluster level.
3. Scaling rate varies: **fast up, slower down**.
4. Serverless v2 instances are **drop-in replacements** for provisioned instances.
5. CPU and network bandwidth are roughly estimated using equivalent R5/R6 instance types.
6. Costs are **compute-based per second**, storage/I/O costs are unchanged.

---

## FAQ

**Q1: What is an ACU?**
A: Aurora Capacity Unit – 1 ACU ≈ 2 GB memory + CPU + network bandwidth. It defines the compute capacity of serverless instances.

**Q2: Can serverless instances scale automatically?**
A: Yes, Aurora v2 instances monitor load and scale up or down automatically based on demand.

**Q3: Why does CPU scale with ACU?**
A: CPU handles query execution, indexing, triggers, aggregations, and other computations. Without sufficient CPU, heavy workloads such as bulk inserts or complex queries slow down.

**Q4: Why does network scale with ACU?**
A: Network bandwidth is required to transfer data between clients and the database, as well as between replicas. High-volume inserts, updates, and queries need more bandwidth for efficient processing.

**Q5: What happens if min ACU = max ACU?**
A: No scaling occurs. The instance behaves like a static provisioned instance with fixed capacity.

**Q6: Can a cluster have both provisioned and serverless instances?**
A: Yes. Aurora supports mixed clusters. Serverless instances scale independently, while provisioned instances remain static.

**Q7: How is cost calculated for serverless instances?**
A: Compute cost is billed per second based on actual ACU usage. Storage and I/O costs are separate.

**Q8: Is scaling instantaneous?**
A: Scaling up is nearly instantaneous, while scaling down is slower and depends on load, instance size, and cluster configuration.
