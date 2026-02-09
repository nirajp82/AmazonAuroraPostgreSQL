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
**A:** Aurora Capacity Unit – 1 ACU ≈ 2 GB memory + CPU + network. Used to define compute capacity.

**Q2: Can serverless instances scale automatically?**
**A:** Yes, Aurora v2 instances monitor load and scale up or down automatically.

**Q3: What happens if min ACU = max ACU?**
**A:** No scaling occurs; the instance behaves like a static, provisioned instance.

**Q4: Can a cluster have both provisioned and serverless instances?**
**A:** Yes, Aurora supports mixed clusters with independent scaling of serverless instances.

**Q5: How is cost calculated for serverless instances?**
**A:** Compute cost is billed **per second** based on ACU usage. Storage and I/O costs are separate.

**Q6: Is scaling instantaneous?**
**A:** Scaling up is nearly instant; scaling down is slower and depends on load and cluster configuration.
