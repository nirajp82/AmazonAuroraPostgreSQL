# PostgreSQL Extensions in Amazon Aurora

In this lesson, you will learn about **PostgreSQL extensions**, their purpose, available examples, and how to manage them on Aurora PostgreSQL.

Extensions are **plug-and-play modules** that extend or enhance the functionality of PostgreSQL without modifying the core database engine.

---

## What Are Extensions?

* Extensions add new features to PostgreSQL, such as:

  * New **data types**
  * Improved **monitoring**
  * **Foreign data wrappers** to access external data sources
  * Enhanced **security features**

* Pre-packaged extensions are included in PostgreSQL distributions and Aurora PostgreSQL.

* Extensions can be **enabled or disabled** as needed, avoiding unnecessary resource usage.

---

## Examples of Extensions

| Extension            | Purpose                                                  |
| -------------------- | -------------------------------------------------------- |
| `postgis`            | Adds support for geographic objects and GIS queries      |
| `pg_stat_statements` | Provides query execution metrics for monitoring          |
| `postgres_fdw`       | Connects to other PostgreSQL databases as foreign tables |
| `plv8`               | Enables JavaScript functions in PostgreSQL               |
| `pgaudit`            | Adds auditing capabilities                               |
| `uuid-ossp`          | Provides functions to generate UUIDs                     |

> **Memory Hook:** Think of extensions as **“plug-ins”**—only load what you need, when you need it.

---

## Managing Extensions in Aurora PostgreSQL

1. **List Available Extensions**

```sql
SELECT * FROM pg_available_extensions;
```

2. **Check Installed Extensions**

```sql
SELECT * FROM pg_extension;
```

3. **Install an Extension**

```sql
CREATE EXTENSION extension_name;
```

*Note:* Only extensions pre-packaged with Aurora PostgreSQL can be installed.

4. **Remove an Extension**

```sql
DROP EXTENSION extension_name [CASCADE];
```

<img width="1018" height="568" alt="image" src="https://github.com/user-attachments/assets/253ea432-2dd3-41cf-9fef-0f06916090e3" />

* The `CASCADE` option automatically removes dependent objects.

5. **Schema Selection**

* Extensions are installed in a schema.
* If no schema is specified, the **default schema** is used:

```sql
CREATE EXTENSION extension_name SCHEMA schema_name;
```

---

## Key Points

* Extensions are **modules** that enhance PostgreSQL functionality.
* Aurora PostgreSQL **pre-packages supported extensions**; you cannot add custom extensions due to lack of server access.
* Extensions are **modular and optional**, preventing bloat in the database engine.
* Some extensions may **depend on other extensions**, which can be handled automatically with the `CASCADE` option.
* Only **database owners or superusers** can create or drop extensions.

  * Aurora PostgreSQL 13+ supports **trusted extensions**, which can be managed without superuser privileges.

---

## Benefits of Using Extensions

* **Modularity:** Only load what you need.
* **No core modifications:** Avoid forking PostgreSQL.
* **Custom features:** Developers can add features for specific use cases.
* **Better resource utilization:** No unnecessary features are loaded.

---

## FAQ

### ❓ Can I add custom extensions in Aurora PostgreSQL?

No. Only extensions pre-packaged with Aurora PostgreSQL can be used, since you don’t have VM or filesystem access.

### ❓ What happens if an extension depends on another extension?

You can use the `CASCADE` option with `CREATE EXTENSION` to automatically install all dependencies.

### ❓ Do extensions affect all databases in a cluster?

No. Extensions are **installed per database**. You must enable them in each database where you need the functionality.

### ❓ Can I control the schema for an extension?

Yes. Use the `SCHEMA` option when creating the extension; otherwise, it defaults to the database's default schema.

---

> **Memory Hook:**
> Extensions = **Plug-and-Play Features for PostgreSQL**
> Only load what you need, no access to server required on Aurora, modular and safe.
>
---
<img width="1075" height="555" alt="image" src="https://github.com/user-attachments/assets/ed53ab77-bfe8-41d6-9bee-7df39600c71e" />

<img width="1136" height="277" alt="image" src="https://github.com/user-attachments/assets/56b3eda4-84ea-4a8e-8ba2-292e7f9232a7" />

<img width="835" height="402" alt="image" src="https://github.com/user-attachments/assets/9a9b001e-ef1b-4c1e-95f9-9a5e8f31382e" />

<img width="748" height="155" alt="image" src="https://github.com/user-attachments/assets/f513fa0b-d506-42d2-aaee-07360431a360" />

<img width="1590" height="351" alt="image" src="https://github.com/user-attachments/assets/07956646-075e-421a-9492-b0d346863b5e" />


