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

This section clarifies how **Aurora endpoints, instance modification, and failover** work together. These concepts are often confusing because they happen simultaneously during scaling.

---

## Key Concepts (Must Understand)

### What Is an Endpoint in Aurora?

An **endpoint** is a **logical DNS name** provided by Aurora. Applications connect to endpoints, not to individual instances.

| Endpoint Type       | What It Always Points To                       |
| ------------------- | ---------------------------------------------- |
| **Writer endpoint** | The current **primary (writer)** instance      |
| **Reader endpoint** | Load-balanced set of **healthy read replicas** |

> **Best Practice**: Applications should never connect directly to instance endpoints in production.

---

## What Happens During Writer Instance Modification (Scale Up / Down)?

When the **writer DB instance** is modified, Aurora must temporarily make that instance unavailable.

Aurora behavior depends on whether **read replicas exist**.

---

### Case 1: No Read Replica Exists

* Writer instance is taken offline for modification
* No instance is available to take over
* ❌ **Application outage occurs**

This is why single-instance clusters experience downtime during vertical scaling.

---

### Case 2: At Least One Read Replica Exists

Aurora performs a **planned failover-style promotion**.

What happens internally:

1. Aurora selects a **read replica**
2. Promotes it to become the **new writer**
3. Modifies the old writer instance in the background

This process minimizes application impact.

---

## Clarifying the Commonly Confusing Statements

### “Aurora automatically adjusts the writer endpoint”

**Meaning**:

* Writer endpoint always points to the **active writer**
* If the writer changes, Aurora updates DNS automatically

**Why it matters**:

* No application configuration changes required
* Connection strings remain the same

---

### “Aurora automatically adjusts the reader endpoint”

**Meaning**:

* Reader endpoint always includes **only healthy replicas**
* Replicas added, removed, or promoted are automatically reflected

**Example**:

* Two replicas → reader endpoint load-balances
* One replica promoted → removed from reader pool
* New replica added → automatically included

---

### “A reader may be promoted during modification”

This refers to a **planned promotion**, not a failure.

* Promotion mechanism is similar to crash failover
* Triggered intentionally during scaling or maintenance

---

## Visual Flow

### Before Scaling

```
Writer endpoint → Instance A (writer)
Reader endpoint → Instance B, Instance C (readers)
```

### During Scaling

```
Instance A → temporarily unavailable
Aurora promotes Instance B → new writer
```

### After Scaling

```
Writer endpoint → Instance B
Reader endpoint → Instance C
```

✔ Endpoints unchanged
✔ Minimal downtime
✔ Transparent to applications

---

## Why This Architecture Matters

* Applications are **decoupled from instance-level changes**
* Scaling, maintenance, and failover are transparent
* AWS strongly recommends:

  * Always having **at least one read replica**
  * Always using **cluster endpoints**

---

## One-Line Memory Hook

> **Aurora endpoints are smart DNS names that always point to the correct instance, even when the writer changes.**

---

## Monitoring and Events

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
