# Amazon Aurora & RDS PostgreSQL Reference Links

---

## A. ROLLBACK / MINOR UPGRADES

**Memory Hook:** Use these when planning minor upgrades or rollback strategies for Aurora/RDS PostgreSQL. Helps avoid downtime and data issues.

1. [Rollback option for Aurora and RDS after minor upgrade or OS patching](https://repost.aws/questions/QUckZZgNvNRruof_94rmgAKw/rollback-option-for-aurora-and-rds-after-minor-upgrade-or-os-patching)
2. [Upgrading RDS PostgreSQL instances – minor versions](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.Minor.html)
3. [Implementing a rollback strategy using Blue/Green deployments](https://aws.amazon.com/blogs/database/implement-a-rollback-strategy-for-amazon-aurora-postgresql-upgrades-using-amazon-rds-blue-green-deployments/)

---

## B. LOGICAL REPLICATION

**Memory Hook:** For replicating tables or databases, not entire clusters. Understand WAL, replication lag, and monitoring.

4. [RDS PostgreSQL logical replication features](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.FeatureSupport.LogicalReplication.html)
5. [Understand replication capabilities in Aurora PostgreSQL](https://aws.amazon.com/blogs/database/understand-replication-capabilities-in-amazon-aurora-postgresql/)
6. [Best practices for Amazon RDS PostgreSQL replication](https://aws.amazon.com/blogs/database/best-practices-for-amazon-rds-postgresql-replication/)

---

## C. PRICING & INSTANCE DETAILS

**Memory Hook:** Use these to calculate costs and compare instance classes, storage types, and Serverless vs provisioned.

7. [Aurora PostgreSQL pricing calculator](https://calculator.aws/#/createCalculator/AuroraPostgreSQL)
8. [Aurora pricing (China region)](https://www.amazonaws.cn/en/rds/aurora/pricing/)
9. [Aurora pricing (Global)](https://aws.amazon.com/rds/aurora/pricing/)

---

## D. WRITE‑FORWARDING (WF)

**Memory Hook:** Use WF to perform writes on secondary regions in Aurora Global Database; be aware of limitations.

10. [Write forwarding in Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-write-forwarding.html)
11. [Aurora PostgreSQL write forwarding limitations](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-postgresql-write-forwarding-limitations.html)

---

## E. AURORA GLOBAL DATABASE (AGD)

**Memory Hook:** Global DB replication for disaster recovery, cross-region reads, and high availability. Monitor replication lag carefully.

12. [Aurora PostgreSQL disaster recovery solutions using AGD](https://aws.amazon.com/blogs/database/aurora-postgresql-disaster-recovery-solutions-using-amazon-aurora-global-database/)
13. [Aurora Global Database – overview](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
14. [Aurora PostgreSQL Global Database – knowledge center](https://repost.aws/knowledge-center/aurora-postgresql-global-database)
15. [Getting started with Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-getting-started.html)
16. [Choosing a database for disaster recovery](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/choosing-database.html)
17. [Aurora Global Database feature details](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.html#Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.apg)
18. [Aurora Global Database limitations](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html#aurora-global-database.limitations)
19. [Disaster recovery strategy using Aurora](https://aws.amazon.com/blogs/database/implementing-a-disaster-recovery-strategy-with-amazon-rds/)
20. [Cross-region DR using Aurora Global Database](https://aws.amazon.com/blogs/database/cross-region-disaster-recovery-using-amazon-aurora-global-database-for-amazon-aurora-postgresql/)
21. [Aurora Global Database disaster recovery](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html)
22. [Failover in Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover)
23. [Manual failover for unplanned outages](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover.manual-unplanned)
24. [Disaster recovery setup in India](https://aws.amazon.com/blogs/database/use-amazon-aurora-global-database-to-set-up-disaster-recovery-within-india/)
25. [Managed planned failovers](https://aws.amazon.com/blogs/database/managed-planned-failovers-with-amazon-aurora-global-database/)

---

## F. AURORA SERVERLESS v2

**Memory Hook:** For variable workloads, fast scaling, multi-tenant apps, and cost optimization.

26. [Aurora Serverless v2 documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
27. [Aurora Serverless overview](https://aws.amazon.com/rds/aurora/serverless/)

---

## G. HIGH AVAILABILITY / MULTI‑AZ

**Memory Hook:** Understand failover, fault tolerance, and multi-AZ deployment options for Aurora/RDS PostgreSQL.

28. [Multi-AZ failover concepts](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html)
29. [Aurora high availability & fault tolerance](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html#Aurora.Managing.FaultTolerance)

---

## H. TRAINING / VIDEO

**Memory Hook:** Practical demo for replication, failover, and Aurora concepts.

30. [Aurora PostgreSQL Training Video](https://www.youtube.com/watch?v=uZJMrciwBYo)
