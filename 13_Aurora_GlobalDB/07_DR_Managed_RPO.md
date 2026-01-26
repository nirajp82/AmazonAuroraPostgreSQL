# Aurora PostgreSQL Global Database – Disaster Recovery & Managed RPO

## Overview

This lesson focuses on **disaster recovery (DR)** concepts and how **Amazon Aurora Global Database – Managed RPO** helps applications **minimize data loss across regional failures**.

You will learn:

* Core **DR concepts (RPO & RTO)**
* Why different applications need different DR strategies
* How **Managed RPO** works internally in Aurora Global Database
* Performance and operational trade‑offs
* Monitoring metrics and functions related to Managed RPO

This README is written for **beginners** and intended for **quick revision** later.

---

## Disaster Recovery (DR) – Fundamentals

### What is Disaster Recovery?

**Disaster Recovery** is the process of restoring IT systems after **unplanned events**, such as:

* Database cluster failure
* Entire region failure
* Natural disasters (floods, earthquakes)
* Infrastructure failures (power outage, networking failure)

---

## Two Core Objectives of Any DR Plan (Memory Hook)

**Memory Hook:**

> *DR is always about **how much data you can lose** and **how fast you can recover***

### 1. Recovery Point Objective (RPO)

**Definition:**
The **maximum amount of data loss** an application can tolerate.

**Examples:**

* **E‑commerce application:** RPO = **1 minute** (very sensitive)
* **Reporting application:** RPO = **6 hours** (less sensitive)

Lower RPO ⇒ **higher complexity and cost**

---

### 2. Recovery Time Objective (RTO)

**Definition:**
The **maximum time allowed** to restore the application after a failure.

**Examples:**

* **E‑commerce application:** RTO = **1 minute**
* **Reporting application:** RTO = **24 hours**

Lower RTO ⇒ **faster failover and higher operational cost**

---

## Why RPO and RTO Must Be Defined Per Application

* Different workloads tolerate failure differently
* Helps:

  * Choose correct architecture
  * Control cost
  * Prioritize recovery during incidents

**Key Insight:**

> *There is no one‑size‑fits‑all DR solution.*

---

## Aurora Global Database – Default RPO Limitation

* Out‑of‑the‑box Aurora Global Database typically provides:

  * **~1 minute RPO**

For some applications, this is **not acceptable**.

---

## Why Managed RPO Exists and When to Use It

### The Problem Managed RPO Solves

By default, **Aurora Global Database** provides an RPO of roughly **~1 minute**.
For many workloads this is acceptable, but for **data-sensitive applications**, even 60 seconds of data loss is too much.

**Example:**

* A payment or order-processing system cannot afford to lose more than **20 seconds** of transactions.

This creates a gap between:

* **Business requirement** → very low data loss
* **Default Global DB behavior** → best-effort asynchronous replication

Managed RPO exists to **bridge this gap**.

---

### What Managed RPO Actually Does

**Managed RPO enforces a hard upper bound on data loss** by controlling when transactions are allowed to commit.

It works by:

* Continuously monitoring **replication lag** to secondary clusters
* Allowing a commit **only if the lag is within the configured RPO**

**Supported RPO range:**

* From **20 seconds** up to **68 years**

---

### How Managed RPO Helps Prioritize Recovery Effort

When a regional failure happens:

* Applications with **Managed RPO** are already operating within a known, enforced data-loss window
* This allows teams to:

  * Prioritize **low-RPO applications first**
  * Make faster, more confident failover decisions

**Key idea:**

> *Managed RPO turns an unknown data-loss risk into a controlled, measurable limit.*

---

## How Managed RPO Works (Clear Explanation)

### Example Setup

* Primary cluster → Region 1
* Secondary cluster → Region 2
* Secondary cluster → Region 3
* Managed RPO configured to **20 seconds**

This means:

> The application can lose **no more than 20 seconds of committed data** during a regional failure.

---

### Commit Decision Logic (Step-by-Step)

Aurora evaluates replication lag **at commit time**.

**Rule:**

> A commit is allowed **only if at least one secondary cluster** has lag **less than the configured RPO**.

---

#### Case 1: Both Secondaries Within RPO

* Secondary A lag = 12s
* Secondary B lag = 15s

