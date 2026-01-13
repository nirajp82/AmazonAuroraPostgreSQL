# PostgreSQL Logging (Aurora / RDS)

## Purpose of PostgreSQL Logs

PostgreSQL logs capture **server-side activity and errors** and are used for:

* Debugging failures
* Auditing activity
* Performance troubleshooting
* Operational maintenance

Logs are **written by a background logging process** that writes log entries to the filesystem.

---

## How PostgreSQL Logging Works (High Level)

* Logging is handled by a **background process**
* Log behavior is controlled by **configuration parameters**
* Parameters are managed via:

  * **Custom Parameter Groups** (persistent)
  * **Session-level settings** (temporary)

---

### Memory Hook 🧠

**Think of PostgreSQL logging as a security camera system**

* Default → only records crimes
* Configuration → decides whether to record entrances, exits, or everything

---

## Configuration Management

### Custom Parameter Groups

Custom parameter groups are used to:

* Persist logging configuration
* Control logging at the database engine level

📌 You can inspect parameter values using:

```sql
SHOW parameter_name;
```

Example:

```sql
SHOW logging_collector;
```

* `1` → logging enabled
* `0` → logging disabled

---

### Session-Level Logging

Logging can also be enabled **per session**.

⚠️ Important:

* Session-level changes:

  * Apply **only to the current session**
  * Are **not permanent**

Example:

```sql
SET log_statement = 'all';
```

* Logs **all statements executed in that session only**

---

## Log Retention

### Default Retention

* **3 days** (Aurora/RDS default)
* Stored in **minutes**, not days

Parameter:

```sql
SHOW rds.log_retention_period;
```

* Value is in **minutes**
* Maximum allowed: **7 days**

---

### Long-Term Retention

If retention > 7 days is required:

* Publish PostgreSQL logs to **CloudWatch Logs**

✔️ Recommended for:

* Compliance
* Auditing
* Long-term analysis

---

## Default Logging Behavior (Out of the Box)

By default, PostgreSQL logs **only errors**, including:

### Logged by Default

* SQL errors
  Example:

  ```sql
  SELECT * FROM non_existing_table;
  ```
* Server-level failures:

  * Out of memory
  * Disk full
* Deadlocks

❗ **Successful queries are NOT logged by default**

---

## Expanding What Gets Logged

You can enable additional logging using configuration parameters.

Examples:

* Connections
* Disconnections
* Query text
* Query duration
* Lock waits
* Execution statistics

---

## Non-Modifiable Log Parameters (Aurora / RDS)

These parameters **cannot be changed**:

| Parameter                  | Reason            |
| -------------------------- | ----------------- |
| `log_directory`            | Fixed destination |
| `log_line_prefix`          | Fixed format      |
| `log_file_mode`            | Fixed permissions |
| `log_timezone`             | Always UTC        |
| Log truncation on rotation | Disabled          |

---

## Modifiable Logging Parameters

These can be set via:

* Custom Parameter Group (persistent)
* Session-level (temporary)

---

### Log File Naming

```sql
log_filename
```

Default pattern:

* `%Y` → year
* `%m` → month
* `%d` → date
* `%H` → hour
* `%M` → minute

---

### Log Rotation

| Parameter           | Purpose                             |
| ------------------- | ----------------------------------- |
| `log_rotation_age`  | Rotate logs based on time (minutes) |
| `log_rotation_size` | Rotate logs based on file size      |

Defaults:

* Rotation age: **60 minutes**
* Size-based rotation may happen earlier on busy systems

---

### Connection Logging

| Parameter                 | Effect                      |
| ------------------------- | --------------------------- |
| `log_connections = on`    | Logs session start          |
| `log_disconnections = on` | Logs session end + duration |
| `log_hostname = on`       | Logs client hostname        |

---

### Statement Logging (Critical Section)

#### Which statements to log

```sql
log_statement
```

Values:

* `none` (default)
* `ddl`
* `mod`
* `all`

⚠️ `all` can generate **huge logs** in production.

---

#### Statement Duration

| Parameter                    | Description                      |
| ---------------------------- | -------------------------------- |
| `log_duration`               | Logs duration for all statements |
| `log_min_duration_statement` | Logs only slow queries           |

Example:

```sql
log_min_duration_statement = 500
```

→ Logs queries taking **≥ 500 ms**

✔️ Preferred for production

---

### Error Verbosity

```sql
log_error_verbosity
```

Controls how detailed error messages are.

More verbosity → faster log growth

---

### Lock & Stats Logging

| Parameter             | Purpose               |
| --------------------- | --------------------- |
| `log_lock_waits`      | Logs lock waits       |
| `log_statement_stats` | Logs cumulative stats |
| `log_parser_stats`    | Parser timing         |
| `log_executor_stats`  | Executor timing       |

📌 Stats logging is **off by default**

---

## Permissions & Constraints

* Some parameters require **superuser privileges**
* Some parameters are **mutually exclusive**

  * Only one may be enabled at a time
* Always consult documentation before enabling multiple logging options

---

## Key Takeaways

* PostgreSQL logs:

  * Errors by default
  * Everything else is opt-in
* Logging can be:

  * Temporary (session)
  * Permanent (custom parameter group)
* Excessive logging can:

  * Increase I/O
  * Increase storage usage
  * Impact performance

---

## Which Parameter Should We Configure? (Clear Answer)

### Most Important Parameters (Recommended Order)

#### 1️⃣ **`log_min_duration_statement`** ✅

✔ Best signal-to-noise ratio
✔ Safest for production
✔ Identifies slow queries without flooding logs

---

#### 2️⃣ **`log_connections` / `log_disconnections`**

✔ Useful for:

* Connection leaks
* Session churn
* Auditing

---

#### 3️⃣ **`log_lock_waits`**

✔ Critical for diagnosing:

* Blocking
* Deadlocks
* Concurrency issues

---

#### 4️⃣ **Avoid `log_statement = all` in production**

❌ Extremely noisy
❌ High performance impact
✔ Use only temporarily or in dev/test

---

### Memory Hook 🧠

**Production logging rule:**

> Log **slow**, **blocked**, and **failed** — not everything.

---

## FAQ

### Q1. Why doesn’t PostgreSQL log all queries by default?

Because:

* High overhead
* Massive log volume
* Most queries are not useful for troubleshooting

---

### Q2. Should I enable `log_statement = all`?

Only:

* Temporarily
* In dev or troubleshooting sessions

---

### Q3. How do I make logging changes permanent?

* Create or modify a **custom parameter group**
* Attach it to the DB instance
* Reboot if required

---

### Q4. Where are logs stored long-term?

* Locally → limited retention
* CloudWatch Logs → long-term retention

---

### Q5. Which single parameter gives the most value?

👉 **`log_min_duration_statement`**
