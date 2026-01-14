# Aurora PostgreSQL – Storage and Compute Scaling

## Memory Hook 🧠

**Aurora separates compute from storage.**
Compute can scale (vertically or horizontally), storage auto-scales, and applications stay stable by connecting through **cluster endpoints**, not individual instances.

---

## 1. Aurora Storage Architecture Overview

Aurora uses **three types of storage**:

| Storage Type               | Purpose                  | Managed By User?    |
| -------------------------- | ------------------------ | ------------------- |
| **Local Instance Storage** | Temporary operations     | ❌ No                |# Aurora PostgreSQL Compute Scaling

## Lesson Objective

Understand how **Amazon Aurora PostgreSQL** supports:

* **Vertical scaling** (scale up / scale down DB instances)
* **Horizontal scaling** (read replicas)
* **Automatic scaling** (Aurora Auto Scaling)
* Operational impact during scaling
* Why downtime is minimal during scaling
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
* Higher network bandwidth

Examples:

* Scale up: `db.r5.xlarge → db.r5.2xlarge`
* Scale down: `db.r5.2xlarge → db.r5.xlarge`

---

### Key Characteristics

* Applies to **individual DB instances**
* **Manual operation**
* Can be performed on an **active cluster**
* Target instance class must be:

  * Available in the region
  * Supported for the PostgreSQL engine version

---

### Why Vertical Scaling Is Important

Aurora PostgreSQL supports:

* **Only one writer (primary) instance** per cluster

From an infrastructure perspective:

* If the **writer becomes CPU or memory bound**, the only way to increase capacity is **vertical scaling**

---

### Seasonal Scaling Example

A database with **seasonal traffic** does not need large capacity year-round.

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

1. Select the DB instance
2. Click **Modify**
3. Choose the target instance class
4. Apply immediately or during maintenance window

#### Using AWS CLI

```bash
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.r5.2xlarge
```

---

### Impact During Vertical Scaling

| Cluster Setup               | Impact                      |
| --------------------------- | --------------------------- |
| **Single-instance cluster** | Application outage          |
| **Cluster with ≥ 1 reader** | Failover → minimal downtime |

---

### ❓ Why Is Downtime Minimal?

During vertical scaling, Aurora performs the following automatically:

1. **Primary instance is taken offline** for modification
2. **A reader is promoted to writer** (if available)
3. **Endpoints are updated transparently**

Aurora automatically adjusts:

* **Writer endpoint** → points to the newly promoted writer
* **Reader endpoint** → continues routing read traffic

⏱ **Typical failover time:**

* ~30 seconds to 2 minutes (varies by workload and instance size)

📌 Applications that:

* Use **cluster writer endpoint** for writes
* Use **cluster reader endpoint** for reads

Experience **minimal or no noticeable downtime**.

---

### Memory Hook 🧠

**Vertical scaling = engine swap**
Same car, stronger engine — brief stop unless another engine is already running.

---

## Horizontal Scaling (Read Replicas)

### What Is Horizontal Scaling?

Horizontal scaling means **adding or removing reader instances**.

Benefits:

* Increases **read throughput**
* Improves **availability**
* Reduces load on writer

Aurora automatically:

* Distributes read traffic via the **reader endpoint**

---

### Manual Horizontal Scaling

Users may:

* Add read replicas manually
* Remove read replicas manually

Best for:

* Predictable workloads
* Planned scaling events

---

## Automatic Scaling (Aurora Auto Scaling)

### What Is Aurora Auto Scaling?

Aurora Auto Scaling automatically:

* Adds read replicas (scale out)
* Removes read replicas (scale in)

Based on a **scaling policy** attached to the cluster.

---

### Requirements for Auto Scaling

✔ At least **one reader instance** must exist
✔ All instances must be in **Available** state
✔ Applications must connect to:

```
Cluster Reader Endpoint
```

❌ Connecting directly to instance endpoints bypasses auto scaling benefits

---

### Auto Scaling Metrics (Target Metrics)

Each auto scaling policy tracks **one target metric**.

Predefined metrics:

| Metric                           | Description                     |
| -------------------------------- | ------------------------------- |
| **Average CPU Utilization**      | Average CPU across replicas     |
| **Average Database Connections** | Avg connections across replicas |

Example:

* Target CPU = **50%**

  * CPU > 50% → scale out
  * CPU < 50% → scale in

📌 One metric per policy
📌 Multiple policies per cluster allowed

---

### How Auto Scaling Works

* Policy maintains the **target metric value**
* Adds replicas as load increases
* Removes replicas as load decreases
* New replicas use:

  * **Same instance class as the writer**

---

### Cooldown Periods

Cooldowns prevent rapid scaling oscillation.

