# Amazon Aurora Storage Architecture – Overview

In this lesson, we will cover how **Aurora storage works**, including **storage nodes, protection groups**, and **fault tolerance mechanisms**. We’ll also explain how Aurora achieves **high availability, scalability, and efficiency** in its storage layer.

The database engine will propagates logs to the backend storage, and the backend storage utilizes a quorum based synchronization scheme to ensure the consistency of the database.

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
<img width="1012" height="564" alt="image" src="https://github.com/user-attachments/assets/32a84e48-251a-4eec-8371-c0db8ea79f8a" />

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
    
<img width="1008" height="596" alt="image" src="https://github.com/user-attachments/assets/2f20da4a-ca41-4255-bccf-f7397073d4c7" />

---

## 4. Protection Groups

* Storage is allocated in **10 GB logical blocks**, called **protection groups**.
* **Data in a protection group** is replicated across **6 nodes in 3 Availability Zones**:

  * 2 copies per AZ
* Nodes in a protection group are called **peers**.
* **Peer-to-peer gossip protocol** ensures that all nodes are up-to-date:

  * A lagging node can fetch missing data from peers
  * A recovering node can synchronize its data from peers
<img width="1015" height="567" alt="image" src="https://github.com/user-attachments/assets/8e6b2731-e1dd-4acb-8f47-65e0dda73aaa" />

---

## 5. Fault Tolerance & Self-Healing

* Aurora continuously monitors **node health**:

  * Not just for outright failure, but also for **performance degradation**
* If a node shows abnormal behavior:

  * Aurora **proactively replaces it** with a healthy node
  * Data is copied from peers
  * **No impact on database availability or performance**
* This gives Aurora **self-healing storage** capabilities.
<img width="1021" height="602" alt="image" src="https://github.com/user-attachments/assets/09ec6ad7-490a-4f58-a68a-069e9845d98a" />

---

## 6. Storage Scaling

* Aurora storage **auto-scales** in **10 GB blocks**:

  * Starts with a **first protection group**
  * Adds additional protection groups as the database grows
* Maximum storage per Aurora cluster: **128 TB**
* A collection of protection groups is referred to as the **Aurora cluster volume**.
<img width="1098" height="573" alt="image" src="https://github.com/user-attachments/assets/44c31fe5-807f-4345-9003-629a3bd92b92" />

---

## 7. Multi-Touch & Multi-Tenant Features

* **Multi-touch:**

  * Multiple compute instances (up to **16**) can attach to the same Aurora storage volume
  * No storage-level replication needed
 
    <img width="997" height="513" alt="image" src="https://github.com/user-attachments/assets/73363aaa-1870-4bb1-8073-4a5e77a23a10" />

* **Multi-tenant:**

  * Storage nodes are shared across **multiple independent Aurora clusters**
  * Aurora ensures **data isolation** between clusters
  * Metadata (storage allocation mappings) is managed by Aurora and is **invisible to users**
  * Capacity is **dynamically managed** by AWS
<img width="1009" height="550" alt="image" src="https://github.com/user-attachments/assets/1f2ed268-46ab-433c-a54e-db5bd5247d73" />

---

## 8. Key Takeaways

* Aurora **does not use traditional attached storage**.
* Storage consists of **thousands of nodes** across multiple AZs.
* Nodes are **self-healing** and continuously monitored.
* **Protection groups** provide redundancy and fault tolerance.
* Storage **auto-scales** seamlessly with database growth.
* Up to **16 compute instances** can share the storage volume.
* Aurora storage is **multi-tenant**, providing shared infrastructure without sacrificing data isolation.
