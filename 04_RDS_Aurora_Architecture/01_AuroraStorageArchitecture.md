# Amazon Aurora Storage Architecture – Overview

In this lesson, we will cover how **Aurora storage works**, including **storage nodes, protection groups**, and **fault tolerance mechanisms**. We’ll also explain how Aurora achieves **high availability, scalability, and efficiency** in its storage layer.

---

## 1. Traditional Database Storage vs Aurora

* Traditional relational databases rely on **attached instance storage** or **networked storage** for persistence.
* The main performance bottleneck is **I/O operations (IOPS)**.
* Performance can be improved by:

  1. **Reducing the number of I/O operations**
  2. **Increasing I/O bandwidth**
* Aurora targets these challenges by **re-architecting the storage layer** from the ground up.
<img width="892" height="603" alt="image" src="https://github.com/user-attachments/assets/554e383f-7289-41df-8f0f-694a2c845d03" />

---

## 2. Aurora Storage Overview

* **Aurora storage** is a **distributed, shared, and multitenant storage system**.
* The storage layer is composed of **thousands of Aurora storage nodes** spread across **multiple Availability Zones (AZs)**.
* **Database compute instances** connect to this storage over the network.

  * Network bandwidth is critical, as heavy write traffic directly impacts performance.

---

## 3. Storage Node Details

* Each node contains a **high-performance SSD**.
* Unlike traditional block storage, a node is **database-aware**:

  * Understands **PostgreSQL pages and blocks**
  * Can **apply WAL (Write-Ahead Logging) records** directly
* **Benefits:**

  * Offloads storage processing from compute instances
  * More efficient use of compute resources
  * Reduced latency for database operations

---

## 4. Protection Groups

* Storage is allocated in **10 GB logical blocks**, called **protection groups**.
* **Data in a protection group** is replicated across **6 nodes in 3 Availability Zones**:

  * 2 copies per AZ
* Nodes in a protection group are called **peers**.
* **Peer-to-peer gossip protocol** ensures that all nodes are up-to-date:

  * A lagging node can fetch missing data from peers
  * A recovering node can synchronize its data from peers

---

## 5. Fault Tolerance & Self-Healing

* Aurora continuously monitors **node health**:

  * Not just for outright failure, but also for **performance degradation**
* If a node shows abnormal behavior:

  * Aurora **proactively replaces it** with a healthy node
  * Data is copied from peers
  * **No impact on database availability or performance**
* This gives Aurora **self-healing storage** capabilities.

---

## 6. Storage Scaling

* Aurora storage **auto-scales** in **10 GB blocks**:

  * Starts with a **first protection group**
  * Adds additional protection groups as the database grows
* Maximum storage per Aurora cluster: **128 TB**
* A collection of protection groups is referred to as the **Aurora cluster volume**.

---

## 7. Multi-Touch & Multi-Tenant Features

* **Multi-touch:**

  * Multiple compute instances (up to **16**) can attach to the same Aurora storage volume
  * No storage-level replication needed
* **Multi-tenant:**

  * Storage nodes are shared across **multiple independent Aurora clusters**
  * Aurora ensures **data isolation** between clusters
  * Metadata (storage allocation mappings) is managed by Aurora and is **invisible to users**
  * Capacity is **dynamically managed** by AWS

---

## 8. Key Takeaways

* Aurora **does not use traditional attached storage**.
* Storage consists of **thousands of nodes** across multiple AZs.
* Nodes are **self-healing** and continuously monitored.
* **Protection groups** provide redundancy and fault tolerance.
* Storage **auto-scales** seamlessly with database growth.
* Up to **16 compute instances** can share the storage volume.
* Aurora storage is **multi-tenant**, providing shared infrastructure without sacrificing data isolation.
