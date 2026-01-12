# Choosing Between Amazon RDS for PostgreSQL and Aurora PostgreSQL

# RDS PostgreSQL vs Aurora PostgreSQL – Decision Guide

## 1. Quick Decision Table

| Criteria                      | RDS for PostgreSQL                | Aurora PostgreSQL                   | Choose This When…                        |
| ----------------------------- | --------------------------------- | ----------------------------------- | ---------------------------------------- |
| **Max Database Size**         | Up to **64 TB**                   | Up to **128 TB**                    | DB > 64 TB → **Aurora**                  |
| **Max IOPS**                  | ~**80,000 IOPS** (EBS dependent)  | No hard limit (instance dependent)  | IOPS > 80K → **Aurora**                  |
| **Instance Types**            | Wider range (incl. smaller sizes) | Limited but optimized               | Need specific/smaller instance → **RDS** |
| **Read Replicas**             | Up to **5**                       | Up to **15**                        | >5 replicas → **Aurora**                 |
| **Replication Mechanism**     | PostgreSQL streaming replication  | Storage-level replication           | Need ultra-low lag → **Aurora**          |
| **Replica Lag**               | Seconds (up to ~30s worst case)   | < 100 ms                            | Low-latency reads → **Aurora**           |
| **Failover Time**             | 60–120 seconds                    | ~30 seconds                         | Fast failover → **Aurora**               |
| **Post-Failover Performance** | Cache may be cold                 | Warm cache (Cluster Cache Mgmt)     | No perf drop allowed → **Aurora**        |
| **Cross-Region DR**           | Read replicas (manual recovery)   | Global Database (<1s lag)           | Low RPO/RTO → **Aurora**                 |
| **Advanced Features**         | Limited                           | Serverless, Global DB, Cloning, QPM | Need advanced features → **Aurora**      |
| **Cost Predictability**       | High                              | Variable (I/O + storage growth)     | Fixed cost needed → **RDS**              |
| **Migration Flexibility**     | Easy to move to Aurora            | Easy to move back                   | Both                                     |

---

## 2. Step-by-Step Decision Checklist

Use this checklist from top to bottom 👇

### Step 1: Database Size

* ❓ Will the database exceed **64 TB**?

  * ✅ Yes → **Aurora**
  * ❌ No → Continue

---

### Step 2: Performance (IOPS)

* ❓ Do you need **more than 80,000 IOPS**?

  * ✅ Yes → **Aurora**
  * ❌ No → Continue

---

### Step 3: Read Scaling

* ❓ Do you need **more than 5 read replicas**?

  * ✅ Yes → **Aurora**
  * ❌ No → Continue

* ❓ Do you need **replica lag < 100 ms**?

  * ✅ Yes → **Aurora**
  * ❌ No → Continue

---

### Step 4: High Availability & Failover

* ❓ Must failover complete in **< 30 seconds**?

  * ✅ Yes → **Aurora**
  * ❌ No → Continue

* ❓ Is **any performance degradation after failover unacceptable**?

  * ✅ Yes → **Aurora (Cluster Cache Management)**
  * ❌ No → Continue

---

### Step 5: Disaster Recovery

* ❓ Do you need **very low RPO/RTO for region-wide failures**?

  * ✅ Yes → **Aurora Global Database**
  * ❌ No → Continue

* ❓ Are **manual DR steps acceptable**?

  * ✅ Yes → **RDS**
  * ❌ No → **Aurora**

---

### Step 6: Feature Requirements

* ❓ Do you need any of the following?

  * Global Databases

  * Serverless scaling

  * Fast database cloning

  * Query Plan Management

  * Cluster cache warm failover

  * ✅ Yes → **Aurora**

  * ❌ No → Continue

---

### Step 7: Cost Considerations

* ❓ Do you require **fixed and predictable database costs**?

  * ✅ Yes → **RDS PostgreSQL**
  * ❌ No → **Aurora PostgreSQL**

---

## 3. Simple Rule of Thumb

* **Choose RDS PostgreSQL when:**

  * Workload is moderate
  * Cost predictability matters
  * Fewer replicas are sufficient
  * Slightly slower failover is acceptable

* **Choose Aurora PostgreSQL when:**

  * High scale or high performance is required
  * Very low replica lag is needed
  * Fast failover is critical
  * Global disaster recovery is required
  * Advanced database features add value

---

## 4. Final Note

* Moving **from RDS PostgreSQL to Aurora** (or back) is **straightforward**
* Start with what fits your **current needs**
* Optimize later as your workload evolves

----

Once you decide to use a **managed PostgreSQL service on AWS**, the next key decision is whether to use:

* **Amazon RDS for PostgreSQL**
* **Amazon Aurora PostgreSQL**

This decision should be based on:

* Workload characteristics
* Scaling requirements
* High availability and disaster recovery needs
* Feature requirements
* Cost considerations

