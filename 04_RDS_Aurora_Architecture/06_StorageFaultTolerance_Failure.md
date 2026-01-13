# Amazon Aurora Storage Fault Tolerance and Failure Scenarios

Like any other enterprise database or database cluster, **Aurora must be fault-tolerant**.

In this lesson, we will discuss:

* Various **failure scenarios** in Aurora storage
* The **AZ+1 fault model**
* How to **prepare for prolonged database unavailability**

Aurora’s designers always start with the assumption that **failures will happen all the time**. The goal is to design services that remain operational even when components fail, and Aurora is no exception.

---

## 1. Aurora Failure Assumptions

The foundational assumption for Aurora is:

* **Failures in storage will happen**
* Failures can be:

  1. **Isolated** – a single node becomes unavailable
  2. **Independent simultaneous** – two nodes fail across same or different AZs
  3. **Correlated/widespread** – multiple nodes fail due to an AZ failure

The **objective** for Aurora’s designers is to ensure that the **database remains available** even under these failure scenarios.

---

## 2. Aurora Storage Architecture Recap

* Aurora Storage maintains **6 nodes across 3 Availability Zones (AZs)** in a **protection group**
* Uses **4-of-6 write quorum** for durability
* Uses **3-of-6 read quorum** for recovery reads

> With this design, Aurora can tolerate **up to 3 independent node failures** while remaining available for reads, and **up to 2 node failures** for writes.

Some failures may be **correlated**, such as losing **two nodes due to an AZ failure**, which is part of the **AZ+1 fault model**.

---

## 3. AZ+1 Fault Model

The **AZ+1 model** defines the **maximum fault tolerance** for an Aurora cluster from a storage perspective:

* The database cluster remains available **even if one AZ fails**
* Plus **one additional node fails in the remaining AZs**

> While the probability of this scenario is low, it is **not zero**, so planning is essential.
<img width="1175" height="500" alt="image" src="https://github.com/user-attachments/assets/806ccea0-c930-4ef4-9280-546652dd42a8" />

---

## 4. Common Failure Scenarios

### 4.1 All Nodes Fail Independently

* Nodes may fail in **same or different AZs**
* With **4-of-6 write quorum** and **3-of-6 read quorum**, availability **is not impacted**
* Reads and writes continue normally

### 4.2 Correlated Failures – AZ Failure

* Two nodes may fail simultaneously due to an **AZ outage**
* **Quorum rules still met** → reads and writes continue

### 4.3 Maximum Failure Scenario – AZ + One Node

* One AZ fails completely
* One additional node fails in another AZ
* Only **3 nodes remain active for write quorum** → **writes fail**
* Read quorum (3-of-6) can still be met → **reads continue**

> Beyond this point, the database becomes unavailable

---

## 5. Recovery Considerations

* Recovery of an AZ depends on the **nature of the failure**:

| Failure Type                   | Recovery Time Estimate |
| ------------------------------ | ---------------------- |
| Network cable / hardware loss  | Minutes to an hour     |
| Natural disaster (flood/quake) | Weeks to months        |

* Aurora introduced a **“dictator mode”** to handle **long-term AZ loss** by temporarily **reducing quorum requirements**
* Users are responsible for planning **disaster recovery** beyond AZ+1:

  * **Periodic backups to other regions**
  * **Global Database replication**
  * Mechanism chosen based on **RTO and RPO requirements**

---

## 6. Key Points

* **Reads and writes available** with up to **2 node failures**
* **Reads only** with **3 node failures**
* Maximum tolerable failure = **AZ + 1 node**
* **4-of-6 write quorum**, **3-of-6 recovery read quorum**
* Beyond this, database **becomes unavailable**

---

## 7. Failure Scenarios Table

| Scenario                       | Writes Available | Reads Available | Notes                                     |
| ------------------------------ | ---------------- | --------------- | ----------------------------------------- |
| 1–2 nodes fail independently   | ✅                | ✅               | Quorum rules satisfied                    |
| 3 nodes fail independently     | ❌                | ✅               | Write quorum fails, read quorum still met |
| 2 nodes lost due to AZ failure | ✅                | ✅               | Quorum rules satisfied                    |
| AZ + 1 node fails              | ❌                | ✅               | Writes unavailable, reads continue        |
| Beyond AZ+1                    | ❌                | ❌               | Database unavailable                      |

---

## 8. Memory Hook

> “Aurora survives AZ+1; plan DR beyond that”

**Easy to remember:**

* Node = Worker (applies redo logs)
* Peer = Equal voter (quorum participant)
* Protection Group = Safety team of 6
* Quorum = Minimum votes (4 writes, 3 recovery reads)
* AZ+1 = Maximum tolerable failure

---

## 9. Exam-Style Q&A

1. **How many nodes are in a protection group?** → 6
2. **Write quorum requirement?** → 4-of-6
3. **Recovery read quorum?** → 3-of-6
4. **Maximum tolerable failure scenario?** → AZ + 1 node in other AZ
5. **What happens if 3 nodes fail independently?** → Writes fail, reads succeed
6. **Can the database survive a complete AZ failure?** → Yes, if AZ+1 not exceeded
7. **Who is responsible for disaster recovery beyond AZ+1?** → User/application owner
