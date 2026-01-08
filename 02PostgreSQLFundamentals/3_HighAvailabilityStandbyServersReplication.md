# PostgreSQL High Availability and Replication Notes

## 1. High Availability Overview

High availability in PostgreSQL is achieved by maintaining **standby servers** that can take over if the **primary server** fails.

* Applications read and write to the **primary server**.
* A **standby server** maintains a copy of the primary’s data to allow quick recovery in case of failure.
* **Challenge:** Keeping the primary and standby in sync in real-time or near real-time.
* **Failover:** When a primary server fails, the standby is promoted to become the primary. The application then reconnects to the standby with minimal downtime.

**Key responsibilities:**

* **DB Administrators:** Set up replication and failover automation.
* **Application Developers:** Ensure the application can detect failures and reconnect to standby servers automatically.

---

## 2. Types of Standby Servers

Standby servers can be configured in two main ways:

| Standby Type     | Description                                                               | Client Connections                                                       |
| ---------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Warm Standby** | Standby is **dedicated for failover** only.                               | Clients **cannot connect** until it is promoted to primary.              |
| **Hot Standby**  | Standby allows **read-only queries** while maintaining sync with primary. | Clients **can connect** for read operations. **Writes are not allowed.** |

---

## 3. Replication in PostgreSQL

**Replication** is the process of keeping data in sync between primary and standby servers.

**Purpose:**

* Keep standby up-to-date for failover.
* Enable read scaling (for hot standby).

**Common replication methods:**

### 3.1 Log Shipping Replication

* WAL (Write-Ahead Log) segments of **16 MB each** are continuously sent from primary to standby.
* Standby applies WAL segments to remain in sync.
* **Lag:** Standby may lag behind primary because WAL segments must be filled before shipping.
<img width="1469" height="850" alt="image" src="https://github.com/user-attachments/assets/f66c82f5-dcc4-466d-9ca3-a30ee2afacb5" />

### 3.2 Streaming Replication

* WAL records are sent **immediately** to the standby as they are generated.
* Reduces replication lag compared to log shipping.
* By default, **asynchronous**: primary does not wait for standby to commit.
* **Replication lag:** Time between transaction commit on primary and it appearing on standby.
<img width="1591" height="861" alt="image" src="https://github.com/user-attachments/assets/0e0c6f4b-65fe-41c8-b2ba-288b72025102" />


**Factors affecting replication lag:**

* Network latency
* Heavy transaction volume
* Misconfiguration of primary/standby
* Hardware differences between primary and standby

**Important:** High replication lag increases the risk of data loss in case of failure.

---

### 3.3 Synchronous Streaming Replication

* Ensures **zero data loss** at the cost of performance.

* Process:

  1. Client initiates a transaction on primary.
  2. Primary adds WAL record and sends it to standby.
  3. Standby **commits the transaction** and confirms.
  4. Primary sends commit response to client **after confirmation from standby**.

* **Trade-off:** Lower performance due to waiting for standby confirmation.

---

### 3.4 Cascading Replication

* Standby server can act as a **source** to replicate data to another standby.
* Always **asynchronous** between cascading standby and downstream standbys.
* Can be useful for reducing load on primary or extending replication to multiple sites.
* Cascading standby may be promoted to primary without affecting other standbys.

---

### 3.5 Physical Replication

* Byte-level replication of storage devices between primary and standby.
* Mirrors filesystem across servers.
* Provides **better performance** than WAL log shipping.

---

### 3.6 Logical Replication

* Table-level replication using the **publisher-subscriber model**.
* **Publisher:** Sends changes for specific tables.
* **Subscriber:** Receives changes in real-time.
* Also called **transactional replication**.

**Use cases:**

* Incremental data capture
* Consolidating changes from multiple databases
* Controlling access to sensitive data for regulatory compliance

---

## 4. Summary of Replication Methods

| Replication Type  | Level        | Lag                            | Client Access               | Notes                               |
| ----------------- | ------------ | ------------------------------ | --------------------------- | ----------------------------------- |
| Log Shipping      | WAL Segment  | High (depends on segment size) | Standby cannot serve reads  | Simple to configure                 |
| Streaming (Async) | WAL Record   | Low                            | Hot standby can serve reads | Risk of data loss                   |
| Streaming (Sync)  | WAL Record   | Minimal                        | Hot standby can serve reads | Performance penalty, zero data loss |
| Cascading         | WAL Record   | Asynchronous                   | Hot standby                 | Useful for multi-tier replication   |
| Physical          | Storage/Byte | Minimal                        | Usually no direct access    | High performance                    |
| Logical           | Table-level  | Minimal                        | Subscriber read access      | Flexible, multiple use cases        |

---

### Key Points to Remember

* High availability relies on standby servers.
* **Replication lag** is a critical factor in failover planning.
* **Synchronous replication** prevents data loss but reduces performance.
* **Hot standby** allows read scaling; **warm standby** is simpler but only used for failover.
* PostgreSQL provides multiple replication mechanisms to suit different use cases.
