# Introduction to Amazon RDS and Amazon Aurora

## 1. Amazon Relational Database Service (RDS)

### 1.1 Self-Managed Database on AWS

* Users can provision a **virtual machine (EC2 instance)** and install/manage the database engine themselves.
* **Responsibilities of the user include:**

  * OS and database installation
  * Patching and upgrades
  * Monitoring, scaling, and backups
  * Security and compliance
  * Failover setup
  * Query optimization
* Users often **build or buy automations** to assist with these tasks.
* Essentially, this setup is similar to an **on-premises database**, but running on AWS infrastructure.

### 1.2 Managed Database with Amazon RDS

* Amazon RDS is a **managed database service**.
* Users provision a database instance via **console, API, or SDK**.
* Amazon RDS handles:

  * Instance provisioning and storage setup
  * Database engine installation and configuration
  * OS and database maintenance tasks (patching, upgrades, backups, failover)
* **Users do not get shell or login access** to the underlying EC2 instance.
* Database tasks are carried out using **RDS automations** through console or SDK.

### 1.3 Key Benefits of RDS

1. **Reduced Operational Burden:** Amazon handles OS and database management.
2. **Built-in Automations:** Users do not need homegrown scripts for high availability, backups, or failover.
3. **Skill Portability:** Standardized automations allow anyone familiar with RDS to manage databases easily.

### 1.4 Supported Database Engines

* Licensed enterprise databases: **Oracle, SQL Server**
* Open-source databases: **MySQL, PostgreSQL, MariaDB**
* Amazon Aurora (covered below)

---

## 2. Amazon Aurora

### 2.1 Overview

* Amazon Aurora is a **PostgreSQL- and MySQL-compatible relational database**.
* Combines **enterprise database performance and availability** with **open-source simplicity and cost-effectiveness**.
* Compatible with existing MySQL or PostgreSQL applications and schemas.

  * Applications can migrate without code or schema changes.

### 2.2 Differences from Community Edition

| Feature            | Community Edition           | Aurora                                                   |
| ------------------ | --------------------------- | -------------------------------------------------------- |
| Storage            | Attached storage volumes    | Distributed storage for higher durability and efficiency |
| Performance        | Depends on attached storage | Offloads storage processing to dedicated storage nodes   |
| Compute Efficiency | Limited to VM compute       | Optimized compute with distributed storage layer         |
| Durability         | Standard replication        | Highly durable, distributed storage architecture         |

### 2.3 Key Benefits

* Enterprise-class database features for MySQL or PostgreSQL workloads
* Can **replace SQL Server or Oracle** in many use cases
* Lower cost compared to traditional enterprise databases
* High performance and availability due to **distributed storage** and compute optimization

---

## 3. Key Points Summary

* **Amazon RDS:** Managed service providing 6 database engine options, including Aurora.
* **Aurora:** Compatible with MySQL and PostgreSQL, built for enterprise applications.
* **Advantages of Aurora:**

  * Enterprise-level performance
  * High durability and availability
  * Simplified migration from MySQL or PostgreSQL
  * Cost-effective alternative to Oracle or SQL Server

