# Aurora Serverless v2 – Monitoring & Metrics

## 1. Overview

All monitoring tools and metrics used for **provisioned Aurora instances** are applicable to **Serverless v2** instances.

Aurora introduces **additional Serverless v2-specific metrics**:

* **Serverless Database Capacity (ACU allocated)**
* **ACU Utilization**
* **Free Memory**
* **Temp Storage IOPS**
* **CPU Utilization**

These metrics help you:

* Understand resource usage at any point in time
* Estimate operating costs
* Proactively adjust cluster capacity to prevent performance issues

---

## 2. Serverless Database Capacity Metric

**Definition:**
Tracks the number of **ACUs allocated** to a Serverless v2 instance at a point in time.

**Key values to monitor:**

* **Minimum ACU allocated**
* **Maximum ACU allocated**
* **Average ACU allocated**

**Example:**

* Cluster ACU range: 2–16
* Writer instance: scales to 16 ACU
* Reader P1: scales to 11.5 ACU
* Reader P2/P5: remains at 2.5 ACU

> **Memory Hook:** The allocated ACU reflects the “power level” of your instance. Monitoring this metric ensures your cluster always has enough capacity for current workloads.

---

## 3. ACU Utilization Metric

**Definition:**
Percentage of ACU currently in use relative to the **maximum ACU** defined in the cluster.

**Calculation:**

[
\text{ACU Utilization (%)} = \frac{\text{Serverless DB Capacity (ACU allocated)}}{\text{Maximum ACU in Cluster}} \times 100
]

**Example:**

* Max ACU = 16
* Writer ACU = 11.5 → Utilization = 11.5 / 16 ≈ 72%
* Reader ACU = 2.5 → Utilization = 2.5 / 16 ≈ 16%

**Interpretation:**

* **Near 100%:** Consider increasing maximum ACU to handle spikes
* **Near minimum:** Consider lowering minimum ACU to reduce cost

---

## 4. CPU Utilization Metric

* Measured as **% of CPU used vs. maximum vCPUs available**
* **Important:** CPU does not scale 1:1 with ACU.
* **Example:** Writer max ACU = 128 → 32 vCPUs; CPU usage may be 20% while ACU is fully allocated.

> Memory Hook: ACU allocation depends on **CPU, memory, and network together**, not just CPU utilization.

---

## 5. Free Memory Metric

* Tracks **unused memory** on a Serverless v2 instance.
* **Calculation:**
  [
  \text{Allocated Memory (GB)} = \text{ACU Allocated} \times 2
  ]
  [
  \text{Free Memory (GB)} = \text{Max Memory at max ACU} - \text{Allocated Memory}
  ]

**Behavior:** As ACU allocation decreases, free memory increases, and vice versa.

---

## 6. Temp Storage IOPS Metric

* Measures **number of IOPS on local storage**
* Helps identify if **disk activity or network transfer** is driving unexpected ACU scaling
* Use with CPU & memory metrics to diagnose performance issues

---

## 7. Performance Insights for Serverless v2

* Optional, but recommended for deeper monitoring
* Uses **additional CPU, memory, and network**, which can slightly increase ACU usage
* Dashboard exposes:

  * Multi-core metrics
  * Database-level counters
  * Serverless v2-specific counters

**Best Practice:**

* Set **minimum ACU ≥ 2** if Performance Insights is enabled

---

## 8. Practical Example

**Cluster:**

* 3 serverless instances, ACU range: 2–16

**Scenario:**

* Writer under heavy load → scales to 16 ACU
* Reader P1 → scales to 11.5 ACU
* Reader P2/P5 → remains at 2.5 ACU
* ACU Utilization: Writer ≈ 72%, Reader ≈ 16%
* Free Memory increases as ACU decreases for low-load replicas
* Temp Storage IOPS indicates storage activity

<img width="1024" height="579" alt="image" src="https://github.com/user-attachments/assets/1638342b-f6cb-4334-8d73-169a887dbd8d" />

<img width="1087" height="519" alt="image" src="https://github.com/user-attachments/assets/75aa8b58-0223-431a-8923-f717e6613f29" />

**Actionable Insights:**

* If ACU stays near max → increase maximum ACU
* If ACU stays near minimum → decrease minimum ACU to save costs
* Monitor CPU, memory, and IOPS together to determine bottlenecks

---

## 9. Key Points Recap

* Serverless v2 introduces **additional CloudWatch metrics**
* **Serverless Database Capacity**: Tracks ACU allocated to each instance
* **ACU Utilization**: Percentage relative to max ACU
* **Free Memory & Temp Storage IOPS**: Help diagnose scaling triggers
* **Performance Insights**: Provides deeper monitoring, but uses resources
* Always **continuously monitor** metrics to optimize **performance and cost**

---

## 10. FAQ

**Q1: Are traditional Aurora metrics applicable to Serverless v2?**
A: Yes, all standard metrics still apply.

**Q2: What is the Serverless Database Capacity metric?**
A: Tracks ACU allocated to an instance at a point in time.

**Q3: How is ACU utilization calculated?**
A: ACU allocated ÷ Maximum ACU in the cluster × 100

**Q4: Does CPU utilization directly trigger scaling?**
A: Not always. ACU scaling considers CPU, memory, network, read/write load, and background processes.

**Q5: How can these metrics help manage costs?**
A: By monitoring minimum and maximum ACU usage, you can adjust cluster ranges to reduce cost while maintaining performance.

**Q6: Should I enable Performance Insights for Serverless v2?**
A: Recommended for detailed monitoring, but ensure minimum ACU is set ≥2 to accommodate additional load.

