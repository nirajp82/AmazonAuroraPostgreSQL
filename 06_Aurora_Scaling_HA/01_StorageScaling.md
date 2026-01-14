# Aurora PostgreSQL Storage Architecture & Monitoring

## Lesson Objective

Understand:

* How storage is managed in **Aurora PostgreSQL**
* Differences between **local storage** and **cluster volume**
* Key **CloudWatch metrics** used to monitor storage usage
* Best practices to avoid storage-related issues

---

## Types of Storage Used by Aurora

Aurora PostgreSQL uses **three types of storage**:

| Storage Type              | Purpose                                         |
| ------------------------- | ----------------------------------------------- |
| **Local Storage**         | Temporary data & logs (attached to DB instance) |
| **Aurora Cluster Volume** | Persistent database storage (shared)            |
| **Amazon S3**             | Automated backups                               |

📌 **This lesson focuses only on**:

* Local storage
* Aurora cluster volume
  (S3 scaling is fully managed and does not require monitoring)

---

## Local Storage (Instance-Attached)

### Key Characteristics

* Attached directly to the **DB instance**
* Backed by **EBS**
* **Fixed size**
* Cannot be resized manually
* Cannot attach additional volumes

To increase local storage:
👉 **Scale up the DB instance**

---

### How Much Local Storage Is Available?

* Typically **~2× the instance RAM**
* Exact size depends on instance type

⚠️ **Clarification (Transcript ambiguity)**
The user **cannot control or configure** local storage size directly.

---

## What Is Local Storage Used For?

Local storage is used primarily for **temporary operations**, not persistent data.

### Main Use Cases

#### 1️⃣ PostgreSQL Log Files

* Server logs
* Error logs
* Optional debug logs

#### 2️⃣ Query Processing (Temporary Files)

Examples:

* `ORDER BY` on large tables without indexes
* Hash joins
* Merge joins
* Large aggregations

📌 PostgreSQL writes intermediate data to **temporary files** during query execution.

---

### Temporary Usage Pattern

* Storage is used **only while query executes**
* Space is released after query completion

---

### Risk: Excessive Logging

If logging levels are set too high (e.g., `DEBUG`):

* Log files grow rapidly
* Local storage may fill up
* Query execution may fail

Example failure:

> Query cannot be processed due to **local storage exhaustion**

---

### Monitoring Local Storage

Use CloudWatch metric:

```
FreeLocalStorage
```

✔ Helps:

* Detect growing temp file usage
* Identify excessive logging
* Prevent query failures

---

### Memory Hook 🧠

**Local storage = scratchpad**
If the scratchpad fills up, the database can’t think.

---

## Aurora Cluster Volume (Shared Storage)

### Key Characteristics

* Shared by **all DB instances** in the cluster
* Automatically scales up and down
* Used for **persistent database data**
* No manual provisioning required

---

### How Cluster Storage Grows & Shrinks

| Action         | Storage Effect  |
| -------------- | --------------- |
| Insert data    | Increases usage |
| Drop table     | Decreases usage |
| Truncate table | Decreases usage |

---

### Maximum Cluster Volume Size

⚠️ **Correction of Transcript Error**

❌ Transcript: *“maximum size is 128 bytes”*
✅ **Correct**: Maximum size is **128 TB**

* You pay only for **storage actually used**
* Scaling is automatic

---

## Monitoring Cluster Volume Storage

### Key CloudWatch Metrics

#### 1️⃣ `VolumeBytesUsed`

* Total cluster storage currently used
* Helps monitor:

  * Growth trends
  * Storage cost

---

#### 2️⃣ `VolumeBytesLeftTotal`

* Remaining storage before hitting 128 TB limit
* Convenience metric

Equivalent calculation:

```
128 TB – VolumeBytesUsed
```

---

### Why These Metrics Matter

* Prevents unexpected storage exhaustion
* Helps forecast growth
* Assists in cost management

---

### Memory Hook 🧠

**Cluster volume = elastic warehouse**
It grows and shrinks automatically as inventory changes.

---

## Key Differences: Local vs Cluster Storage

| Feature           | Local Storage     | Cluster Volume      |
| ----------------- | ----------------- | ------------------- |
| Scope             | Per instance      | Shared cluster-wide |
| Persistence       | Temporary         | Persistent          |
| Scaling           | Fixed             | Automatic           |
| Resize            | Instance scale-up | Automatic           |
| Monitoring Metric | FreeLocalStorage  | VolumeBytesUsed     |

---

## Key Takeaways

* Local storage:

  * Fixed size
  * Used for logs & temp query files
  * Requires careful monitoring
* Cluster storage:

  * Automatically scales
  * Used for database data
  * Pay-per-use
* Always monitor:

  * **Local storage** → availability
  * **Cluster storage** → growth & cost

---

## FAQ

### Q1. Can I resize local storage directly?

**No.**
You must scale up the DB instance.

---

### Q2. What happens if local storage fills up?

* Queries may fail
* Temporary operations cannot complete
* Database performance degrades

---

### Q3. Which metric should I monitor for local storage?

👉 `FreeLocalStorage`

---

### Q4. Does Aurora cluster storage ever shrink?

**Yes.**

* Dropping or truncating tables reduces usage

---

### Q5. What is the maximum Aurora cluster volume size?

👉 **128 TB**

---

### Q6. Do I need to worry about S3 backup storage?

**No.**

* Fully managed
* Automatically scaled

---

### Final Memory Hook 🧠

**If it’s temporary → local storage**
**If it’s data → cluster volume**
