# PostgreSQL Monitoring & Telemetry (Aurora / RDS Context)

## Purpose of Telemetry Data

Databases publish **telemetry data** to support:

* **Monitoring** (real-time health & performance)
* **Troubleshooting / incident investigation**
* **Automation** (auto-scaling, alerts, remediation)
* **Capacity planning & optimization**

This telemetry data is consumed by:

* Humans (DBAs, SREs, developers)
* Automated systems (alerts, scaling policies, dashboards)

---

## Holistic View of Database Performance

Database performance **cannot be evaluated in isolation**.
It depends on multiple interconnected layers:

| Layer            | Why It Matters                      |
| ---------------- | ----------------------------------- |
| Database Engine  | Query execution, caching, locking   |
| Operating System | CPU scheduling, memory management   |
| Virtual Machine  | Resource limits, instance sizing    |
| Network          | Storage & replica communication     |
| Storage          | I/O latency, throughput, durability |
| Queries          | Inefficient SQL, contention         |

👉 **Key idea**: A slow database may not be caused by SQL alone.

---

## Database Engine Metrics

The database engine exposes **engine-level metrics** that reveal performance constraints and optimization opportunities.

### Examples

* **Buffer Cache Hit Ratio**

  * Low ratio indicates frequent disk reads
  * Opportunity: increase buffer cache memory or tune queries

### What Engine Metrics Help With

* Identifying memory pressure
* Detecting CPU-bound workloads
* Understanding concurrency and locking issues

---

### Memory Hook 🧠

**Database engine = brain**
If the brain keeps forgetting things (low cache hit ratio), it keeps “looking them up” on disk → slower responses.

---

## OS & Virtual Machine Metrics

The database engine **runs on top of**:

* An operating system
* A virtual machine (EC2 / RDS instance)

### Why This Matters

A:

* Resource-constrained VM
* Misconfigured OS

can cause:

* High CPU wait
* Memory swapping
* I/O throttling

These issues **surface as database slowness**, even if SQL is perfect.

---

## Network Metrics

In Aurora and RDS:

* **Local EBS volumes**
* **Aurora cluster volumes**

are attached via:

* Network interfaces
* Storage networks

### Impact of Network Issues

* Limited bandwidth
* High latency
* Network congestion

➡ Directly degrades:

* Query latency
* Replication
* Storage access

---
* EBS and Aurora storage are **network-attached**
* Network health directly impacts storage performance

---

## Storage Metrics

Storage is designed for:

* **High availability**
* **High performance**

But users still need visibility into:

* IOPS usage
* Latency
* Throughput
* Utilization

### Why Storage Metrics Matter

* Identify I/O bottlenecks
* Prevent saturation
* Plan scaling decisions

---

## Query-Level Performance Metrics

Application developers often need **query-level visibility** to diagnose:

* Slow queries
* Lock contention
* Inefficient execution plans

### Source of Query-Level Metrics

* PostgreSQL internal statistics
* Performance Insights (Aurora / RDS)

These metrics help:

* Pinpoint problematic SQL
* Understand query wait states
* Optimize execution paths

---

## Metric Consumption & Access

All collected metrics are exposed via:

* **Tools**
* **APIs**
* **Dashboards**

---

## CloudWatch – Central Monitoring Tool

### What CloudWatch Provides

* Central access to metrics across **all layers**
* Long-term metric storage
* Alerting and dashboards

### Important Limitation

❗ CloudWatch **does NOT provide query-level metrics**

Query-level data comes from **Performance Insights**, not CloudWatch.

---

### Enhanced Monitoring

Enhanced Monitoring provides:

* **Highly granular OS-level metrics**
* Higher resolution than standard CloudWatch metrics

📌 **Storage Location**:
Enhanced monitoring metrics are stored in **CloudWatch Logs**

---

* Standard CloudWatch → coarse metrics (1–5 min)
* Enhanced Monitoring → fine-grained OS metrics (seconds)

---

## PostgreSQL Logs

PostgreSQL logs are used for:

* Debugging
* Auditing
* Performance analysis
* Maintenance

### Access Methods

* AWS Console
* Download directly
* Publish logs to CloudWatch Logs (optional)

---

## Aurora / RDS Events

Events indicate **changes in database environment**, such as:

* Failover
* Instance restart
* Shutdown
* Maintenance actions

### Event Handling

* Users can **subscribe** to events
* Events can trigger:

  * Notifications
  * Automation
  * Incident workflows

Events may also be:

* Stored in CloudWatch for long-term retention

---

## Performance Insights

Performance Insights is a **graphical analysis tool** providing:

* Instance-level performance data
* Query-level performance insights

### Use Cases

* Identify top SQL by load
* Detect wait events
* Optimize query performance
* Plan corrective actions

📌 Covered in detail in a later section (as per transcript)

---

## What This Section Covers

Hands-on lessons include:

* CloudWatch metrics
* Enhanced monitoring
* PostgreSQL logs
* Aurora/RDS events

---

## Learning Outcomes

By the end of this section, you should be able to:

* Explain Aurora/RDS monitoring features
* Use CloudWatch metrics and dashboards
* Set up alerts
* Enable Enhanced Monitoring
* Understand:

  * Difference between CloudWatch OS metrics and Enhanced Monitoring
* Access PostgreSQL logs using multiple methods
* Subscribe to Aurora/RDS events

---

## Memory Hook Summary 🧠

**Monitoring Stack Mental Model**

```
Queries → Database Engine → OS → VM → Network → Storage
```

* CloudWatch → Everything *except* query-level
* Performance Insights → Query-level visibility
* Logs → Detailed forensic evidence
* Events → Environment changes

---

## FAQ

### Q1. Does CloudWatch provide query-level metrics?

**No.** Query-level metrics are provided by **Performance Insights**, not CloudWatch.

---

### Q2. Why can a database be slow even when queries are optimized?

Because:

* OS
* VM
* Network
* Storage

may be constrained or misconfigured.

---

### Q3. What is the difference between CloudWatch OS metrics and Enhanced Monitoring?

| Feature     | CloudWatch         | Enhanced Monitoring |
| ----------- | ------------------ | ------------------- |
| Resolution  | Coarse             | Fine-grained        |
| Data Source | Hypervisor         | OS level            |
| Storage     | CloudWatch Metrics | CloudWatch Logs     |

---

### Q4. When should I look at PostgreSQL logs?

* Unexpected errors
* Slow queries
* Maintenance activities
* Debugging application issues

---

### Q5. What are Aurora/RDS events mainly used for?

* Detect failovers
* Track maintenance actions
* Trigger alerts or automation
