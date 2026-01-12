# Aurora PostgreSQL Versions and Feature Availability

Before committing to a **specific PostgreSQL version**, an **Aurora feature**, or an **AWS region**, it is critical to understand how **Aurora PostgreSQL versions differ from community PostgreSQL**, and how **feature availability depends on both the engine version and the region**.

---

## 1. Aurora PostgreSQL vs Community PostgreSQL

Amazon Aurora PostgreSQL is **not a direct mirror** of the community PostgreSQL release.

* Aurora manages its **own versions, release cycles, and timelines**, independent of the PostgreSQL community.
* The **Aurora PostgreSQL engine version** does **not match** the community PostgreSQL version number.

### Example

*  ```SELECT Version()```
  * Community PostgreSQL version: **12.4**
* ```SELECT Aurora_Version()```
  * Aurora PostgreSQL engine version: **4.4.2**

Even though Aurora is compatible with PostgreSQL, the **version numbers are different**, and this distinction must be kept in mind when planning upgrades or enabling features.

---

## 2. Aurora Release and Support Policy

* Aurora generally releases **minor versions on a quarterly basis**.
* Each minor version is supported for **12 months**.
* Some minor versions are marked as **Long-Term Support (LTS)**, meaning they are supported **beyond 12 months**.

This allows customers to remain on a stable version for a longer period without frequent upgrades.

---

## 3. Community PostgreSQL Versions Lag Behind on Aurora

The **latest community PostgreSQL version is not immediately available on Aurora**.

### Example (as of December 2021)

* Latest community PostgreSQL version: **14**
* Latest Aurora PostgreSQL-supported version: **13.4**

This lag exists because AWS thoroughly tests and integrates PostgreSQL features into Aurora’s distributed storage architecture before releasing support.

---

## 4. Regional Availability of Versions

New Aurora PostgreSQL versions are **not rolled out to all AWS regions simultaneously**.

* A PostgreSQL version available in one region **may not yet be available in another**.
* This is especially important for **multi-region architectures**, such as **Aurora Global Databases**.

**Key takeaway:**
Always verify **version availability in every AWS region** you plan to use.

---

## 5. Feature Availability Depends on Version and Region

Not all Aurora features are:

* Available in **every PostgreSQL version**
* Available in **every AWS region**

Feature availability depends on:

1. **PostgreSQL-compatible version**
2. **Aurora engine version**
3. **AWS region**

Before enabling a feature, you must ensure:

* The feature is supported for your **PostgreSQL version**
* The feature is available in **all target regions**

---

## 6. Example: Verifying Aurora Global Database Support

Suppose you want to use **Aurora Global Database** with:

* PostgreSQL version: **12.4**
* Regions: **us-east-2** and **us-east-1**

Steps:

1. Check AWS documentation under
   **“Aurora supported features by region and engine”**
2. Verify:

   * Global Database is supported for **PostgreSQL 10.11 or higher**
   * PostgreSQL **12.4** is supported in **us-east-2**
   * PostgreSQL **12.4** is also supported in **us-east-1**

Since all conditions are met, you can safely create a **Global Database spanning us-east-1 and us-east-2** using PostgreSQL 12.4.

---

## 7. Key Takeaways

* Aurora PostgreSQL versions are **independent of community PostgreSQL versions**
* Aurora controls its **own release cycles and support timelines**
* New PostgreSQL versions **arrive later** on Aurora
* PostgreSQL versions and Aurora features are **not available in all regions simultaneously**
* **Feature availability depends on both version and region**
* **Always check AWS documentation** before committing to:

  * A PostgreSQL version
  * An Aurora feature
  * A specific AWS region

---

### Bottom Line

Before finalizing your database architecture:

* Validate the **PostgreSQL version**
* Confirm **feature support**
* Confirm **regional availability**

Doing this upfront avoids **re-architecture, downtime, or unsupported configurations later**.
