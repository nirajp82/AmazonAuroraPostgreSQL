# Runbook: Create and Test Amazon Aurora PostgreSQL Global Database

## Objective

Create an **Amazon Aurora PostgreSQL Global Database** with:

* A **primary cluster** in Region 1 (read/write)
* A **secondary cluster** in Region 2 (read-only)

Validate **cross-region replication**, confirm **read-only behavior** in the secondary region, and test dashboard access.

---

## Architecture Overview

* **Region 1**: Primary Aurora PostgreSQL cluster
* **Region 2**: Secondary Aurora PostgreSQL cluster (standby / read-only)
* **Replication**: Asynchronous, managed by Aurora Global Database
* **Use case**: Global reads and disaster recovery

---

## Prerequisites

* AWS account with permissions for RDS, Aurora, and VPC
* Two AWS regions selected (Region 1 and Region 2)
* Aurora PostgreSQL version that supports **Global Database**
* VPC, subnets, and security groups configured in both regions
* PostgreSQL client (psql, pgAdmin, DBeaver, etc.)

---

## Step 1: Create the Primary Aurora PostgreSQL Cluster (Region 1)

1. Sign in to the **AWS Management Console**
2. Switch to **Region 1**
3. Navigate to **RDS → Databases → Create database**
4. Select **Standard create**
5. Engine options:

   * Engine type: **Amazon Aurora**
   * Edition: **Aurora PostgreSQL-Compatible**
6. Choose the **Production** template
7. Configure:

   * DB cluster identifier (e.g., `aurora-global-primary`)
   * Master username and password
8. Select DB instance class
9. Configure connectivity:

   * VPC and subnets
   * Security group allowing inbound traffic on port **5432**
10. Enable backups and encryption (recommended)
11. Create the database and wait until the status is **Available**

---

## Step 2: Create the Global Database and Add a Secondary Region

1. In **RDS → Databases**, select the primary cluster
2. Choose **Actions → Add region**
3. Configure:

   * Global database identifier (e.g., `aurora-global-db`)
   * Secondary region (Region 2)
4. Confirm and create

AWS automatically provisions a **read-only Aurora cluster** in Region 2 and configures replication.

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

From **RDS → Databases**, note:

* Primary cluster endpoint (Region 1)
* Secondary cluster endpoint (Region 2)

---

## Step 5: Connect to the Primary Cluster (Region 1)

```bash
psql -h <primary-endpoint> -U <username> -d <database>
```

---

## Step 6: Create Test Data on the Primary Cluster

```sql
CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO test (message)
VALUES ('Data inserted in the primary cluster');
```

---

## Step 7: Validate Data from the Secondary Cluster (Region 2)

Connect using the secondary endpoint:

```bash
psql -h <secondary-endpoint> -U <username> -d <database>
```

Run a selective query:

```sql
SELECT * FROM test;
```

**Expected result**

* The same data created in the primary cluster is visible in the secondary cluster, confirming successful replication.

---

## Step 8: Verify Read-Only Transaction Mode

### On the Primary Cluster

Run the following command:

```sql
SHOW transaction_read_only;
```

**Expected output:**

```
off
```

This confirms the primary cluster allows **read and write operations**.

---

### On the Secondary Cluster

Run the same command:

```sql
SHOW transaction_read_only;
```

**Expected output:**

```
on
```

This confirms:

* The secondary cluster is in **standby mode**
* The cluster is **read-only**

---

## Important Note on Secondary Cluster Behavior

* The secondary region **does not allow DML or DDL operations**
* Statements such as `INSERT`, `UPDATE`, `DELETE`, `CREATE`, or `ALTER` are **not permitted**
* The secondary cluster is intended for:

  * Read-only queries
  * Dashboards
  * Reporting workloads
  * Disaster recovery readiness

Write operations must always be executed on the **primary cluster**.

---

## Step 9: Dashboard Read Testing (Secondary Region)

1. Deploy or access the dashboard application in Region 2
2. Configure the database connection to use the **secondary cluster endpoint**
3. Run read-only queries
4. Validate:

   * Data consistency with the primary cluster
   * Low-latency read performance
   * No write access is possible

---

## (Optional) Step 10: Global Database Failover Test

1. Navigate to **RDS → Global databases**
2. Select the global database
3. Choose **Failover global database**
4. Promote the secondary region to primary (for DR testing)

---

## Expected Outcome

* Aurora PostgreSQL Global Database is successfully configured
* Data replication between regions is verified
* Read-only enforcement in the secondary cluster is confirmed
* Dashboard workloads operate correctly in the secondary region
* Architecture supports global read scaling and disaster recovery

