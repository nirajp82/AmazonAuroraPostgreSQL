# Aurora PostgreSQL Compute Scaling

## Lesson Objective

Understand how **Amazon Aurora PostgreSQL** supports:

* Vertical scaling (scale up / scale down DB instance size)
* Horizontal scaling (read replicas)
* Automatic scaling (Aurora Auto Scaling for replicas)
* Operational impact during scaling
* Monitoring and events related to scaling activities

---

## Overview of Compute Scaling in Aurora

Aurora PostgreSQL supports **three compute scaling models**:

1. **Vertical Scaling** – change the instance size (manual)
2. **Horizontal Scaling** – add or remove read replicas (manual)
3. **Automatic Scaling** – automatically scale read replicas based on metrics (policy-driven)

> **Important Constraint**: Aurora PostgreSQL supports **only one writer (primary) instance** per cluster. This makes vertical scaling critical for writer-side performance.

---

## 1. Vertical Scaling (Scale Up / Scale Down)

### What Is Vertical Scaling?

Vertical scaling means **changing the DB instance class** to a larger or smaller size, for example:

* Scale up: `db.r5.xlarge` → `db.r5.2xlarge`
* Scale down: `db.r5.2xlarge` → `db.r5.xlarge`

This changes CPU, memory, and network capacity of the instance.

---

### When to Use Vertical Scaling

Vertical scaling is useful when:

* The workload is **write-heavy** (writer bottleneck)
* You have **seasonal or predictable traffic spikes**
* You want to avoid paying for unused capacity year-round

**Example**:

An online store scales up DB capacity during the last two months of the year (holiday season) and scales down afterward, reducing cost during low-traffic periods.

---

### How Vertical Scaling Is Performed

Vertical scaling is **manual** and can be done using:

* AWS Management Console (RDS Console)
* AWS CLI (`modify-db-instance`)
* AWS SDK / API

**Console-based steps (high level)**:

1. Select the DB instance
2. Click **Modify**
3. Choose a new DB instance class
4. Apply the change

> **Pre-check Required**: Ensure the target instance class is **available in the region** and **supported by your Aurora PostgreSQL version**.

---

### Availability Impact During Vertical Scaling

* During modification, the **instance becomes unavailable**
* For **single-instance clusters**:

  * Applications experience **downtime**
* For clusters with **at least one reader**:

  * Aurora performs a **failover**
  * Impact on applications is **minimal**

Aurora automatically updates:

* Writer endpoint
* Reader endpoint

Applications do not need endpoint changes.

---

### Key Characteristics of Vertical Scaling

| Aspect              | Behavior                    |
| ------------------- | --------------------------- |
| Manual or Automatic | Manual                      |
| Affects Writer      | Yes                         |
| Downtime Possible   | Yes (reduced with replicas) |
| Cost Control        | Good for seasonal workloads |

---

## 2. Horizontal Scaling (Read Replicas)

### What Is Horizontal Scaling?

Horizontal scaling means **adding or removing Aurora read replicas** to distribute read traffic.

* Improves read throughput
* Does **not** increase writer capacity

Replicas can be added or removed:

* Manually
* Automatically (via auto scaling)

---

### Endpoints and Application Design

* **Writer endpoint** → write traffic
* **Reader endpoint** → read traffic

> Applications **must use the reader endpoint** to benefit from horizontal and automatic scaling.

---

## 3. Automatic Scaling (Aurora Auto Scaling)

### What Is Auto Scaling?

Aurora Auto Scaling automatically:

* Adds replicas (scale out)
* Removes replicas (scale in)

Based on a **scaling policy** attached to the cluster.
<img width="1037" height="223" alt="image" src="https://github.com/user-attachments/assets/91eb6ba3-0ef9-48d5-9c3b-00325f202d75" />
<img width="640" height="510" alt="image" src="https://github.com/user-attachments/assets/4d458214-8c58-43d6-89c4-871e9ebbe69b" />
<img width="708" height="533" alt="image" src="https://github.com/user-attachments/assets/a9c1846f-0d24-4659-bbe2-ae860df70316" />

---

### Prerequisites for Auto Scaling

Auto scaling works only if:

* The cluster has **at least one replica**
* All instances are in **Available** state
* Applications use the **reader endpoint**

---

### Auto Scaling Metrics

Auto scaling policies track a **target metric**.

#### Predefined Metrics

1. **Average CPU utilization across replicas**

   * Example target: 50%
   * Scale out when average CPU > 50%
   * Scale in when average CPU < 50%

2. **Average number of database connections across replicas**

> Although multiple metrics exist, **only one auto scaling policy can be attached to a cluster at a time**.

---

### Scaling Behavior

* Auto scaling attempts to **maintain the target metric value**
* Scaling actions:

  * Scale out → add replicas
  * Scale in → remove replicas

**Replica properties**:

* New replicas use the **same DB instance class as the writer**

---

### Cooldown Periods

Cooldown periods prevent rapid scaling oscillations.

#### Scale-Out Cooldown