| Cooldown Type          | Purpose                            |
| ---------------------- | ---------------------------------- |
| **Scale-out cooldown** | Wait time after adding a replica   |
| **Scale-in cooldown**  | Wait time after removing a replica |

Cooldown values are specified in **seconds**.

---

### Replica Removal Rules

* Auto scaling removes **only replicas it created**
* Manually created replicas:

  * Are not removed automatically
  * Must be deleted manually

Auto scaling may also be:

* Temporarily disabled

---

### Memory Hook 🧠

**Auto scaling = thermostat**
Adds or removes heaters (replicas) to maintain temperature (metric).

---

## Scaling Events & Monitoring

### Scaling Events

Scaling generates **DB instance events**, including:

* Replica creation
* Replica deletion
* Failover
* Instance modification

---

### Where to View Scaling Activity

* AWS Console → DB Cluster
* **Logs & Events** tab

Users may:

* Subscribe to events
* Trigger alerts or automation

---

## Summary of Compute Scaling

| Scaling Type | Manual / Auto | Primary Use Case         |
| ------------ | ------------- | ------------------------ |
| Vertical     | Manual        | Increase writer capacity |
| Horizontal   | Manual        | Increase read capacity   |
| Auto Scaling | Automatic     | Dynamic read scaling     |

---

## Key Takeaways

* Aurora supports:

  * Vertical scaling of DB instances
  * Horizontal scaling using replicas
  * Automatic scaling via policies
* Vertical scaling:

  * Is manual
  * May cause outage without readers
* Auto scaling:

  * Applies only to **read replicas**
  * Requires use of **reader endpoint**
* Failover keeps downtime minimal when readers exist

---

## FAQ

### Q1. Can Aurora scale the writer automatically?

❌ No. Writer scaling is **manual (vertical only)**.

---

### Q2. Does auto scaling change instance size?

❌ No. Auto scaling only **adds or removes replicas**.

---

### Q3. What happens during vertical scaling?

* Primary becomes unavailable
* Reader is promoted (if present)
* Endpoints are updated automatically

---

### Q4. Can I attach multiple auto scaling policies?

✔ Yes
❌ One target metric per policy

---

### Q5. Why must apps use the reader endpoint?

Because replicas are added/removed dynamically, and the reader endpoint always routes correctly.

---

### Final Memory Hook 🧠

```
Vertical scaling → Bigger instance
Horizontal scaling → More replicas
Auto scaling → Automatic replica management
```

| **Aurora Cluster Volume**  | Persistent database data | ❌ No (auto-managed) |
| **Amazon S3**              | Automated backups        | ❌ No                |

This lesson focuses on **local storage** and **Aurora cluster volume**.

---

## 2. Local Storage (Instance-Attached Storage)

### Key Characteristics

* Physically attached to each DB instance
* **Fixed size** (cannot be resized or extended)
* Size is typically **~2× the RAM of the DB instance**
* To increase local storage → **scale up the DB instance**

📌 Users **cannot**:

* Resize the EBS volume
* Attach additional volumes

---

## 3. What Is Local Storage Used For?

Local storage is **not used for persistent database data**.
It is mainly used for **temporary operations**.

### Main Use Cases

### 1️⃣ PostgreSQL Log Files

* Server logs
* Error logs
* Optional debug logs

⚠️ Excessive logging (e.g., `log_min_messages = debug`) can consume local storage quickly and impact query processing.

---

### 2️⃣ Query Processing (Temporary Files)

PostgreSQL writes **intermediate results** to temporary files during query execution when data cannot fit in memory.

#### Common Examples

* `ORDER BY` on large tables without indexes
* Hash joins
* Merge joins
* Large aggregations (`GROUP BY`, `COUNT`, `SUM` on big datasets)

📌 These temporary files:

* Exist **only during query execution**
* Are automatically removed after query completion
* Cause **short-lived spikes** in local storage usage

---

### What Happens If Local Storage Is Exhausted?

* Queries may fail with **disk space errors**
* Query processing slows down or stops

#### Monitoring Metric

* **CloudWatch: `FreeLocalStorage`**

Monitoring this metric helps:

* Detect risky workloads
* Prevent query failures

---

## 4. Aurora Cluster Volume (Persistent Storage)

### Key Characteristics

* Shared across **all instances** in the cluster
* Automatically scales **up and down**
* Used for **actual database data**
* Maximum size: **128 TB**
* You pay only for **storage actually used**

### Behavior Examples

| Operation             | Effect on Storage |
| --------------------- | ----------------- |
| Insert data           | Storage increases |
| Drop / truncate table | Storage decreases |

---

### Monitoring Cluster Storage

#### Key CloudWatch Metrics

| Metric                 | Meaning                    |
| ---------------------- | -------------------------- |
| `VolumeBytesUsed`      | Storage currently consumed |
| `VolumeBytesLeftTotal` | Remaining growth capacity  |

