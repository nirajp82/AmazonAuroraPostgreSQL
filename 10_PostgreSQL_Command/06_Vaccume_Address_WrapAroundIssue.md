## Transaction ID (TXID) Wraparound Problem

PostgreSQL assigns a **Transaction ID (TXID)** to every transaction. TXIDs are **32-bit values**, which means they live in a **finite, circular space**.

This design enables efficient comparisons but introduces a serious risk known as **transaction ID wraparound**.


Historically, TXIDs are stored as **32-bit integers**, giving a total space of about **4.29 billion values**. PostgreSQL intentionally treats this space as **circular**, not linear, so TXIDs can be compared efficiently using modular arithmetic. This choice makes MVCC fast, but it requires special handling to avoid wraparound corruption.

---

## Circular TXID Space

TXIDs do not grow forever. Instead, they wrap around after reaching their maximum value.

### Circular TXID Space

PostgreSQL transaction IDs (TXIDs) are **32-bit values**, so they do not grow forever.
After reaching the maximum value, they **wrap around**, forming a circular ID space.

#### How PostgreSQL interprets TXIDs

At any moment, PostgreSQL has a **current TXID**.
This current TXID conceptually divides the entire TXID space into **two logical halves**:

* **Past TXIDs** – transactions that started **before the current transaction’s snapshot**
* **Future TXIDs** – transactions that started **after the snapshot was taken**

Because the TXID space is circular, this division always results in:

* Roughly **2 billion TXIDs considered “past”**
* Roughly **2 billion TXIDs considered “future”**

This is why PostgreSQL describes the TXID space as **circular** rather than linear.


Internally, PostgreSQL compares TXIDs using special comparison logic rather than normal integer comparison. A TXID that is numerically “smaller” may be considered **newer** if it falls into the future half of the circular space relative to the current snapshot.

#### Important clarification: “past” and “future” are not time-based

The terms **past** and **future** do **not** refer to wall-clock time.

They mean only:

* **Past TXID** → a transaction that started **before the snapshot**
* **Future TXID** → a transaction that started **after the snapshot**

#### Visibility rule

A transaction can see **only rows created by past transactions** relative to its snapshot.

Row visibility is determined by comparing:

* the row’s transaction metadata (`xmin`, `xmax`)
* against the transaction’s snapshot

**Wall-clock time is never used** for visibility decisions.

For ex: When the current TXID is 2, PostgreSQL treats TXIDs 2 to 2³¹ + 1 as belonging to the future and therefore invisible, while TXIDs 2³¹ + 2 through 0 (after wraparound) are treated as the past and are visible. <img width="1113" height="482" alt="image" src="https://github.com/user-attachments/assets/2b6c4f3e-a4d4-4381-a29f-5ca40adee867" />

---

## Visibility Boundary

For a given current TXID:

* Rows inserted by transactions in the **past half** of the TXID space are visible
* Rows inserted by transactions in the **future half** are invisible

This boundary continuously shifts as new transactions start.


This shifting boundary is what makes **very old rows dangerous**: as the current TXID advances, rows that were once safely in the “past” half can eventually cross into the “future” half unless they are frozen.

---

## How TXID Wraparound Happens

PostgreSQL transaction IDs (TXIDs) are **32-bit numbers** that go from **1 up to about 4.29 billion**.
To handle visibility, PostgreSQL treats TXIDs as a **circle**. This is why very old rows can eventually appear “in the future” if wraparound is not prevented.


In practice, the danger point is not the full 4.29 billion range, but roughly **half of it (~2.1 billion)**. Crossing that midpoint breaks the assumption that “older TXIDs are always in the past.”

### Step-by-step example

1. **Row is inserted**

   * **TXID = 1** inserts a row (`xmin = 1`)
   * Millions or billions of transactions happen afterward
   * The **current TXID** advances, but TXID 1 is still considered in the **past**
   * ✅ The row remains **visible**

2. **Approaching the halfway point**

   * Current TXID gets close to **2.1 billion** (half of the TXID range)
   * TXID 1 is still behind the current TXID
   * ✅ The row is **still visible**

