* [Postgres SQL Internal Architecture & Operations Guide](https://github.com/nirajp82/AmazonAuroraPostgreSQL/blob/main/02PostgreSQLFundamentals/01_StorageProcessesQueryProcessing.md?utm_source=chatgpt.com)  
  * [ ] **Storage:** Data is split into 1GB files and 8KB pages.
  * [ ] **Processes:** Postmaster listens; Backend processes execute queries.
  * [ ] **Query Flow:** Parse  Analyze  Rewrite  Plan  Execute.
  * [ ] **Tuning:** Keep stats updated with `ANALYZE`; check plans with `EXPLAIN`.
  * [ ] **Caching:** Monitor Hit Ratio using the provided script and tune `shared_buffers` accordingly.


