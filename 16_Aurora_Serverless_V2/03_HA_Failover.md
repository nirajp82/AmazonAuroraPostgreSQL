# Aurora Serverless v2 – High Availability & Failover

## High Availability in Aurora Serverless v2

* Aurora Serverless clusters are **highly available**, similar to provisioned clusters.
* Instances are placed across **multiple Availability Zones (AZs)**, ensuring redundancy.
* **Cluster type:** Multi-AZ Aurora DB cluster.

> **Memory Hook:** HA is like having multiple backup kitchens: if one fails, another automatically takes over.

---

## Failover Behavior

* In case of primary (writer) failure, the **replica with the lowest failure priority** is promoted as the new writer.
* **Mixed configuration clusters**: whether writer or reader is provisioned or serverless, failover works the same way.
* **Capacity considerations:** The new writer must have enough capacity to handle the current write load.

---

### Example: Mixed Cluster Failover

Cluster configuration:

* Writer: Provisioned instance with 128 GB memory
* Reader replicas: Serverless instances with cluster capacity range **8–64 ACU**
* Current load: CPU usage = 75% (~48 ACU equivalent)

**Scenario:**

* Failover occurs, reader with priority 1 is promoted to writer.
* Reader capacity = 8 ACU (insufficient for previous writer load of 48 ACU)
* **Impact:** Temporary performance degradation until load stabilizes.

> **Memory Hook:** Capacity mismatch after failover is like moving from a big kitchen to a small one — tasks slow down until adjustments happen.

---

## Reader Follows Writer Mechanism (Serverless v2)

* Aurora Serverless v2 introduces **“Reader Follows Writer”** to address failover performance:

  * Reader replicas with **priority 0 or 1** automatically **scale in lockstep** with the writer.
  * Ensures that failover target has sufficient capacity.

### Example: Lockstep Scaling

* Cluster capacity range: 2–16 ACU
* Writer load spikes → capacity scales to 12 ACU
* Reader (priority 1) capacity also scales to 12 ACU
* Other readers (priority >1) maintain their independent capacity

> **Memory Hook:** Reader replicas with priority 0/1 act like a **shadow writer**, always ready to take over without performance loss.

---

## Mixed Cluster Capacity Behavior

| Cluster Type    | Writer Type | Reader Type | Failover Behavior                                 | Notes                                                        |
| --------------- | ----------- | ----------- | ------------------------------------------------- | ------------------------------------------------------------ |
| Mixed           | Provisioned | Serverless  | Reader with priority 0/1 promoted if writer fails | Capacity may be limited if not using “Reader Follows Writer” |
| Mixed           | Serverless  | Provisioned | Writer scales automatically                       | Readers remain static unless priority = 0/1                  |
| Serverless-only | Serverless  | Serverless  | All instances scale independently                 | Full flexibility, lockstep optional for priority 0/1         |

**Key Note:**

* If serverless replicas are **set to priority 1** with a provisioned writer, minimum ACU = writer capacity.
* Cost savings from serverless instances can **decrease significantly or vanish**, because replicas reserve capacity equal to the writer.

---

## Capacity Adjustment Rules

1. **Serverless readers with priority 0/1 follow writer capacity**.
2. **Other serverless readers scale independently**, based on read load.
3. Capacity **never exceeds max ACU** defined in cluster settings.
4. Failover does **not reduce allocated ACU** below cluster minimum.

---

### Example: Adjusting Cluster Capacity

* Cluster capacity range: 2–12 ACU
* Reader (priority 1) initially at 8 ACU
* Cluster min/max modified: 2–4 ACU
* Result: Reader capacity reduces to 4 ACU (cluster max)
* Writer load exceeds 4 ACU → reader stays at 4 ACU

> **Memory Hook:** Lockstep scaling ensures critical readers are never starved, but adjusting cluster min/max can reduce allocation.

---

## Cost Considerations in Mixed Clusters

* Mixed clusters with **priority 1 replicas** may reduce cost savings:

  * Minimum ACU allocated = writer capacity
  * Serverless replicas act like provisioned instances in terms of cost
* Always evaluate the **tradeoff between HA readiness and cost savings**.

---

## Key Takeaways

1. Aurora Serverless v2 clusters provide **multi-AZ high availability**.
2. **Failover promotes the lowest-priority replica** as new writer.
3. **Reader Follows Writer** ensures priority 0/1 readers match writer capacity, preventing performance degradation.
4. Mixed clusters require careful **capacity planning** to balance performance and cost.
5. Maximum ACU limits are enforced; readers never scale beyond cluster maximum.

---

## FAQ – High Availability & Failover

**Q1: How does failover work in Aurora Serverless v2?**
A: The replica with the lowest failure priority is promoted to writer if the current writer fails.

**Q2: Do serverless readers automatically scale after failover?**
A: Yes, priority 0/1 readers follow the writer’s capacity automatically (lockstep).

**Q3: Can failover cause temporary performance degradation?**
A: Yes, if the new writer’s capacity is insufficient relative to the previous writer load.

**Q4: How does cluster min/max ACU affect failover?**
A: Readers cannot scale below the minimum or above the maximum ACU defined in the cluster.

**Q5: What is the cost impact of priority 1 serverless replicas?**
A: Minimum ACU equals writer capacity, which can reduce expected savings from serverless instances.

**Q6: Can a mixed cluster have provisioned and serverless writers/readers?**
A: Yes. Serverless instances scale independently unless priority 0/1 lockstep is used.


