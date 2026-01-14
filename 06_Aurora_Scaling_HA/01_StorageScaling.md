# Aurora PostgreSQL Storage & Local Storage Usage

## Lesson Objective

## Types of Storage Used by Aurora

Aurora PostgreSQL uses **three types of storage**:

| Storage Type              | Purpose                                             |
| ------------------------- | --------------------------------------------------- |
| **Local Storage**         | Temporary data & logs (attached to DB instance)     |
| **Aurora Cluster Volume** | Persistent database storage (shared across cluster) |
| **Amazon S3**             | Automated backups                                   |

📌 **Focus of this lesson:** Local storage and Aurora cluster volume

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

### Typical Size

* Usually **~2× the instance RAM**
* Depends on **DB instance type**

---

## What Is Local Storage Used For?

Local storage is used **primarily for temporary operations**, not persistent data.
It acts as a **scratchpad** where PostgreSQL writes **intermediate results or logs**.

---

### 1️⃣ PostgreSQL Log Files

Local storage holds **server logs**, including:

| Log Type                | Purpose                                                                 |
| ----------------------- | ----------------------------------------------------------------------- |
| **Server logs**         | Track database engine activity (startup, shutdown, errors, maintenance) |
| **Error logs**          | Record SQL or system errors (failed queries, deadlocks, memory issues)  |
| **Optional debug logs** | Detailed logs for troubleshooting or development                        |

**Notes:**

* Logs are rotated based on `log_rotation_age` or `log_rotation_size`.
* Excessive logging (e.g., debug level) can **consume large amounts of local storage**, potentially blocking queries.

💡 **Memory Hook:**

> Local storage for logs = “notepad” where PostgreSQL writes what it’s doing. If the notepad overflows, the database cannot continue operations.

---

### 2️⃣ Query Processing (Temporary Files)

PostgreSQL often needs more working space than **RAM** to execute certain queries.
It writes **temporary files to local storage** when operations exceed memory allocation.

**Main Scenarios:**

| Scenario                                     | Description                                                                        |
| -------------------------------------------- | ---------------------------------------------------------------------------------- |
| **ORDER BY on large tables without indexes** | Sorts rows too large to fit in memory → writes intermediate sorted chunks to disk  |
| **Hash Joins / Merge Joins**                 | Joins large tables → spills data to temp files if memory is insufficient           |
| **Large aggregations**                       | GROUP BY or SUM operations with huge datasets → uses disk for intermediate results |

**How Temporary Files Work:**

1. PostgreSQL reads pages into memory
2. If operation exceeds `work_mem` → spills to **temporary files** on local storage
3. Query completes → temp files deleted → storage freed

**Example:**

```sql
SELECT * FROM orders ORDER BY total_amount;
```

* Without an index and with millions of rows, PostgreSQL uses **temp files** on local storage

**Risk:**

* If local storage fills up → query fails:

> ERROR: could not write to temporary file: No space left on device

💡 **Memory Hook:**

> Query processing like cooking on a countertop:
> RAM = countertop space, Local storage = side table for overflow. Too little side table → can’t finish the recipe → query fails

---

### Monitoring Local Storage

**CloudWatch Metric:**

```
FreeLocalStorage
```

* Shows available space on instance-attached storage
* Helps prevent query failures
* Tracks high log usage or excessive temp file creation

**Best Practices:**

* Monitor frequently for high usage
* Increase `work_mem` judiciously if temp file usage is high
* Avoid excessive logging in production

---

## Aurora Cluster Volume (Shared Storage)

### Key Characteristics

* Shared by **all DB instances** in the cluster
* Automatically scales **up and down**
* Stores **persistent database data**
* No manual provisioning required
* Pay-per-use: you pay only for storage used

---

### How Cluster Storage Grows & Shrinks

| Action         | Effect on Storage |
| -------------- | ----------------- |
| Insert data    | Increases usage   |
| Drop table     | Decreases usage   |
| Truncate table | Decreases usage   |

---

### Maximum Cluster Volume Size

* Maximum = **128 TB**
* Automatically managed by Aurora
* Scaling is transparent to users

---

### Monitoring Cluster Volume Storage

**Key CloudWatch Metrics:**

| Metric                 | Description                                 |
| ---------------------- | ------------------------------------------- |
| `VolumeBytesUsed`      | Storage currently used by cluster           |
| `VolumeBytesLeftTotal` | Remaining space before hitting 128 TB limit |

* Track usage trends
* Estimate storage growth
* Monitor operational costs

💡 **Memory Hook:**

> Cluster volume = “elastic warehouse” that grows/shrinks automatically as inventory (data) changes.

---

## Key Differences: Local vs Cluster Storage

| Feature           | Local Storage      | Cluster Volume        |
| ----------------- | ------------------ | --------------------- |
| Scope             | Single instance    | Shared across cluster |
| Persistence       | Temporary          | Persistent            |
| Scaling           | Fixed              | Automatic             |
| Resize            | Scale up instance  | Automatic             |
| Monitoring Metric | `FreeLocalStorage` | `VolumeBytesUsed`     |

---

## Visual Diagram: Local vs Cluster Storage

```
                    +----------------------+
                    |   Aurora Cluster     |
                    |   Persistent Data    |
                    |   (VolumeBytesUsed)  |
                    +----------------------+
                       ^           ^
                       |           |
           Shared by all DB   CloudWatch
            instances in       Metric
            cluster
                       
                       
+--------------------+                 +----------------------+
|   DB Instance 1    |                 |   DB Instance 2      |
|--------------------|                 |----------------------|
| Local Storage      |                 | Local Storage        |
| - Temp query files |                 | - Temp query files   |
| - Logs             |                 | - Logs               |
| (FreeLocalStorage) |                 | (FreeLocalStorage)   |
+--------------------+                 +----------------------+
          ^                                    ^
          |------------------------------------|
                 Connection & Query workload
```

**Diagram Explanation:**

* **Local Storage** = per instance, temporary scratchpad for logs & query temp files
* **Cluster Volume** = shared, persistent storage
* **Monitoring** via CloudWatch metrics: `FreeLocalStorage` for local, `VolumeBytesUsed` for cluster

---

## Key Takeaways

* **Local storage:**

  * Fixed size, temporary
  * Holds logs & temp query files
  * Monitor to prevent failures
* **Cluster storage:**

  * Automatically scales
  * Holds persistent data
  * Monitor for growth and cost
* Always monitor:

  * **Local storage** → availability
  * **Cluster storage** → growth trends

---

## FAQ

**Q1. Can I resize local storage directly?**

* No. Scale up the DB instance to increase local storage.

**Q2. What happens if local storage fills up?**

* Queries fail
* Temporary operations cannot complete
* Performance degrades

**Q3. Which metric should I monitor for local storage?**

* `FreeLocalStorage`

**Q4. Does Aurora cluster storage shrink automatically?**

* Yes. Dropping or truncating tables reduces usage

**Q5. What is the maximum Aurora cluster volume size?**

* 128 TB

**Q6. Do I need to monitor S3 backup storage?**

* No. S3 is fully managed and scales automatically

---

✅ **Memory Hook Summary:**

* Local storage = scratchpad (temporary, per instance)
* Cluster storage = elastic warehouse (persistent, shared, auto-scaling)
