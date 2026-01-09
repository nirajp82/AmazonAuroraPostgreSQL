* [Postgres SQL Internal Architecture & Operations Guide](https://github.com/nirajp82/AmazonAuroraPostgreSQL/blob/main/02PostgreSQLFundamentals/01_StorageProcessesQueryProcessing.md?utm_source=chatgpt.com)  
  *  **Storage:** Data is split into 1GB files and 8KB pages.
  *  **Processes:** Postmaster listens; Backend processes execute queries.
  *  **Query Flow:** Parse  Analyze  Rewrite  Plan  Execute.
  *  **Tuning:** Keep stats updated with `ANALYZE`; check plans with `EXPLAIN`.
  *  **Caching:** Monitor Hit Ratio using the provided script and tune `shared_buffers` accordingly.
    
* [WAL Checkpointing Archiver Background Processes](https://github.com/nirajp82/AmazonAuroraPostgreSQL/blob/main/02PostgreSQLFundamentals/02_WALCheckpointingArchiverBackgroundProcesses.md)
  * **WAL** ensures durability using fast sequential writes
  * **Commit** happens only after WAL is flushed to disk
  * **Checkpoints** sync memory with data files
  * **BgWriter** prevents sudden I/O pressure
  * **Archiving** enables long-term recovery options*  

* [PostgreSQL High Availability and Replication](https://github.com/nirajp82/AmazonAuroraPostgreSQL/edit/main/02PostgreSQLFundamentals/03_HighAvailabilityStandbyServersReplication.md)
  * High availability relies on standby servers.
  * **Replication lag** is a critical factor in failover planning.
  * **Synchronous replication** prevents data loss but reduces performance.
  * **Hot standby** allows read scaling; **warm standby** is simpler but only used for failover.
  * PostgreSQL provides multiple replication mechanisms to suit different use cases.

* [PostgreSQL Scalability](https://github.com/nirajp82/AmazonAuroraPostgreSQL/blob/main/02PostgreSQLFundamentals/04_Scaling.md)
  * For read-heavy systems, **hot standby + load balancer** is a simple horizontal scaling solution.
  * **Replication type** affects data freshness and performance:
    * **Asynchronous:** Low overhead, eventual consistency.
    * **Synchronous:** No data loss, but slower writes.
  * For write-heavy scaling, consider **sharding** or **commercial multi-master solutions**.
  * PostgreSQL does not provide a native out-of-the-box mechanism for scaling writes across multiple servers.
     
* [PostgreSQL Tools, Configuration, and System Catalog](https://github.com/nirajp82/AmazonAuroraPostgreSQL/blob/main/02PostgreSQLFundamentals/05_Tools.md)
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
