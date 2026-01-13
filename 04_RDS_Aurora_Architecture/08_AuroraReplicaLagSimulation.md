# Amazon Aurora Replica Lag – SQL-Level Deep Dive & Failure Simulation

This document explains **how Aurora replica lag works**, **which SQL utilities are involved**, and **how to safely simulate replica lag** using Aurora’s built-in fault injection framework.

The goal is to understand **what each SQL statement does**, **what happens internally**, and **how Aurora reacts**.

---

## 1. Why Replica Lag Exists in Aurora

Aurora uses a **shared distributed storage volume**, so **data replication is NOT the same as PostgreSQL streaming replication**.

### What Still Needs Replication?

Even though storage is shared:

* **Each DB instance has its own buffer cache**
* The **writer updates its buffer cache immediately**
* Readers may still have **stale pages in memory**

To solve this, Aurora replicates **buffer cache changes** from the writer to reader instances.

This replication is:

* Memory-to-memory
* Network-based
* Asynchronous
* Extremely fast (usually < 100 ms)

This delay is what we call **Aurora replica lag**.

---

## 2. SQL to Check Replica Lag

### 2.1 `aurora_replica_status()`

```sql
SELECT aurora_replica_status();
```

### What This SQL Does

This function:

* Queries Aurora’s internal replication metadata
* Reports how far behind the replica is compared to the writer
* Returns **replica lag in milliseconds**

### Why This Matters

* This is the **most direct and accurate way** to check lag
* It reflects **buffer cache replication delay**, not storage lag
* It is the same lag shown in:

  * RDS Console
  * CloudWatch metrics

### Typical Output

```
8
19
27
```

### Interpretation

* Lag < 100 ms → **Healthy**
* Lag spikes during heavy writes → **Normal**
* Sustained high lag → **Problem**

---

## 3. Aurora Fault Injection Framework (Why It Exists)

Aurora provides **fault injection** so engineers can:

* Test failure behavior safely
* Validate monitoring and alerts
* Observe automatic recovery

Fault injection **does NOT corrupt data**.
It simulates delays and failures internally.

---

## 4. SQL to Simulate Replica Lag

### 4.1 `aurora_inject_replica_failure(...)`

```sql
SELECT aurora_inject_replica_failure(
    failure_percentage,
    duration_seconds,
    replica_instance_name
);
```

---

## 5. Understanding Each Parameter (Critical)

### Parameter 1: `failure_percentage`

| Value | Meaning                    |
| ----- | -------------------------- |
| 0     | No failure                 |
| 100   | Complete simulated failure |

This controls **how severely the replica is impaired**.

---

### Parameter 2: `duration_seconds`

| Value | Meaning       |
| ----- | ------------- |
| 20    | Short failure |
| 120   | Long failure  |

This controls **how long Aurora will simulate the failure**.

---

### Parameter 3: `replica_instance_name`

Example:

```text
reader-2
```

This is the **DB instance identifier** of the replica.

---

## 6. Simulating Short Replica Lag (Safe Test)

### Step 1: Check Current Lag

```sql
SELECT aurora_replica_status();
```

Example:

```
18
```

---

### Step 2: Inject Replica Failure (20 seconds)

```sql
SELECT aurora_inject_replica_failure(100, 20, 'reader-2');
```

### What Happens Internally

* Aurora artificially delays replication
* Buffer cache updates stop flowing
* Replica starts falling behind
* Storage remains consistent

---

### Step 3: Observe Lag Increase

```sql
SELECT aurora_replica_status();
```

Example:

```
8469
13156
```

### Interpretation

* Lag increases rapidly
* Replica is still **available**
* After 20 seconds → automatic recovery

---

## 7. Simulating Sustained Replica Lag (Restart Scenario)

### Step 1: Inject Failure for 120 Seconds

```sql
SELECT aurora_inject_replica_failure(100, 120, 'reader-2');
```

---

### What Happens Internally

1. Replica cannot keep up
2. Lag keeps increasing
3. Aurora monitors lag continuously
4. Lag exceeds safe threshold
5. Aurora **restarts the replica automatically**

---

### Step 2: Verify Restart in RDS Console

Navigate to:

```
RDS → DB Instances → Replica → Logs & Events
```

You will see an event similar to:

> **Read replica has fallen behind the writer too much.
> Restarting PostgreSQL DB instance.**

---

## 8. Why Aurora Restarts the Replica

Aurora restarts a replica when:

* Lag remains high for too long
* Replica cannot catch up safely
* Buffer cache becomes too stale

Restarting:

* Clears stale memory pages
* Forces fresh reads from storage
* Restores low-lag replication

---

## 9. Important Observations

* Replica lag is **expected**
* Lag spikes under write-heavy workloads
* Lag is usually **< 100 ms**
* Aurora handles lag **automatically**
* Restart is a **protective mechanism**, not a failure

---

## 10. Best Practices

* Use **same instance size** for writer and readers
* Monitor replica lag continuously
* Avoid long-running transactions
* Expect higher lag during write spikes
* Set alert thresholds (e.g., 500 ms)

---

## 11. Exam-Style Q&A

### Q1: Does Aurora use WAL shipping for replication?

**No.**
Aurora uses **buffer cache replication**, not WAL streaming.

---

### Q2: Why does replica lag exist if storage is shared?

Because **buffer caches are not shared** between instances.

---

### Q3: Is replica lag synchronous or asynchronous?

**Asynchronous**, but with very low latency.

---

### Q4: What happens if replica lag becomes too high?

Aurora **automatically restarts the replica**.

---

### Q5: Does fault injection cause data loss?

**No.** It only simulates delays and failures.

---

## 12. Memory Hook

> **“Aurora replicates memory, not storage — and restarts readers when memory falls too far behind.”**
