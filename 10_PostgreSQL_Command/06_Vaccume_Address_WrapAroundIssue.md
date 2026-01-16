## Transaction ID (TXID) Wraparound Problem

PostgreSQL assigns a **Transaction ID (TXID)** to every transaction. TXIDs are **32-bit values**, which means they live in a **finite, circular space**.

This design enables efficient comparisons but introduces a serious risk known as **transaction ID wraparound**.

---

## Circular TXID Space

TXIDs do not grow forever. Instead, they wrap around after reaching their maximum value.

Conceptually:

* At any moment, there is a **current TXID**
* Roughly **2 billion TXIDs are in the past**
* Roughly **2 billion TXIDs are in the future**

Only rows created by **past transactions** are visible to the current transaction.

> Visibility is determined by comparing row metadata against the transaction’s snapshot, not by wall-clock time.

---

## Visibility Boundary

For a given current TXID:

* Rows inserted by transactions in the **past half** of the TXID space are visible
* Rows inserted by transactions in the **future half** are invisible

This boundary continuously shifts as new transactions start.

---

## How Wraparound Happens

Consider this simplified sequence:

1. TXID = 1 inserts a row (`xmin = 1`)
2. Millions or billions of transactions execute
3. Current TXID advances close to `2^31 + 1`

Up to this point, the row created by TXID 1 is still considered **in the past** and remains visible.

Now the critical moment:

4. Current TXID advances to `2^31 + 2`

At this point:

* TXID 1 is now considered **in the future**
* The row created by TXID 1 becomes **invisible to all new transactions**

This is the **wraparound catastrophe**.

---

## Why Wraparound Is Dangerous

If wraparound is not prevented:

* Valid rows suddenly appear invisible
* PostgreSQL can no longer determine correct visibility
* Data corruption becomes possible

To protect data integrity, PostgreSQL will:

* **Stop accepting new transactions**
* Put the database into a **protective shutdown mode**

This is not a performance issue — it is a **correctness emergency**.

---

## Tuple Freezing: The Solution

PostgreSQL prevents wraparound using a process called **tuple freezing**.

### What Freezing Does

* Marks old rows as **frozen**
* Frozen rows are visible to **all transactions, forever**
* Visibility no longer depends on `xmin`

Once frozen, a row is immune to wraparound.

---

## How Freezing Works Internally

Each heap tuple contains header flags.

Freezing:

* Sets a **frozen bit** in the tuple header
* Uses the header field `t_infomask`

Before PostgreSQL 9.4:

* A special TXID value (`FrozenXID = 2`) was used

PostgreSQL 9.4 and later:

* A dedicated **frozen flag** is used instead

> Frozen tuples bypass normal `xmin` visibility checks.

---

## Frozen vs Non-Frozen Tuples

| Tuple Type | Visibility Rule                      |
| ---------- | ------------------------------------ |
| Frozen     | Visible to all transactions          |
| Non-frozen | Visible based on `xmin` and snapshot |

Example:

* First 3 rows frozen → always visible
* Remaining rows → normal MVCC rules apply

---

## When Does PostgreSQL Freeze Tuples?

Freezing is performed by the **VACUUM** command.

VACUUM decides which tuples to freeze based on:

* Age of the tuple’s `xmin`
* System-defined thresholds (e.g., `autovacuum_freeze_max_age`)

Older tuples are frozen first.

---

## Freezing Modes

PostgreSQL uses **two freezing strategies**:

---

### 1. Lazy Freezing (Normal Mode)

* Uses the **Visibility Map (VM)**
* Skips pages that have no dead tuples
* Minimizes I/O
* Used during routine VACUUM and autovacuum

Efficient and low-impact.

---

### 2. Aggressive Freezing (Eager Mode)

* Ignores the visibility map
* Scans **all pages** in the table
* Freezes as many tuples as possible

This mode is triggered when wraparound risk is high.

---

## Additional Visibility Map Optimization

During aggressive freezing:

* An additional VM bit may be set
* Indicates that **all tuples on the page are frozen**

Pages marked this way:

* Can be skipped by future freezing runs
* Reduce long-term maintenance cost

---

## What If Freezing Does Not Happen?

If VACUUM does not run frequently enough:

* TXIDs continue to age
* Wraparound risk increases

Eventually PostgreSQL will:

* Refuse new transactions
* Log wraparound warnings
* Force emergency maintenance

This is why **vacuuming is mandatory**, not optional.

---

## Vacuum Cost-Based Control

VACUUM can be resource-intensive.

PostgreSQL uses **cost-based throttling** to limit impact:

* `vacuum_cost_limit`
* `vacuum_cost_delay`

Mechanism:

* Each vacuum action consumes cost units
* When cost limit is exceeded, vacuum sleeps
* Prevents vacuum from overwhelming I/O

Proper tuning balances:

* System performance
* Wraparound safety

---

## Operational Reality: A Double-Edged Sword

* No vacuum → database becomes unsafe and unusable
* Infrequent vacuum → long, disruptive maintenance
* Regular vacuum → stable performance and safety

The goal is **continuous, low-impact maintenance**.

---

## Key Takeaways

* TXIDs live in a circular 32-bit space
* Wraparound can make valid rows invisible
* PostgreSQL prevents corruption by freezing tuples
* Frozen tuples are always visible
* VACUUM performs freezing automatically
* Cost-based controls prevent vacuum overload

---

## Memory Hook 🧠

> **TXIDs wrap. Data must not.**
>
> Freezing locks visibility in time so the database can keep moving forward.

---

## FAQ

### Q: Is wraparound rare?

No. Busy databases reach wraparound quickly without vacuum.

### Q: Can I disable freezing?

No. Freezing is essential for correctness.

### Q: Does autovacuum handle freezing?

Yes. Autovacuum performs freezing automatically.

### Q: Is freezing the same as VACUUM FULL?

No. Freezing happens during normal VACUUM and does not rewrite tables.

### Q: What happens if wraparound is detected?

PostgreSQL blocks new transactions to prevent data corruption.

### Q: Why does vacuum sometimes slow the system?

Because it performs real I/O work; cost-based controls limit the impact.
