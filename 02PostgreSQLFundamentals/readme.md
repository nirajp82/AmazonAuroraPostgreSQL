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

 * 