📌 `VolumeBytesLeftTotal` is a convenience metric.
You can also calculate remaining space as:

```
128 TB – VolumeBytesUsed
```

These metrics help track **operational storage cost**.

---

## 5. Aurora Compute Scaling Overview

Aurora supports **three types of compute scaling**:

| Scaling Type | What Scales        | How                 |
| ------------ | ------------------ | ------------------- |
| Vertical     | Instance size      | Manual              |
| Horizontal   | Number of replicas | Manual or Automatic |
| Automatic    | Replicas           | Policy-based        |

---

## 6. Vertical Scaling (Instance Scaling)

### What Is Vertical Scaling?

Changing the **DB instance class**.

Examples:

* Scale up: `db.r5.xlarge → db.r5.2xlarge`
* Scale down: `db.r5.2xlarge → db.r5.xlarge`

### Key Points

* Performed **manually**
* Can be done on an **active cluster**
* Instance becomes temporarily unavailable during modification

### How to Perform

* AWS Console → Modify DB instance
* AWS CLI / API (`modify-db-instance`)

📌 Before scaling:

* Ensure the target instance class is supported in the region
* Ensure it is supported for your PostgreSQL version

---

## 7. Impact of Vertical Scaling on Availability

### Without Read Replica

* Writer becomes unavailable
* ❌ Application outage

### With At Least One Read Replica

* Aurora performs **promotion-based modification**
* ✔ Minimal downtime

---

## 8. Aurora Endpoints and Promotion (Critical Concept)

### Endpoint Types

| Endpoint        | Purpose                         |
| --------------- | ------------------------------- |
| Writer endpoint | Always points to current writer |
| Reader endpoint | Load-balances across replicas   |

### What Happens During Modification?

* Aurora may **promote a read replica** to writer
* Writer endpoint automatically updates
* Reader endpoint automatically updates

📌 Applications do **not** need configuration changes if they use cluster endpoints.

---

## ❓ Why Is Downtime Minimal? (With Time Explanation)

Downtime is minimal because of **three Aurora design choices**:

### 1️⃣ Shared Cluster Storage (0–5 seconds impact)

* All instances use the same underlying data volume
* No data copy required during promotion

### 2️⃣ Fast Replica Promotion (Typically < 30 seconds)

* Replica already has up-to-date data
* Only role switch is required

### 3️⃣ Automatic DNS Update for Endpoints (Seconds)

* Writer endpoint DNS is updated automatically
* Applications reconnect without config change

📌 **Typical observed impact:**

* Few seconds to **tens of seconds**, not minutes

---

## 9. Horizontal Scaling (Read Replicas)

Aurora supports horizontal scaling by:

* Manually adding/removing replicas
* Automatically managing replicas via auto scaling

Replica benefits:

* Read scaling
* High availability
* Reduced downtime during maintenance

---

## 10. Aurora Auto Scaling of Replicas

### Requirements

* At least **one read replica**
* All instances must be in **available** state
* Applications must use **reader endpoint**

---

### Auto Scaling Policy

* Tracks a **target metric**
* Adds or removes replicas to maintain desired value

#### Predefined Metrics

| Metric                  | Description     |
| ----------------------- | --------------- |
| Average CPU utilization | Across replicas |
| Average DB connections  | Across replicas |

Example:

* Target CPU = 50%
* CPU > 50% → scale out
* CPU < 50% → scale in

---

### Cooldown Periods

| Cooldown Type      | Purpose                    |
| ------------------ | -------------------------- |
| Scale-out cooldown | Time before next scale-out |
| Scale-in cooldown  | Time before next scale-in  |

Prevents rapid scaling fluctuations.

---

### Replica Removal Rules

* Auto scaling removes **only replicas it created**
* Manually created replicas are not deleted automatically

---

## 11. Monitoring Scaling Events

* Scaling events are logged as **DB instance events**
* Viewable in:

  * RDS Console → Logs & Events

---

## Key Takeaways ✅

* Local storage is temporary and fixed in size
* Aurora cluster storage auto-scales up to 128 TB
* Vertical scaling is manual and may cause brief unavailability
* Read replicas enable minimal downtime during scaling
* Auto scaling manages replicas using policies
* Always use **writer and reader endpoints**

---

## FAQ ❓

### Q1. Can I resize local storage directly?

❌ No. You must scale the DB instance.

---

### Q2. Does replica promotion cause data loss?

❌ No. All instances share the same storage.

---

### Q3. How long does downtime usually last?

✔ Typically **a few seconds to under a minute**, depending on workload.

---

### Q4. Why should apps use reader endpoints?

To benefit from:

* Load balancing
* Auto scaling
* Failover transparency

---

### Q5. Who pays for storage in Aurora?

You pay only for **actual storage used**, not allocated capacity.
