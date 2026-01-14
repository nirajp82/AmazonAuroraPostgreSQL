# Aurora PostgreSQL Cluster Cache Management

## Lesson Objective

Understand how **Aurora PostgreSQL cluster cache management** ensures consistent application performance after failover by:

* Managing buffer cache state across primary and replica instances
* Reducing performance impact immediately after failover
* Using intelligent synchronization mechanisms to keep replicas "warm"

---

## Buffer Cache Challenges During Failover

When a **replica is promoted to primary** during failover:

* Its **buffer cache** is not aligned with the previous primary instance
* Pages needed by the workload may not be in cache
* Application queries must read missing pages from storage
* This leads to **increased I/O latency**, longer query response times, and reduced throughput

### Illustration Example

Assume a cluster with **1 primary and 1 replica** and a shared cluster volume with pages 1–9.

* **Primary workload**: Web application accessing recent data
* **Replica workload**: Reporting queries reading older data

| Page | In Primary Cache? | In Replica Cache? |
| ---- | ----------------- | ----------------- |
| 1    | No                | Yes               |
| 2    | No                | Yes               |
| 3    | No                | No                |
| 4    | No                | Yes               |
| 5    | No                | No                |
| 6    | Yes               | No                |
| 7    | No                | No                |
| 8    | Yes               | Yes               |
| 9    | Yes               | No                |

After failover, the new primary (former replica) **lacks pages 6 and 9**, which are needed by the web application. These pages must be read from storage, causing temporary performance degradation.

> In real workloads, the number of pages needing to be loaded may be **hundreds of thousands**, magnifying the performance impact.

---

## Buffer Cache Warm-Up

* **Buffer cache warm-up** is the process of loading frequently accessed pages into the cache of the new primary
* During this time:

  * Application response time may increase
  * Transaction throughput may drop
* Eventually, the cache is fully populated, restoring normal performance

> The duration and impact depend on workload size and query patterns.

---

## Cluster Cache Management Feature

Aurora PostgreSQL provides **Cluster Cache Management (CCM)** to address this issue.

### How CCM Works

1. **Replica targeted for failover** continuously tracks the primary's buffer cache
2. The **replica informs the primary** of its cache contents periodically
3. Primary responds with the list of pages the replica needs to keep its cache in sync
4. Replica pre-loads these pages into its cache using **Bloom filters** for efficient communication

### Example of CCM in Action

* Primary has pages 6, 8, 9 in cache
* Replica has pages 1, 2, 4, 8
* Replica sends Bloom filter to primary indicating cached pages
* Primary responds with missing pages 6 and 9
* Replica loads these pages into its cache
* Subsequent pages (10, 13, etc.) are also synchronized as workloads evolve

**Result:** Replica is kept in **lockstep** with primary, minimizing performance impact after failover.

---

## Operational Benefits

* Failover completes in under a minute, **independent of CCM**
* CCM ensures **minimal impact on transaction throughput** after failover
* Web applications and other workloads experience near-continuous performance
* Benchmarking applications is recommended to evaluate CCM impact

### Observed Performance Impact

* Without CCM: Post-failover transaction rate drops until cache is rebuilt
* With CCM: Post-failover transaction rate remains steady

---

## Key Takeaways

* Buffer cache state varies between primary and replicas due to different workloads
* Failover without CCM can lead to **high I/O latency and reduced throughput**
* **Cluster Cache Management** keeps replicas' buffer cache synchronized with primary
* Enables **fast recovery of application performance** after failover
* Duration of failover is unaffected by CCM, but CCM reduces post-failover performance impact

---

## Memory Hooks (Quick Recall)

* **Failover = new primary may have cold cache**
* **Buffer cache warm-up** = temporary performance drop
* **CCM enabled** = replica cache pre-warmed to match primary
* **Bloom filter** used for efficient cache synchronization
* Benchmark applications to gauge post-failover performance

---

## FAQ

### Q1. Does CCM affect failover time?

No. Failover duration remains under a minute. CCM only impacts **post-failover performance**, not the failover speed.

---

### Q2. How does CCM synchronize cache?

Replicas track their cached pages, send Bloom filters to primary, which responds with missing pages. Replica pre-loads these pages into cache.

---

### Q3. Can CCM eliminate all performance impact?

It minimizes the impact but cannot eliminate it entirely. The effect depends on workload, query patterns, and cluster size.

---

### Q4. Is CCM automatic or needs manual enablement?

CCM must be **enabled on the cluster**. Once enabled, it automatically keeps targeted replicas' cache in sync.

---

### Q5. Should all clusters enable CCM?

Recommended for **performance-sensitive applications** where post-failover latency is critical.