---

## 1. Workload Characteristics

### Database Size

* **RDS for PostgreSQL** supports databases up to **64 TB**
* **Aurora PostgreSQL** supports databases up to **128 TB**

**Guideline:**

* If your database is expected to grow beyond **64 TB**, choose **Aurora**
* Otherwise, both options are viable

---

### IOPS Requirements

* **RDS for PostgreSQL**

  * IOPS depend on the EBS volume type
  * Maximum achievable IOPS is approximately **80,000**
* **Aurora PostgreSQL**

  * No explicit IOPS limit at the storage layer
  * Performance scales with instance type

**Guideline:**

* If you need **more than 80,000 IOPS**, choose **Aurora**
* Otherwise, **RDS for PostgreSQL** may be sufficient

---

## 2. Compute and Instance Sizing

Both services require selecting instance sizes based on:

* CPU requirements
* Memory requirements
* Query processing needs

However:

* **RDS for PostgreSQL** offers a **wider range of instance types**
* Some **smaller or specialized instance types** may only be available on RDS

**Guideline:**

* If your workload needs a **very specific or smaller instance type** not available in Aurora, choose **RDS**
* Otherwise, both services work equally well

---

## 3. Read Scaling Requirements

### Number of Read Replicas

* **RDS for PostgreSQL**: up to **5 read replicas**
* **Aurora PostgreSQL**: up to **15 read replicas**

**Guideline:**

* If you need **more than 5 read replicas**, choose **Aurora**

---

### Replication Lag

* **RDS for PostgreSQL**

  * Uses asynchronous PostgreSQL streaming replication
  * Replication lag can be **seconds (up to ~30 seconds in worst cases)**
* **Aurora PostgreSQL**

  * Replication happens at the **storage layer**
  * Replication lag is typically **under 100 milliseconds**

**Guideline:**

* If your application requires **very low read replica lag**, choose **Aurora**
* Otherwise, **RDS** is acceptable

---

## 4. Failover and High Availability

### Failover Time

* **RDS for PostgreSQL**

  * Failover typically takes **60–120 seconds**
* **Aurora PostgreSQL**

  * Failover typically completes in **~30 seconds**

**Guideline:**

* If your application requires **failover in under 30 seconds**, choose **Aurora**

---

### Performance After Failover

* In **RDS**, failover may lead to **performance degradation** due to cold caches
* **Aurora** supports **Cluster Cache Management**

  * Keeps replica buffer cache warm
  * Prevents performance degradation after failover

**Guideline:**

* If **no performance degradation after failover is acceptable**, choose **Aurora with Cluster Cache Management**
* Otherwise, **RDS** is sufficient

---

## 5. Disaster Recovery (Region-Level Failures)

### Cross-Region Replication

* **RDS for PostgreSQL**

  * Supports **cross-region read replicas**
  * Higher replication lag
  * Manual steps often required for recovery
* **Aurora PostgreSQL**

  * Uses **Aurora Global Database**
  * Replication lag is **under 1 second**
  * Very low **RPO and RTO**

**Guideline:**

* If your application requires:

  * **Very low RPO**
  * **Very low RTO**
  * Fast recovery from **region-wide failures**

  → Choose **Aurora**

* If higher RPO is acceptable and **manual disaster recovery steps are okay**, **RDS** may be sufficient

---

## 6. Advanced Aurora-Only Features

Aurora provides several features **not available in RDS**, such as:

* Global Databases
* Serverless
* Cluster Cache Management
* Fast Database Cloning
* Query Plan Management

**Guideline:**

* If these features provide **clear value** to your application, choose **Aurora**
* Otherwise, **RDS for PostgreSQL** is a simpler option

---

## 7. Cost Considerations

Cost is always an important factor.

### RDS for PostgreSQL

* Mostly **fixed and predictable costs**
* Easier cost estimation

### Aurora PostgreSQL

* Has:

  * Fixed costs (instances)
  * Variable costs (storage growth, I/O operations)
* Less predictable than RDS

**Guideline:**

* If **predictable and stable costs** are critical, choose **RDS**
* If you can tolerate **variable costs** in exchange for performance and features, choose **Aurora**

---

## 8. Final Decision Summary

The choice between **RDS for PostgreSQL** and **Aurora PostgreSQL** depends on:

* Database size
* IOPS requirements
* Read scaling needs
* Replication lag tolerance
* Failover speed and performance expectations
* Disaster recovery requirements
* Need for advanced features
* Cost predictability

---

## 9. Switching Between RDS and Aurora

An important point to note:

* **Migrating from RDS PostgreSQL to Aurora PostgreSQL (and vice versa) is straightforward**
* AWS provides tools and supported migration paths

**Bottom line:**

* Choose the option that best fits your **current requirements**
* Switching later is **not difficult**
