# Aurora PostgreSQL Compute Scaling

## Lesson Objective

Understand how **Amazon Aurora PostgreSQL** supports:

* **Vertical scaling** (scale up / down DB instances)
* **Horizontal scaling** (read replicas)
* **Automatic scaling** (Aurora Auto Scaling)
* Operational impact during scaling
* Monitoring and events related to scaling activities

---

## Types of Compute Scaling in Aurora

Aurora supports **three forms of compute scaling**:

| Scaling Type           | Description                                           |
| ---------------------- | ----------------------------------------------------- |
| **Vertical Scaling**   | Change DB instance class (bigger or smaller instance) |
| **Horizontal Scaling** | Add or remove read replicas                           |
| **Automatic Scaling**  | Automatically add/remove replicas based on policies   |

---

## Vertical Scaling (Scale Up / Scale Down)

### What Is Vertical Scaling?

Vertical scaling means **changing the DB instance class** to one with:

* More CPU
* More memory
* More network bandwidth

Example:

* Scale up: `db.r5.xlarge → db.r5.2xlarge`
* Scale down: `db.r5.2xlarge → db.r5.xlarge`

---

### Key Characteristics

* Applies to **DB instances**
* **Manual operation**
* Can be done on an **active cluster**
* Requires the target instance type to be:

  * Available in the region
  * Supported for the PostgreSQL version

---

### Why Vertical Scaling Is Important

Aurora PostgreSQL currently supports:

* **Single writer (primary) instance**

From an infrastructure perspective:

* If the **writer becomes a bottleneck**, the only way to increase its capacity is **vertical scaling**

---

### Seasonal Scaling Example

A database with **seasonal traffic** does not need a large instance all year.

Example:

* Online retail store
* Scale up during:

  * Holiday season
* Scale down after:

  * Traffic normalizes

✔ Saves cost by avoiding unused capacity

---

### How to Perform Vertical Scaling

#### Using AWS Console

1. Select DB instance
2. Click **Modify**
3. Choose target instance class
4. Apply immediately or during maintenance window

#### Using AWS CLI

```bash
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.r5.2xlarge
```

---

### Impact During Vertical Scaling

| Scenario                    | Impact                           |
| --------------------------- | -------------------------------- |
| **Single-instance cluster** | Application outage               |
| **Cluster with ≥ 1 reader** | Failover occurs → minimal impact |

Explanation:

* Aurora automatically adjusts:

  * Writer endpoint
  * Reader endpoint
* A reader may be promoted during modification

---

### Memory Hook 🧠

**Vertical scaling = bigger engine**
Same car, stronger engine — but you must stop briefly to replace it.

---

## Horizontal Scaling (Read Replicas)

### What Is Horizontal Scaling?

Horizontal scaling means **adding or removing reader instances**.

* Increases **read throughput**
* Improves **availability**
* Reduces load on writer

Aurora automatically:

* Distributes read traffic via **reader endpoint**

---

### Manual Horizontal Scaling

Users can:

* Add replicas manually
* Remove replicas manually

This is useful when:

* Workload is predictable
* Scaling is planned

---

## Automatic Scaling (Aurora Auto Scaling)

### What Is Aurora Auto Scaling?

Aurora Auto Scaling automatically:

* Adds read replicas (scale out)
* Removes read replicas (scale in)

Based on a **scaling policy** attached to the cluster.

---

### Requirements for Auto Scaling

✔ At least **one reader** must exist
✔ All instances must be in **Available** state
✔ Applications must connect to:

```
Cluster Reader Endpoint
```

❌ If apps connect directly to instance endpoints, auto scaling provides no benefit

---

### Auto Scaling Metrics (Target Metrics)

An auto scaling policy tracks **one target metric**.

Aurora provides **two predefined metrics**:

| Metric                           | Description                     |
| -------------------------------- | ------------------------------- |
| **Average CPU Utilization**      | Avg CPU across replicas         |
| **Average Database Connections** | Avg connections across replicas |

Example:

* Target CPU = **50%**

  * CPU > 50% → scale out
  * CPU < 50% → scale in

📌 Only **one target metric per policy**
📌 Multiple policies per cluster are allowed (one metric per policy)

---

### How Auto Scaling Works

* Policy tries to **maintain target value**
* Scales **out** when load increases
* Scales **in** when load decreases
* New replicas use:

  * **Same instance class as primary**

---

### Cooldown Periods

Cooldowns prevent **rapid oscillation** of scaling.

| Cooldown Type          | Purpose                            |
| ---------------------- | ---------------------------------- |
| **Scale-out cooldown** | Wait time after adding a replica   |
| **Scale-in cooldown**  | Wait time after removing a replica |

Cooldown is specified in **seconds**.

---

### Replica Removal Rules

* Auto scaling removes **only replicas created by auto scaling**
* Manually created replicas:

  * Are not deleted automatically
  * Must be removed manually

Auto scaling can also be:

* Temporarily disabled

---

### Memory Hook 🧠

**Auto scaling = thermostat**
It adds or removes heaters (replicas) to keep the temperature (metric) steady.

---

## Scaling Events & Monitoring

### Scaling Events

Scaling generates **DB instance events**, such as:

* Replica creation
* Replica deletion
* Failover
* Instance modification

---

### Where to View Scaling Activity

* AWS Console → DB Cluster
* **Logs & Events** tab
* Auto scaling activity history is visible here

Users can:

* Subscribe to scaling events
* Trigger alerts or automation

---

## Summary of Compute Scaling

| Scaling Type | Manual / Auto | Use Case                 |
| ------------ | ------------- | ------------------------ |
| Vertical     | Manual        | Increase writer capacity |
| Horizontal   | Manual        | Increase read capacity   |
| Auto Scaling | Automatic     | Dynamic read scaling     |

---

## Key Takeaways

* Aurora supports:

  * Vertical scaling of DB instances
  * Horizontal scaling using replicas
  * Automatic scaling using policies
* Vertical scaling:

  * Manual
  * May cause outage without readers
* Auto scaling:

  * Works only for **read replicas**
  * Requires apps to use reader endpoint
* Cooldowns prevent unstable scaling behavior

---

## FAQ

### Q1. Can Aurora scale the writer automatically?

❌ No.
Writer scaling is **manual (vertical only)**.

---

### Q2. Does auto scaling change instance size?

❌ No.
Auto scaling **adds/removes replicas**, not instance size.

---

### Q3. What happens during vertical scaling?

* Instance becomes unavailable
* Failover occurs if readers exist

---

### Q4. Can I attach multiple auto scaling policies?

✔ Yes
❌ Only one target metric per policy

---

### Q5. Why must apps use the reader endpoint?

Because auto scaling adds/removes replicas dynamically — the reader endpoint automatically routes traffic.

---

### Final Memory Hook 🧠

```
Vertical scaling → Bigger instance
Horizontal scaling → More replicas
Auto scaling → Automatic replica management
```
* Add **real outage scenarios during scaling**
* Move to the **hands-on vertical scaling exercise README**
