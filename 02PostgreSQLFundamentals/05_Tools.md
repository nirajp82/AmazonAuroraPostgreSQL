# PostgreSQL Tools, Configuration, and System Catalog

This document summarizes the **tools**, **configuration management**, and **system catalog** in PostgreSQL.

---

## 1. PostgreSQL Client Tools

PostgreSQL provides several tools for interacting with the database:

### 1.1 `psql` Client

* **Description:** Command-line tool for interacting with PostgreSQL using SQL statements.
* **Common Use:** Database administrators use it for administrative tasks and running queries interactively.

**Key points:**

* Terminal-based interface.
* Can execute SQL statements, manage databases, and check server status.

---

### 1.2 pgAdmin

* **Description:** Web-based graphical management tool for PostgreSQL.
* **Common Use:**

  * For developers and DBAs to run SQL queries and commands via a GUI.
  * Provides visual management of databases, users, and configurations.

---

### 1.3 pgBench

* **Description:** Benchmarking tool for PostgreSQL.
* **Function:** Runs SQL workloads against the database to measure performance (transactions per second).
* **Standard:** Based on TPC (Transaction Processing Performance Council) standard tests.
* **Website:** [tpc.org](https://www.tpc.org) for more information on benchmarks.

---

## 2. PostgreSQL Configuration

* PostgreSQL is configured via a **property file** named `postgresql.conf`.
* **Format:** Parameter name = value (name-value pairs).
* **Purpose:** Controls server behavior, default database settings, and rules.

### 2.1 Managing Parameters

* Parameters can be set in multiple ways:

  1. **Configuration file (`postgresql.conf`)**: Permanent changes applied on reload.

     * Reload with: `pg_ctl reload` or SQL: `SELECT pg_reload_conf();`
  2. **Database/Role Overrides:** Use `ALTER DATABASE` or `ALTER ROLE`.
  3. **Session-specific changes:** Use `SET` command (applies only to the current session).

* **Check parameter value:**

  ```sql
  SHOW parameter_name;
  ```

* **Change parameter in session:**

  ```sql
  SET parameter_name = value;
  ```

### 2.2 Logging

* PostgreSQL logs messages to a **destination**; default is **standard error**.
* Controlled by parameters in `postgresql.conf`:

  * `logging_collector = on` → enables background process to redirect logs into files.
  * Other parameters control **what** and **when** to log.
* Recommended: Review documentation to understand logging options for your setup.

---

## 3. PostgreSQL System Catalog

* PostgreSQL maintains **metadata** about databases and clusters in the **system catalog**.
* Schema: `pg_catalog`
* Managed entirely by PostgreSQL; **users should not modify system catalog tables directly** unless absolutely necessary.

### 3.1 Important Views

* **pg_settings**

  * Provides access to runtime configuration parameters.
  * Can be queried like a regular table:

    ```sql
    SELECT name, setting, unit, context 
    FROM pg_settings;
    ```
  * Shows the same values as `SHOW` commands.

* **pg_roles** / **pg_rules**

  * Views providing information about database roles and rules.
  * Useful for understanding permissions and database-level configurations.

**Tip:** Browsing these views helps understand database metadata and configuration without manually querying system tables.

---

## 4. Summary

* **Tools:**

  * `psql` → command-line SQL client
  * `pgAdmin` → web-based GUI
  * `pgBench` → benchmarking tool

* **Configuration:**

  * Managed in `postgresql.conf`
  * Can be overridden per session or per database/role
  * Logging controlled by configuration parameters

* **System Catalog:**

  * Schema: `pg_catalog`
  * Contains metadata for databases, roles, and rules
  * Important views: `pg_settings`, `pg_roles`, `pg_rules`

> These tools and configurations are essential for PostgreSQL administration and performance management. You will use them extensively throughout the course.