3. **The critical moment**

   * Current TXID passes **2.1 billion + 1**
   * Now, because of **circular TXID logic**, TXID 1 is considered in the **future**
   * ❌ The row created by TXID 1 becomes **invisible** to all new transactions

**Result:**
Even though the row still exists on disk, PostgreSQL “thinks” it hasn’t been created yet. This is the **wraparound problem**. <img width="1043" height="518" alt="image" src="https://github.com/user-attachments/assets/ca4ecaa4-ec44-45a6-9c20-680a79f1a2db" />

---

### Key takeaway

> TXIDs are not linear forever — they **wrap around after half of the 4.29-billion space**, and PostgreSQL uses circular arithmetic to decide visibility.
> Without proper maintenance (like **VACUUM**), very old rows can temporarily disappear from queries.

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


PostgreSQL may first log **wraparound warnings**, then force **aggressive autovacuum**, and finally **refuse writes or shut down** if it cannot guarantee correct visibility.

---

## Tuple Freezing: The Solution

PostgreSQL prevents wraparound using a process called **tuple freezing**.

### What Freezing Does

* Marks old rows as **frozen**
* Frozen rows are visible to **all transactions, forever**
* Visibility no longer depends on `xmin`

  <img width="1012" height="406" alt="image" src="https://github.com/user-attachments/assets/d97380e8-708e-4153-8b88-edadae244211" />

Once frozen, a row is immune to wraparound.

🔹 **(Added – lifetime clarification)**
**Yes: once a row becomes frozen, it stays frozen for its entire lifetime.**
A frozen tuple will **never be “unfrozen”**. The only way it stops existing is if the row is **updated or deleted**, in which case a **new tuple version** is created (with a new `xmin`) and that new version must eventually be frozen again.

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


Internally, frozen tuples are treated as if they were created “before any possible transaction,” making them permanently visible without comparing TXIDs.

---

## Frozen vs Non-Frozen Tuples

| Tuple Type | Visibility Rule                      |
| ---------- | ------------------------------------ |
| Frozen     | Visible to all transactions          |
| Non-frozen | Visible based on `xmin` and snapshot |

Example:

* First 3 rows frozen → always visible
* Remaining rows → normal MVCC rules apply

  <img width="1111" height="506" alt="image" src="https://github.com/user-attachments/assets/68800519-3efa-4c22-9c4b-f08d215cb2ca" />


If a frozen row is later **updated**, the new version is **not frozen** initially. It will follow normal MVCC rules and must be frozen again later by VACUUM.

---

## When Does PostgreSQL Freeze Tuples?

Freezing is performed by the **VACUUM** command.

VACUUM decides which tuples to freeze based on:

* Age of the tuple’s `xmin`
* System-defined thresholds (e.g., `autovacuum_freeze_max_age`)

Older tuples are frozen first.


PostgreSQL tracks freezing progress using:

* **`relfrozenxid`** (per table)
* **`datfrozenxid`** (per database)

These values represent the oldest TXID that is still unfrozen and are used to calculate **TXID age**.

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


Aggressive freezing is automatically triggered when table or database age approaches safety thresholds, even if it causes noticeable I/O.

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


Long-running or idle-in-transaction sessions can **block freezing**, because PostgreSQL cannot freeze tuples that might still be visible to an old snapshot.

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
* **Once frozen, a tuple stays frozen for its lifetime**
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

### Q: Once a row is frozen, does it stay frozen forever?

**Yes.** A frozen tuple remains frozen for its entire lifetime.
Only **updates or deletes** create new tuple versions that must be frozen again later.

### Q: What happens if wraparound is detected?

PostgreSQL blocks new transactions to prevent data corruption.

### Q: Why does vacuum sometimes slow the system?

Because it performs real I/O work; cost-based controls limit the impact.

### Q: If TXIDs are only 32-bit, doesn’t PostgreSQL eventually run out of TXIDs?

**No. PostgreSQL does not run out of TXIDs.**

TXIDs are **designed to wrap around**. Reuse of TXID values is intentional and safe **as long as old rows are frozen**.

PostgreSQL does **not** require TXIDs to be globally unique forever. It only requires that **unfrozen rows never become ambiguous** during wraparound.

---

