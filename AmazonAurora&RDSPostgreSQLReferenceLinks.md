## 1️⃣ HIGH AVAILABILITY / MULTI-AZ (FOUNDATION)

**Why first:** This is the baseline for any production database — availability, fault tolerance, and failover behavior.

**Memory Hook:** Understand failover mechanics and AZ-level resilience before scaling or going global.

1. [Multi-AZ failover concepts](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html)
2. [Aurora high availability & fault tolerance](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html#Aurora.Managing.FaultTolerance)

---

## 2️⃣ LOGICAL REPLICATION (DATA MOVEMENT BASICS)

**Why here:** Once HA is understood, the next step is controlled data replication at table/database level.

**Memory Hook:** Logical replication ≠ cluster replication. WAL, slots, lag, and monitoring matter.

1. [RDS PostgreSQL logical replication features](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.FeatureSupport.LogicalReplication.html)
2. [Understand replication capabilities in Aurora PostgreSQL](https://aws.amazon.com/blogs/database/understand-replication-capabilities-in-amazon-aurora-postgresql/)
3. [Best practices for Amazon RDS PostgreSQL replication](https://aws.amazon.com/blogs/database/best-practices-for-amazon-rds-postgresql-replication/)

---

## 3️⃣ AURORA GLOBAL DATABASE (CROSS-REGION ARCHITECTURE)

**Why here:** Builds on replication knowledge to enable global reads, DR, and region-level resilience.

**Memory Hook:** Global DB = physical replication + fast failover. Always track replication lag.

### Overview & Getting Started

1. [Aurora Global Database – overview](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
2. [Getting started with Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-getting-started.html)
3. [Aurora PostgreSQL Global Database – knowledge center](https://repost.aws/knowledge-center/aurora-postgresql-global-database)
4. [Aurora Global Database feature details](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.html#Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.apg)

### Disaster Recovery Concepts

5. [Choosing a database for disaster recovery](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/choosing-database.html)
6. [Aurora PostgreSQL disaster recovery solutions using AGD](https://aws.amazon.com/blogs/database/aurora-postgresql-disaster-recovery-solutions-using-amazon-aurora-global-database/)
7. [Aurora Global Database disaster recovery](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html)
8. [Cross-region DR using Aurora Global Database](https://aws.amazon.com/blogs/database/cross-region-disaster-recovery-using-amazon-aurora-global-database-for-amazon-aurora-postgresql/)
9. [Disaster recovery strategy using Aurora](https://aws.amazon.com/blogs/database/implementing-a-disaster-recovery-strategy-with-amazon-rds/)

### Failover & Operations

10. [Failover in Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover)
11. [Manual failover for unplanned outages](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover.manual-unplanned)
12. [Managed planned failovers](https://aws.amazon.com/blogs/database/managed-planned-failovers-with-amazon-aurora-global-database/)
13. [Aurora Global Database limitations](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html#aurora-global-database.limitations)

### Region-Specific Example

14. [Disaster recovery setup within India](https://aws.amazon.com/blogs/database/use-amazon-aurora-global-database-to-set-up-disaster-recovery-within-india/)

---

## 4️⃣ WRITE-FORWARDING (GLOBAL WRITES)

**Why here:** Write forwarding is an *extension* of Aurora Global Database.

**Memory Hook:** Enables writes on secondary regions — but with strict constraints.

1. [Write forwarding in Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-write-forwarding.html)
2. [Aurora PostgreSQL write forwarding limitations](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-postgresql-write-forwarding-limitations.html)

---

## 5️⃣ ROLLBACK / MINOR UPGRADES (CHANGE MANAGEMENT)

**Why here:** After architecture is solid, handle upgrades safely.

**Memory Hook:** Minor upgrades can still break things — always plan rollback.

1. [Upgrading RDS PostgreSQL instances – minor versions](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.Minor.html)
2. [Rollback option for Aurora and RDS after minor upgrade or OS patching](https://repost.aws/questions/QUckZZgNvNRruof_94rmgAKw/rollback-option-for-aurora-and-rds-after-minor-upgrade-or-os-patching)
3. [Implementing a rollback strategy using Blue/Green deployments](https://aws.amazon.com/blogs/database/implement-a-rollback-strategy-for-amazon-aurora-postgresql-upgrades-using-amazon-rds-blue-green-deployments/)

---

## 6️⃣ AURORA SERVERLESS v2 (SCALING MODEL)

**Why here:** Choose deployment *after* understanding HA, DR, and replication needs.

**Memory Hook:** Best for spiky workloads, multi-tenant apps, and cost efficiency.

1. [Aurora Serverless v2 documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
2. [Aurora Serverless overview](https://aws.amazon.com/rds/aurora/serverless/)

---

## 7️⃣ TRAINING / VIDEO (REINFORCEMENT)

**Why here:** Reinforces concepts after you understand the theory.

**Memory Hook:** Visual demo for replication, failover, and Aurora internals.

1. [Aurora PostgreSQL Training Video](https://www.youtube.com/watch?v=uZJMrciwBYo)

---

## 8️⃣ PRICING & INSTANCE DETAILS

**Why last:** Cost optimization only makes sense once architecture is final.

**Memory Hook:** Compare Serverless vs provisioned, region pricing, and storage costs.

1. [Aurora PostgreSQL pricing calculator](https://calculator.aws/#/createCalculator/AuroraPostgreSQL)
2. [Aurora pricing (Global)](https://aws.amazon.com/rds/aurora/pricing/)
3. [Aurora pricing (China region)](https://www.amazonaws.cn/en/rds/aurora/pricing/)