✅ Commit succeeds

---

#### Case 2: One Secondary Within RPO

* Secondary A lag = 25s
* Secondary B lag = 18s

✅ Commit succeeds (one replica is safe)

---

#### Case 3: All Secondaries Exceed RPO

* Secondary A lag = 22s
* Secondary B lag = 21s

⛔ Commit is **blocked**

Aurora **pauses the application** until:

* At least one secondary recovers to lag < 20s

Then the commit proceeds.

---

## How Managed RPO Works (Step‑by‑Step)

### Example Setup

* Primary cluster → Region 1
* Secondary cluster → Region 2
* Secondary cluster → Region 3
* Managed RPO set to **20 seconds**

---

### Transaction Flow (Memory Hook)

**Memory Hook:**

> *Aurora commits only when **at least one secondary is close enough***

#### Transaction T1

* Secondary A lag = 15s
* Secondary B lag = 10s

✅ Commit allowed

---

#### Transaction T2

* Secondary A lag = 25s
* Secondary B lag = 18s

✅ Commit allowed (one secondary < RPO)

---

#### Transaction T3

* Secondary A lag = 22s
* Secondary B lag = 21s

⛔ Commit blocked

Aurora **pauses the application** until:

* At least one secondary lag < 20s

Then commit is sent.
<img width="1308" height="583" alt="image" src="https://github.com/user-attachments/assets/4d8b495e-80a4-42c4-820c-61d77edd265f" />

---

## Impact of Managed RPO on Applications (Very Important)

### What Blocking Looks Like

* Slower application response times
* Increased latency
* Possible application timeouts

### Risk

* User experience degradation
* Potential availability issues if not tested properly

---

## Best Practice for Using Managed RPO

* Always perform **thorough testing** before production
* Test under:

  * High write load
  * Network latency scenarios
* Balance:

  * Application performance
  * User experience
  * Data loss tolerance

**Key Insight:**

> *Managed RPO is a trade‑off between safety and speed.*

---

## Monitoring Managed RPO

### CloudWatch Metric: Aurora Global DB RPO Lag

**What it measures:**

* Lag (in milliseconds) between primary and secondary
* Calculated **only for user transactions**

---

### Difference: RPO Lag vs Progress Lag

| Metric           | What It Includes           |
| ---------------- | -------------------------- |
| **RPO Lag**      | User transactions only     |
| **Progress Lag** | User + system transactions |

---

## Utility Function for Managed RPO Monitoring

### aurora_global_db_status()

#### Actual Query

```sql
SELECT * FROM aurora_global_db_status();
```
<img width="1235" height="312" alt="image" src="https://github.com/user-attachments/assets/cea000a1-a9be-4908-b8f9-770d27f29c4c" />

**What it provides:**

* Point‑in‑time RPO lag values for each secondary cluster

**Important Note:**

* RPO lag is calculated **only after Managed RPO is enabled**

---

## When Should You Use Managed RPO?

Use Managed RPO when:

* Data loss tolerance is **very low**
* Business impact of lost transactions is high

Avoid or carefully test when:

* Latency‑sensitive workloads
* Applications with aggressive timeout settings

---

## Key Takeaways (Quick Refresh)

* DR planning revolves around **RPO and RTO**
* Different applications require different DR strategies
* Default Global DB RPO may not be sufficient
* Managed RPO enforces stricter data loss guarantees
* It works by **blocking commits** when lag exceeds RPO
* Blocking affects performance and must be tested
* CloudWatch and utility functions help monitor RPO behavior

---

## FAQ

### Q1: Is Managed RPO enabled by default?

No. It must be explicitly enabled using a global parameter.

---

### Q2: Does Managed RPO eliminate all data loss?

No. It **limits** data loss to the configured RPO window.

---

### Q3: Why does Managed RPO slow down my application?

Because commits are **paused** until replication lag meets RPO requirements.

---

### Q4: Is Managed RPO suitable for read‑heavy workloads?

Yes, but write latency impact must still be tested.

---

### Q5: What happens if all secondary clusters exceed RPO?

Application commits are **blocked** until at least one secondary recovers.

---

## Memory Summary (One‑Line Recall)

> *Managed RPO protects data by slowing the app when replicas fall behind.*
