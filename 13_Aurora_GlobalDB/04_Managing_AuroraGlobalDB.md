# Amazon Aurora PostgreSQL – Global Database: Cluster Configuration, Deletion, and Failover

## Overview

This lesson covers:

1. Cluster configuration in a Global Database
2. Deleting a Global Database
3. Failover strategies (planned and unplanned)

Amazon Aurora Global Database allows you to span multiple regions, providing high availability and disaster recovery. However, **failover and deletion require careful handling**, unlike local Aurora clusters.

**Memory Hook:** Think of the global database like a “multi-region team”—all members (clusters) must be coordinated to avoid chaos.

---

## 1. Cluster Configuration

* **Independent management:** Each cluster can have its own database parameters.
* **Best practice:** Use **consistent parameters across all clusters**.

  **Why:** Secondary clusters act as failover targets. If configurations differ, failover behavior may be **inconsistent** with the original primary.

**Memory Hook:** Consistency = predictable failover behavior.

---

## 2. Deleting a Global Database

### Key Steps

1. **Detach clusters**: Remove each cluster from the Global Database individually.

   * Detached clusters become **standalone Aurora clusters**.
   * Data replication from the primary stops.
2. **Delete clusters**: If a standalone cluster is no longer needed, delete all instances.
3. **Delete Global Database**: Only after all clusters are removed and deleted.

**Memory Hook:** Think of it as “disbanding a team”: each member (cluster) must leave individually before the team (global database) is officially gone.

---

## 3. Failover in Global Database

Aurora Global Database **does not automatically failover** like a local cluster. Two types of failovers are supported:

### a) Managed (Planned) Failover

* Initiated by the user (e.g., for maintenance).
* Moves the primary role to another region.
* Failover takes ~1 minute.
* Aurora ensures **full data synchronization** before the switch.
* **Zero data loss.**

**Memory Hook:** Planned failover = careful, pre-coordinated move.

---

### b) Unplanned Failover (Detach and Promote)

* Used when the primary cluster fails unexpectedly.
* Requires **recreating the Global Database**.

#### Steps:
<img width="1706" height="546" alt="image" src="https://github.com/user-attachments/assets/284ea873-4fea-4f99-9db1-d81dfa7d1aae" />

1. **Stop DML operations**: Pause all write operations to avoid inconsistencies.
   <img width="1691" height="518" alt="image" src="https://github.com/user-attachments/assets/4142787b-8edf-496b-b019-b0c0cf240abd" />
3. **Detach all secondary clusters**: Each becomes a standalone cluster. (In this case it would be 3 indepedent cluster) 
   <img width="1707" height="524" alt="image" src="https://github.com/user-attachments/assets/3bc72cf5-c41e-4f64-92a3-14a337971a2b" />
   * This prevents a **split-brain scenario** (applications writing to different clusters, causing data inconsistencies or loss).
4. **Point applications to one standalone cluster**: Preferably the one with the **lowest replication lag**.
    <img width="1740" height="539" alt="image" src="https://github.com/user-attachments/assets/e5a68129-71f3-4d29-85d5-c0d3bad28aed" />
6. **Delete unused clusters**: They still consume resources and cost money.
    <img width="1020" height="427" alt="image" src="https://github.com/user-attachments/assets/008aed0f-c418-42ad-92a4-ed4fdb4a4074" />
8. **Recreate the Global Database**: Add clusters back, possibly with the same names to minimize application reconfiguration.
    <img width="1024" height="421" alt="image" src="https://github.com/user-attachments/assets/215ea17f-f1ac-48b5-9ff6-679c963b1cd2" />
9. **Recover lost data** (if any) and perform **managed failover** to preferred region if needed.

**Memory Hook:** Unplanned failover = emergency evacuation; pick the safest cluster first.
---

## 4. Best Practices

* **Consistency:** Keep configuration parameters identical across clusters.
  This ensures **consistent behavior of the primary cluster before and after a failover or when deleting the Global Database**.

* **Proper Deletion Order:**
  When deleting a Global Database, **each individual cluster must first be detached from the Global Database and then deleted individually**.
  The Global Database itself cannot be deleted until all member clusters are detached and removed.

* **Redundancy:** Maintain **at least 2 secondary clusters** to improve resilience and regional fault tolerance.

* **Lag Awareness:** Route applications to the secondary cluster with the **smallest replication lag**, especially during unplanned failover scenarios.

* **Recovery Playbook:** Maintain a documented recovery playbook for unplanned events.
  The playbook should cover **both database recovery steps and application-level actions** (endpoint updates, DNS changes, validation checks).

**Memory Hook:** *Consistent configs, clean detach-before-delete, and a rehearsed playbook keep global failures predictable.*

---

## 5. Key Points Summary

* Cluster parameters should be consistent for predictable failover.
* Global Database deletion requires detaching and deleting clusters individually.
* Aurora Global Database **does not support automatic failover**.
* Failover types: **managed (planned)** and **unplanned (detach and promote)**.
* Always prepare a **recovery playbook** for emergency events.

---

## FAQ

**Q1: Can secondary clusters have different parameters from the primary?**
**A:** Yes, but it is **not recommended**. Different parameters may cause inconsistent behavior during failover.

**Q2: Does Aurora Global Database failover automatically like a local cluster?**
**A:** No. Failover must be initiated manually (planned) or handled via unplanned recovery steps.

**Q3: What happens when a cluster is detached from the Global Database?**
**A:** It becomes a **standalone cluster**, stops receiving replicated data from the primary, and can be used independently.

**Q4: Why recreate the Global Database after unplanned failover?**
**A:** To restore multi-region replication and ensure the global topology is intact, avoiding split-brain issues.

**Q5: How to minimize data loss in unplanned failover?**
**A:** Point applications to the standalone cluster with **lowest replication lag** and use proper recovery procedures.
