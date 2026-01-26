# Runbook: Create and Test Amazon Aurora PostgreSQL Global Database

## Objective

Create an **Amazon Aurora PostgreSQL Global Database** with:

* A **primary cluster** in Region 1
* A **secondary (read-only) cluster** in Region 2
  Then validate **cross-region replication** by running SQL queries and testing dashboard access from the secondary region.

---

## Architecture Overview

* **Region 1**: Primary Aurora PostgreSQL cluster (read/write)
* **Region 2**: Secondary Aurora PostgreSQL cluster (read-only)
* **Replication**: Asynchronous, managed by Aurora Global Database
* **Use case**: Low-latency reads and disaster recovery

---

## Prerequisites

* AWS account with permissions for:

  * Amazon RDS
  * Aurora
  * VPC and networking
* Two AWS regions selected (Region 1 and Region 2)
* Aurora PostgreSQL engine version that supports **Global Database**
* VPC, subnets, and security groups configured in both regions
* PostgreSQL client (psql, pgAdmin, DBeaver, etc.)

---

## Step 1: Create the Primary Aurora PostgreSQL Cluster (Region 1)

1. Sign in to the **AWS Management Console**
2. Switch to **Region 1**
3. Navigate to **RDS → Databases → Create database**
4. Select **Standard create**
5. Under **Engine options**:

   * Engine type: **Amazon Aurora**
   * Edition: **Aurora PostgreSQL-Compatible**
6. Choose **Production** template
7. Configure the cluster:

   * DB cluster identifier (e.g., `aurora-global-primary`)
   * Master username and password
8. Select DB instance class
9. Configure connectivity:

   * Choose VPC and subnets
   * Assign security group allowing inbound traffic on port **5432**
10. Enable backups and encryption (recommended)
11. Create the database and wait until status is **Available**

---

## Step 2: Create the Global Database and Add Secondary Region

1. In **RDS → Databases**, select the primary cluster
2. Choose **Actions → Add region**
3. Provide:

   * Global database identifier (e.g., `aurora-global-db`)
   * Secondary region (Region 2)
4. Confirm and create

AWS will automatically:

* Provision a secondary Aurora cluster in Region 2
* Configure cross-region replication

Wait until the global database status is **Available**

---

## Step 3: Verify Global Database Configuration

1. Navigate to **RDS → Global databases**
2. Select the global database
3. Confirm:

   * Both regions are listed
   * Primary and secondary roles are correct
   * Replication status is healthy

---

## Step 4: Retrieve Cluster Endpoints

* From **RDS → Databases**, note:

  * **Primary cluster endpoint** (Region 1)
  * **Secondary cluster endpoint** (Region 2)

---

## Step 5: Connect to the Primary Cluster (Region 1)

Use a PostgreSQL client to connect:

```bash
psql -h <primary-endpoint> -U <username> -d <database>
```

---

## Step 6: Run SQL on the Primary Cluster

Create test data to validate replication:

```sql
CREATE TABLE dashboard_test (
    id SERIAL PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO dashboard_test (message)
VALUES ('Data inserted in primary region');
```

---

## Step 7: Connect to the Secondary Cluster (Region 2)

Use the secondary cluster endpoint:

```bash
psql -h <secondary-endpoint> -U <username> -d <database>
```

---

## Step 8: Validate Replication in the Secondary Region

Run a read-only query:

```sql
SELECT * FROM dashboard_test;
```

Expected results:

* Data inserted in Region 1 is visible in Region 2
* Write operations are blocked in the secondary region

---

## Step 9: Dashboard Read Testing (Secondary Region)

1. Configure the dashboard application in Region 2
2. Point the database connection to the **secondary cluster endpoint**
3. Run read-only queries
4. Validate:

   * Low latency
   * Data consistency
   * No write access

---

## (Optional) Step 10: Global Database Failover Test

1. Navigate to **RDS → Global databases**
2. Select the global database
3. Choose **Failover global database**
4. Promote the secondary region to primary (for DR testing)

---

## Expected Outcome

* Global Aurora PostgreSQL database is successfully deployed
* Cross-region replication is confirmed
* Dashboard reads are served from the secondary region
* Architecture supports disaster recovery and global read scaling

---
