# Aurora PostgreSQL High Availability and Failover

Understand how **Amazon Aurora PostgreSQL** maintains high reliability and availability by:

* Monitoring the health of DB instances
* Automatically detecting failures
* Promoting replicas to become the primary instance
* Recovering from infrastructure and database-level failures
<img width="966" height="492" alt="image" src="https://github.com/user-attachments/assets/b328cc46-dc13-45ad-b3cf-d1067f7f1bb9" />
<img width="1160" height="278" alt="image" src="https://github.com/user-attachments/assets/050d8ebf-67d2-4ca2-ab5a-3eb404faaf03" />

---

## High Availability in Aurora PostgreSQL

Aurora PostgreSQL is designed to provide **high availability (HA)** at the cluster level. Instead of relying on a single database instance, Aurora clusters can contain **multiple DB instances**:

* One **primary (writer)** instance
* One or more **read replica** instances

High availability is achieved by continuously monitoring instance health and automatically recovering from failures.

---

## Continuous Health Monitoring

Aurora continuously checks the health of:

* The **primary (writer) instance**
* All **read replicas**
* The underlying infrastructure supporting the cluster

Health checks allow Aurora to quickly detect failures and initiate recovery actions without manual intervention.

---

## Failure Scenarios

Aurora can detect and recover from several types of failures, including:

* Primary instance becoming **unresponsive**
* **Crash of database engine (DBE) processes** on the primary instance
* **Complete Availability Zone (AZ) failure** hosting the primary instance

These failures trigger Aurora’s automated recovery mechanisms.

---

## Automatic Failover Process

When the **primary instance fails**, Aurora initiates an automatic failover process:

1. Aurora detects the failure through health checks
2. A suitable **read replica** is selected
3. The selected replica is **promoted to become the new primary**
4. Cluster endpoints are updated automatically

This process is referred to as **failover and failure recovery**.

---

## Replica Priority and Promotion

In a **multi-instance cluster**, each replica has a **promotion priority** (also called replica priority or tier).

* Aurora uses this priority to decide **which replica becomes the new primary** during failover
* Replicas with higher priority are promoted first
* <img width="1026" height="598" alt="image" src="https://github.com/user-attachments/assets/3b58b37f-4914-4ba9-b693-e2bcea91a990" />
<img width="582" height="486" alt="image" src="https://github.com/user-attachments/assets/b31beca3-bcac-4bc9-8b0c-d5da5cda8713" />
<img width="1101" height="370" alt="image" src="https://github.com/user-attachments/assets/08b5614f-192a-416a-80ee-80b1f10cde28" />

* If the highest-priority replica is unavailable, Aurora selects the next eligible replica
<img width="1035" height="615" alt="image" src="https://github.com/user-attachments/assets/e555db75-245f-43e6-841a-b743edc73e68" />

This allows controlled and predictable failover behavior.

---

## Single-Instance vs Multi-Instance Clusters

The impact of a failure depends on the cluster configuration.

### Single-Instance Cluster

* No read replicas exist
* Primary instance failure causes a **database outage**
* Recovery requires restarting or replacing the instance
* Application downtime is **longer**
* Recoery may take up to 10 minutes
<img width="1036" height="578" alt="image" src="https://github.com/user-attachments/assets/42484ecb-596e-43ba-b361-6beb06b7b960" />

### Multi-Instance Cluster

* One or more read replicas exist
* Primary failure triggers **automatic failover**
* A replica takes over as primary
* Application downtime is **significantly reduced**
* May take up to 2 mins (But mostly under 60 seconds)

---

## Impact on Applications

Failover is a **disruptive event** for applications:

* Existing database connections are dropped
* Applications must retry connections
* Availability depends on how quickly applications reconnect

The total recovery time depends on:

* Single-instance vs multi-instance cluster
* Application retry logic
* Failover duration

---

## Key Takeaways

* Aurora continuously monitors DB instance health
* Failures are detected automatically
* Replicas are promoted to primary during failover
* Multi-instance clusters provide much higher availability
* Application design affects perceived downtime

---

## Memory Hooks (Quick Recall)

* **HA = replicas + monitoring**
* **Failover = replica promotion**
* **Replica priority decides new primary**
* **Single instance = longer downtime**
* **Multi-instance = fast recovery**

---

## FAQ

### Q1. What is failover in Aurora PostgreSQL?

Failover is the automatic process where Aurora promotes a read replica to become the primary instance when the existing primary fails.

---

### Q2. Does Aurora require manual intervention during failover?

No. Failover and recovery are fully automated.

---

### Q3. Can Aurora recover from an Availability Zone failure?

Yes. If replicas exist in other AZs, Aurora promotes one of them to restore availability.

---

### Q4. Is failover completely transparent to applications?

No. Connections are interrupted, but applications can recover quickly if they use retry logic and cluster endpoints.

---

### Q5. Why is a multi-instance cluster recommended for production?

Because it significantly reduces downtime by enabling automatic failover when the primary instance fails.