### Q: Then why does “TXID wraparound” sound so dangerous?

Because wraparound becomes dangerous **only if old rows are still unfrozen**.

The real problem is not “TXIDs wrapping”, but this situation:

* An old row still has an unfrozen `xmin`
* The TXID counter advances past half the TXID space
* PostgreSQL can no longer tell whether that `xmin` is in the past or the future

That ambiguity breaks MVCC visibility rules — **that’s what PostgreSQL must prevent at all costs**.

---

### Q: How does freezing prevent running out of TXIDs?

Freezing **removes the dependency on TXIDs entirely** for old rows.

When a row is frozen:

* Its `xmin` is no longer compared to transaction snapshots
* PostgreSQL treats it as “created before all possible transactions”
* TXID wraparound becomes irrelevant for that row

Once rows are frozen, PostgreSQL can reuse TXIDs indefinitely without confusion.

---

### Q: If all rows are frozen, can PostgreSQL keep allocating new TXIDs forever?

**Yes.**

If all existing rows are frozen:

* `relfrozenxid` advances
* `datfrozenxid` advances
* Database “age” stays low

At that point, PostgreSQL can allocate **billions of new TXIDs**, wrap around the TXID counter, and continue operating safely.

This is exactly how PostgreSQL is designed to run long-term.

---

### Q: What happens when TXIDs actually wrap?

Nothing bad — **if freezing has kept up**.

In a healthy system:

1. TXIDs advance
2. Old rows get frozen
3. TXIDs wrap around
4. Visibility still works correctly
5. The database keeps running

Wraparound itself is normal and expected.
**Unfrozen old rows are the only risk.**

---

### Q: Why doesn’t PostgreSQL just use 64-bit TXIDs instead?

Because wraparound would still happen eventually, just later.

Using a larger counter would:

* Delay the problem
* Increase storage and comparison costs
* Not eliminate the need for freezing

Freezing is the **fundamental solution**, not counter size. PostgreSQL optimizes for efficiency and correctness rather than infinite counters.

---

### Q: If everything is frozen, is there “no room left” for new rows?

No. Freezing has **nothing to do with storage space or row limits**.

Freezing only affects:

* Visibility rules
* TXID comparisons

You can always:

* INSERT new rows
* UPDATE existing rows
* DELETE rows

New rows simply start as **unfrozen** and will be frozen later by VACUUM.

---

### Q: Can frozen rows ever become unfrozen?

**No. Frozen means frozen forever — for that tuple version.**

A frozen tuple:

* Will never be unfrozen
* Will never depend on TXIDs again

However:

* If a frozen row is **updated**, PostgreSQL creates a **new tuple version**
* That new version starts **unfrozen**
* It will later be frozen by VACUUM

So freezing is permanent per tuple, not per logical row.

---

### Q: Can a database be in a state where everything is frozen?

Yes — temporarily.

At a given moment:

* All existing rows may be frozen
* Database age may be very low
* Wraparound risk is minimal

As soon as new writes happen, new unfrozen rows appear and the normal freeze cycle continues.

This is a **healthy and desirable state**.

---

### Q: What actually causes PostgreSQL to shut down for wraparound?

PostgreSQL shuts down **before** wraparound causes corruption, when it detects that:

* `datfrozenxid` is dangerously old
* There exist unfrozen rows that cannot be safely compared after wraparound
* Freezing is being blocked (often by long-running transactions)

This shutdown is a **protective correctness measure**, not a failure of design.

---

### Q: What are the real operational risks related to TXIDs?

The real risks are:

* Autovacuum disabled or misconfigured
* Very large tables not vacuumed often enough
* Long-running or idle-in-transaction sessions blocking freezing
* Ignoring wraparound warnings in logs

Not TXID exhaustion.

---

### Q: What is the correct mental model for TXIDs?

**Incorrect model ❌**

> TXIDs must remain unique forever

**Correct model ✅**

> TXIDs are temporary labels; old data must stop depending on them

Freezing is how PostgreSQL safely forgets old TXIDs.

---

### One-sentence summary

> **PostgreSQL never “runs out” of TXIDs — it only runs out of safety if old rows are not frozen before TXIDs wrap.**