* Time (in seconds) to wait **after a scale-out completes** before another scaling action can occur

#### Scale-In Cooldown

* Time (in seconds) to wait **after a scale-in completes** before another scaling action can occur

---

### Replica Deletion Rules

* Auto scaling **only removes replicas created by auto scaling**
* Manually created replicas are **never removed automatically**
* Auto scaling can be **disabled** at any time

If auto scaling is disabled:

* Auto-created replicas must be **removed manually**

---

## Endpoint Behavior During Scaling and Failover

This section explains how **Aurora PostgreSQL endpoints behave during instance scaling and modification**, and how Aurora minimizes application impact.

---

## Aurora Endpoints

Aurora provides **logical cluster-level endpoints** that abstract individual DB instances. Applications connect to these endpoints instead of directly to instances.

| Endpoint Type       | Description                                                 |
| ------------------- | ----------------------------------------------------------- |
| **Writer endpoint** | Routes traffic to the current primary (writer) instance     |
| **Reader endpoint** | Load-balances read traffic across all healthy read replicas |

Using cluster endpoints allows Aurora to handle scaling, failover, and maintenance transparently.

> **Best Practice**: Production applications should always use the cluster writer and reader endpoints, not instance endpoints.

---

## Writer Instance Modification (Scale Up / Scale Down)

When the writer DB instance is scaled up or down, Aurora must briefly take that instance out of service. The resulting behavior depends on the cluster configuration.

---

### Cluster Without Read Replicas

If no read replicas exist:

* The writer instance is taken offline for modification
* No other instance can assume the writer role
* Applications experience an outage for the duration of the change

This is the expected behavior for single-instance Aurora clusters during vertical scaling.

---

### Cluster With One or More Read Replicas

If at least one read replica exists, Aurora performs a **planned promotion** to reduce downtime:

1. A read replica is selected
2. The replica is promoted to become the new writer
3. The original writer is modified in the background

This process is similar to failover but is **planned and controlled**, resulting in minimal application disruption.

---

## Endpoint Updates During Promotion

Aurora automatically maintains endpoint accuracy during scaling operations.

### Writer Endpoint Behavior

* Always resolves to the active writer instance
* Automatically updated when a new writer is promoted
* No application-side configuration changes are required

### Reader Endpoint Behavior

* Routes traffic only to healthy read replicas
* Automatically reflects replica additions, removals, and promotions
* Promoted replicas are removed from the reader pool

---

## Example Flow

**Before scaling**:

```
Writer endpoint → Instance A (writer)
Reader endpoint → Instance B, Instance C (readers)
```

**During scaling**:

```
Instance A → temporarily unavailable
Instance B → promoted to writer
```

**After scaling**:

```
Writer endpoint → Instance B
Reader endpoint → Instance C
```

Applications continue using the same endpoints throughout the process.

---

## Operational Benefits

* Applications are decoupled from instance-level changes
* Scaling and maintenance operations are transparent
* Downtime is minimized when replicas are present

For these reasons, AWS recommends maintaining at least one read replica in production clusters.

---

## Monitoring and Ev

Scaling activities generate **RDS events**, including:

* Instance modification
* Replica creation
* Replica deletion
* Failover events

### Where to View Events

* RDS Console → Cluster → **Logs & Events** tab
* Users can **subscribe to events** via the RDS console

These logs help track:

* When scaling occurred
* Why scaling was triggered
* Impact on cluster availability

---

## Summary

* Aurora supports **vertical, horizontal, and automatic compute scaling**
* **Vertical scaling**:

  * Manual
  * Impacts writer
  * May cause downtime (reduced with replicas)
* **Horizontal scaling**:

  * Adds/removes read replicas
  * Improves read performance
* **Auto scaling**:

  * Policy-based
  * Uses metrics like CPU or connections
  * Requires reader endpoint usage

Upcoming exercises demonstrate:

* Vertical scaling in action
* Auto scaling behavior and policies

---

## Memory Hooks (Quick Recall)

* **One writer only** → vertical scaling matters
* **Seasonal load** → scale up, then scale down
* **Reader endpoint required** → auto scaling works
* **One policy per cluster** → choose metric wisely
* **Auto scaling deletes only its own replicas**

---

## FAQ

### Q1. Does vertical scaling require downtime?

Yes. The instance becomes unavailable during modification. With at least one replica, Aurora performs a failover to reduce impact.

---

### Q2. Can Aurora auto scale the writer instance?

No. Auto scaling applies **only to read replicas**, not the writer.

---

### Q3. Can I attach multiple auto scaling policies?

No. Only **one auto scaling policy** can be attached to a cluster at a time.

---

### Q4. Will auto scaling remove replicas I created manually?

No. Auto scaling removes **only replicas created by auto scaling**.

---

### Q5. What happens if applications connect directly to replica endpoints?

Auto scaling benefits may not apply. Applications should use the **cluster reader endpoint**.

---

### Q6. Is vertical scaling automatic?

No. Vertical scaling must be **initiated manually** via console, CLI, or API.
