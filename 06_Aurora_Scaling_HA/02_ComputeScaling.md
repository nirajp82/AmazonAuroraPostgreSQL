# Aurora PostgreSQL – Storage and Compute Scaling

## Memory Hook 🧠

**Aurora separates compute from storage.**
Compute can scale (vertically or horizontally), storage auto-scales, and applications stay stable by connecting through **cluster endpoints**, not individual instances.

---

## 1. Aurora Storage Architecture Overview

Aurora uses **three types of storage**:

| Storage Type               | Purpose                  | Managed By User?    |
| -------------------------- | ------------------------ | ------------------- |
| **Local Instance Storage** | Temporary operations     | ❌ No                |
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
