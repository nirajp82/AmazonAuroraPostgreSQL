# Amazon Aurora Storage Architecture – Overview

Aurora storage is **distributed, database-aware, self-healing, and highly performant**, designed to handle heavy workloads efficiently.

The database engine propagates **WAL logs** to the storage layer, which uses a **quorum-based synchronization** scheme for consistency.

---

## 1. Traditional Database Storage vs Aurora

* Traditional relational databases rely on **attached instance storage** or **networked storage**.
* Main bottleneck: **I/O operations (IOPS)**.
* Performance improvements:

  1. Reduce I/O operations
  2. Increase I/O bandwidth
* **Aurora re-architects storage from the ground up** for **high availability, durability, and efficiency**.

<img width="892" height="603" alt="image" src="https://github.com/user-attachments/assets/554e383f-7289-41df-8f0f-694a2c845d03" />

---

## 2. Aurora Storage Overview

* Aurora storage is **distributed, shared, and multitenant**.
* Composed of **thousands of storage nodes across multiple AZs**.
* **Database compute instances connect over the network**: network bandwidth is critical.

<img width="1012" height="564" alt="image" src="https://github.com/user-attachments/assets/32a84e48-251a-4eec-8371-c0db8ea79f8a" />

**Memory Hook:**

> “Thousands of nodes, spread across AZs, all talking and healing themselves.”

---

## 3. Storage Node Details

* Each node: **high-performance SSD**
* **Database-aware**: understands **Postgres pages & WAL**, applies log records directly
* **Benefits:**

  * Offloads storage processing from compute instances
  * Reduces latency
  * Improves efficiency

<img width="1008" height="596" alt="image" src="https://github.com/user-attachments/assets/2f20da4a-ca41-4255-bccf-f7397073d4c7" />

**Memory Hook:**

> “Node knows Postgres pages, applies WAL, saves compute”

---

## 4. Protection Groups

* Storage allocated in **10 GB logical blocks** = **protection groups**
* **Replicated across 6 nodes in 3 AZs** (2 copies per AZ)
* Nodes in a group = **peers**
* **Peer-to-peer gossip protocol** ensures synchronization

  * Lagging/recovering nodes fetch missing data from peers

<img width="1015" height="567" alt="image" src="https://github.com/user-attachments/assets/8e6b2731-e1dd-4acb-8f47-65e0dda73aaa" />

**Memory Hook:**

> “10GB blocks, 6 nodes, 3 AZs, 2 copies each”

---

## 5. Fault Tolerance & Self-Healing

* Aurora monitors **node health continuously**:

  * Node failure
  * Performance degradation
* Failing nodes are **proactively replaced**
* Data fetched from peers
* **No impact** on database availability

<img width="1021" height="602" alt="image" src="https://github.com/user-attachments/assets/09ec6ad7-490a-4f58-a68a-069e9845d98a" />

**Memory Hook:**

> “Monitor → Replace → Sync → Zero downtime”

---

## 6. Storage Scaling

* Auto-scaled in **10 GB blocks**
* Starts with first protection group, adds more as needed
* Maximum cluster size: **128 TB**
* Collection of protection groups = **Aurora cluster volume**

<img width="1098" height="573" alt="image" src="https://github.com/user-attachments/assets/44c31fe5-807f-4345-9003-629a3bd92b92" />

**Memory Hook:**

> “Grow by 10GB blocks, up to 128TB”

---

## 7. Multi-Touch & Multi-Tenant Features

**Multi-Touch:**

* Up to **16 compute instances** can attach to the same storage volume
* No storage-level replication needed

<img width="997" height="513" alt="image" src="https://github.com/user-attachments/assets/73363aaa-1870-4bb1-8073-4a5e77a23a10" />

**Multi-Tenant:**

* Storage nodes shared across **multiple clusters**
* Aurora ensures **data isolation**
* Metadata managed internally, **invisible to users**
* Capacity dynamically managed by AWS

<img width="1009" height="550" alt="image" src="https://github.com/user-attachments/assets/1f2ed268-46ab-433c-a54e-db5bd5247d73" />

**Memory Hook:**

> “Many users, many instances, same shared storage, isolated”

---

## 8. Key Takeaways

* Aurora **does not use traditional attached storage**
* Storage consists of **thousands of nodes across multiple AZs**
* Nodes are **self-healing and continuously monitored**
* **Protection groups** provide redundancy and fault tolerance
* Storage **auto-scales** seamlessly
* Up to **16 compute instances** can share storage
* Aurora storage is **multi-tenant**, providing shared infrastructure without sacrificing isolation

---

**Quick Memory Cheat Sheet**

| Feature        | Hook                         | Key Detail                       |
| -------------- | ---------------------------- | -------------------------------- |
| Storage type   | “Distributed & shared”       | Thousands of nodes across AZs    |
| Redundancy     | “10GB, 6 nodes, 3 AZs”       | 2 copies per AZ                  |
| Database-aware | “Node knows WAL”             | Offloads compute, fast I/O       |
| Auto-scaling   | “Grow in 10GB blocks”        | Up to 128 TB                     |
| Multi-touch    | “16 instances share storage” | No storage replication needed    |
| Multi-tenant   | “Shared but isolated”        | Multiple clusters use same nodes |
| Self-healing   | “Monitor → Replace → Sync”   | Zero downtime                    |

