#  Aurora Serverless v2 – Factors Influencing Capacity & Auto Scaling

This lesson explains how **serverless v2 instance capacity** is determined, what factors influence scaling, and how **auto scaling** works in **serverless and mixed clusters**. By the end, you’ll understand **base vs. peak capacity**, the difference between **serverless scaling and cluster auto scaling**, and how workloads, priorities, and background processes affect capacity.

## 1. Overview

Aurora Serverless v2 continuously monitors **resource pressure** on each instance and adjusts **ACU (Aurora Capacity Unit)** usage automatically.

Factors influencing instance capacity:

1. **Write load:** Load on the primary instance affects its capacity.
2. **Read load:** Load on reader instances affects capacity based on priority.
3. **Reader priority:** Instances with priority **0 or 1** follow the writer’s capacity in lockstep.
4. **Performance Insights:** Enables monitoring, consumes CPU, memory, and network, increasing ACU usage.
5. **Background processes:** Maintenance, backups, or automated tasks use resources, affecting capacity.

---

## 2. Base and Peak Capacity in a Cluster

**Example Cluster:**

* **3 serverless instances**
* **Cluster ACU range:** 16–64 ACU

**Base level capacity:**

* Base = number of instances × minimum ACU
* 3 × 16 = **48 ACU**

**Maximum capacity:**

* Writer: 1 × 64 = **64 ACU**
* Readers: 2 × 64 = **128 ACU**

**Peak capacity across cluster:** 3 × 64 = **192 ACU**

> Memory Hook: Think of **base capacity** as the guaranteed minimum power your cluster always has, while **peak capacity** is the max “boost” available during spikes.

---

## 3. How Factors Influence Serverless v2 Capacity

### Writer Instance:

* Capacity adjusts for **write load**, **performance insight**, and **background processes**.

### Reader Instance:

* **Priority 0/1:** Follows writer capacity plus any read load.
* **Priority ≥2:** Scales independently based on read load, performance insights, and background tasks.

### Performance Insights & Background Processes:

* Enable detailed monitoring of DB activity.
* Consumes **CPU, memory, and network**, causing temporary ACU increases.
<img width="1108" height="587" alt="image" src="https://github.com/user-attachments/assets/df6b8f2a-6029-4947-bfd9-31266e56e12f" />

---

## 4. Auto Scaling vs Serverless v2 Scaling

| Feature        | Auto Scaling                          | Serverless v2 Scaling                   |
| -------------- | ------------------------------------- | --------------------------------------- |
| Type           | **Horizontal** (add/remove replicas)  | **Vertical** (adjust instance ACU)      |
| Trigger        | Auto scaling policies                 | Resource pressure on instance           |
| Speed          | Minutes                               | Seconds (near-instant)                  |
| Metrics        | Average across replicas (connections) | Real-time CPU, memory, network pressure |
| Cluster Impact | Adds new replicas to scale reads      | Adjusts individual instance ACU         |
| Use Case       | Gradual read scaling                  | Instant load spikes on writer/reader    |

**Example:**

* Cluster has 2 serverless V2 replicas.
* Auto scaling policy: add replica if average DB connections > 100.
* Serverless V2: ACU scales instantly if CPU/memory usage spikes, without waiting for new replicas.

---

## 5. Capacity Calculation Flow

1. **Identify instance role:** Writer or Reader
2. **Measure resource pressure:** CPU, memory, network
3. **Check reader priority:**

   * 0/1 → capacity follows writer
   * ≥2 → capacity scales independently
4. **Include Performance Insights & Background processes**
5. **Apply cluster ACU range limits:** Capacity will never exceed the **maximum ACU** specified

**Memory Hook:** Each instance behaves like a **mini power plant**, adjusting its power output (ACU) to match demand.

---

## 6. Mixed Cluster Behavior

* Writer or reader can be **provisioned** or **serverless**.
* Failover works the same way: the replica with **lowest failure priority** is promoted as writer.
* Serverless replicas follow writer’s ACU if priority is 0/1; others scale independently.
* Auto scaling adds/removes replicas as needed for read-heavy workloads.

**Important:** If serverless replicas have a **minimum ACU equal to provisioned writer**, cost savings may reduce drastically.

---

## 7. Practical Example

**Cluster:**

* 3 instances, ACU range 16–64

**Scenario:**

* Writer load = high → scales to 48 ACU
* Reader priority 1 → follows writer → 48 ACU
* Reader priority 2 → light read load → stays at 16 ACU
* Performance insights enabled → additional 2 ACU for monitoring tasks

**Result:** Total cluster ACU usage = 48 (writer) + 48 (reader1) + 16 (reader2) + 2 (monitoring) = 114 ACU

---

## 8. Key Points Recap

* Serverless V2 instance capacity depends on **write load, read load, priority, monitoring, background processes**.
* Base capacity = **#instances × min ACU**
* Peak capacity = **#instances × max ACU**
* **Serverless scaling** is vertical and fast; **auto scaling** is horizontal and slower.
* Mixed clusters behave predictably: failover and ACU adjustments work regardless of instance type.

---

## 9. Diagram: Serverless v2 Capacity & Auto Scaling

```
+-------------------+                +-------------------+
|  Writer (ACU)     |   ← Write load |  Reader P0/1      |
|  Scales Vertically |--------------> |  Follows Writer   |
+-------------------+                +-------------------+
       |
       | Read Load
       v
+-------------------+
| Reader P2         |
| Scales Independently|
+-------------------+

Notes: 
- Horizontal auto scaling adds/removes replicas if cluster read load is high.
- ACU limits enforced by cluster min/max settings.
```

*(Diagram can be replaced with a graphical flow showing Writer → Reader1/Reader2 → auto scaling arrows)*

---

## 10. FAQ

**Q1: What factors influence serverless v2 capacity?**
A: Write load, read load, reader priority, performance insights, and background processes.

**Q2: What is base vs peak capacity?**
A: Base = #instances × min ACU. Peak = #instances × max ACU.

**Q3: How does priority affect reader scaling?**
A: Readers with priority 0/1 follow writer capacity. Priority ≥2 scales independently.

**Q4: How does auto scaling differ from serverless v2 scaling?**
A: Auto scaling is horizontal (adds/removes replicas), slower, and uses target metrics. Serverless scaling is vertical, fast, and driven by instance resource pressure.

**Q5: Can mixed clusters failover reliably?**
A: Yes. Failover promotes the lowest priority replica to writer, and serverless ACU scales as needed.

**Q6: Does monitoring affect ACU?**
A: Yes. Performance Insights uses CPU/memory/network, which can increase ACU temporarily.

