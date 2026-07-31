# 05 — Concurrency & Transactions

[← Back to index](README.md) · [← Performance](04-performance.md) · [Next: T-SQL Cookbook →](06-tsql-cookbook.md)

Why your refresh procedure blocks your website, and what to do about it.

---

## Contents

- [Transactions](#transactions)
- [ACID](#acid)
- [Isolation levels](#isolation-levels)
- [Snapshot isolation and RCSI](#snapshot-isolation-and-rcsi)
- [Locks](#locks)
- [Blocking vs deadlock](#blocking-vs-deadlock)
- [Deadlock prevention](#deadlock-prevention)
- [Application locks](#application-locks)
- [Optimistic concurrency](#optimistic-concurrency)
- [Practical guidance](#practical-guidance)

---

## Transactions

A unit of work that either fully happens or fully doesn't.

```sql
BEGIN TRAN;
    UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;
    UPDATE Accounts SET Balance = Balance + 100 WHERE Id = 2;
COMMIT;
```

Every individual statement is already a transaction implicitly. Explicit transactions group several into one all-or-nothing unit.

### The standard scaffolding

```sql
SET XACT_ABORT ON;

BEGIN TRY
    BEGIN TRAN;
        -- work
    COMMIT;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK;
    THROW;
END CATCH
```

**`SET XACT_ABORT ON`** — without it, some runtime errors abort only the *statement* and leave the transaction open and half-applied. If the client then disconnects, SQL Server rolls back, but meanwhile you're holding locks. **Always on when you open a transaction.**

**`XACT_STATE()`** — returns `1` (committable), `-1` (doomed, must roll back), `0` (none active). Checking beats `@@TRANCOUNT` because a doomed transaction still has a non-zero trancount but cannot be committed.

**`THROW`** re-raises with the original error number and severity intact. `RAISERROR` is legacy and loses that information.

### Nested transactions are a lie

`BEGIN TRAN` inside `BEGIN TRAN` just increments `@@TRANCOUNT`. The inner `COMMIT` decrements it and commits nothing. **A single `ROLLBACK` anywhere rolls back everything**, all the way out, regardless of nesting.

If you need partial rollback, use **savepoints**:

```sql
SAVE TRANSACTION MySavePoint;
-- ...
ROLLBACK TRANSACTION MySavePoint;   -- rolls back to here only
```

### Keep transactions short

The single most important operational rule about transactions. A transaction holds its locks until commit or rollback. Long transactions mean long blocking.

**Never** put a user interaction, an HTTP call, a file read, or a `WAITFOR` inside an open transaction. Do the slow work first, then open the transaction, write, and commit.

---

## ACID

| Property | Means | Delivered by |
|---|---|---|
| **Atomicity** | All or nothing | Transaction log + rollback |
| **Consistency** | Constraints hold at the end | Constraint enforcement |
| **Isolation** | Concurrent transactions don't see each other's mess | Locking / row versioning |
| **Durability** | Committed survives a crash | Write-ahead logging |

**Write-ahead logging** is the mechanism behind both atomicity and durability: the log record is written to disk *before* the data page is. On crash recovery, SQL Server replays committed transactions from the log and rolls back uncommitted ones. This is why the log disk's write latency matters so much, and why you never delete the log file.

---

## Isolation levels

The dial controlling how much of other transactions' work-in-progress you can see, traded against how much blocking you cause.

| Level | Dirty read | Non-repeatable read | Phantom | Mechanism |
|---|---|---|---|---|
| `READ UNCOMMITTED` | ✅ possible | ✅ | ✅ | No shared locks |
| `READ COMMITTED` *(default)* | ❌ | ✅ | ✅ | Shared locks, released immediately |
| `REPEATABLE READ` | ❌ | ❌ | ✅ | Shared locks held to commit |
| `SERIALIZABLE` | ❌ | ❌ | ❌ | Range locks |
| `SNAPSHOT` | ❌ | ❌ | ❌ | Row versions in tempdb |
| `READ COMMITTED SNAPSHOT` | ❌ | ✅ | ✅ | Row versions, statement-level |

### The three anomalies

**Dirty read** — you read data another transaction wrote but hasn't committed. It may be rolled back, meaning you read a value that never existed.

**Non-repeatable read** — you read a row twice in one transaction and get different values, because someone committed a change between your reads.

**Phantom read** — you run the same range query twice and get *extra rows*, because someone inserted into your range.

### `NOLOCK` — the most misused hint in SQL Server

`WITH (NOLOCK)` is `READ UNCOMMITTED` on one table. What people think it does: "read without blocking." What it actually does:

- Reads uncommitted data that may be rolled back (values that never existed)
- **Can return the same row twice**
- **Can skip rows entirely** — if a page split moves rows during your scan
- Can fail with error 601 ("could not continue scan with NOLOCK due to data movement")

Those middle two are the ones people don't know. It's not a performance feature; it's a correctness trade, and the price is silently wrong results with no error.

> **If you want non-blocking reads, enable `READ_COMMITTED_SNAPSHOT`.** It gives you exactly what people want from `NOLOCK` — readers don't block, writers don't block readers — while returning transactionally consistent data.

Legitimate `NOLOCK` uses: a rough row count for monitoring; checking progress on a long-running load. Anything where "approximately right" is genuinely acceptable.

---

## Snapshot isolation and RCSI

Both use **row versioning**: instead of blocking, readers get the last committed version of a row from a version store in tempdb.

### `READ_COMMITTED_SNAPSHOT` (RCSI) — the one to turn on

```sql
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
```

Changes the behavior of the **existing default** isolation level. **No code changes needed** — every query already running as `READ COMMITTED` starts using versions instead of shared locks.

**Effect:** readers see data as of the start of their *statement*. Readers don't block writers. Writers don't block readers. Writers still block writers, as they must.

**This is usually the single biggest concurrency quality-of-life improvement available to a SQL Server shop**, and it's the direct answer to "my refresh procedure blocks the website."

**Costs:**
- 14 bytes added to each row as it's modified (one-time, on first update after enabling)
- tempdb usage for the version store — size it accordingly
- Long-running transactions hold versions open, growing tempdb
- Requires exclusive database access to enable (`WITH ROLLBACK IMMEDIATE` kicks everyone off — do it in a maintenance window)

**One behavior change to know:** with RCSI, a read-modify-write pattern can now operate on a slightly stale read where before it would have blocked and gotten fresh data. If you have code doing `SELECT` then `UPDATE` based on what it read, add `UPDLOCK` to the select or use a single statement.

### `SNAPSHOT` isolation — the opt-in sibling

```sql
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;
-- then, per session:
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

Transaction-level rather than statement-level: you see the database as of the start of your *transaction*, consistently, for its whole duration. Excellent for long reports that must be internally consistent.

Cost: update conflicts throw error 3960 if someone else modified a row you're modifying. Your code must handle and retry.

---

## Locks

### Lock modes

| Mode | Name | Compatible with |
|---|---|---|
| **S** | Shared (reading) | S, and IS |
| **X** | Exclusive (writing) | nothing |
| **U** | Update (intent to write) | S only — prevents conversion deadlocks |
| **IS/IX** | Intent (a lock exists at a lower level) | signals up the hierarchy |
| **Sch-S / Sch-M** | Schema stability / modification | Sch-M blocks everything |

**Update locks (U)** exist specifically to prevent a common deadlock: two transactions both take shared locks on a row, then both try to upgrade to exclusive, and neither can because of the other. U locks are taken during the search phase of an update and are mutually incompatible, so only one searcher can be in position to upgrade.

### Lock granularity and escalation

Row → Page → Table. SQL Server picks a level based on the operation.

**Lock escalation:** at roughly 5,000 locks on one object, the engine trades them all for a single table lock to save memory. Efficient for the engine, terrible for everyone else waiting on that table.

This is a primary reason to **batch large DML**. It's also why a delete of 50 million rows blocks the entire table rather than just its rows.

You can disable escalation per table (`ALTER TABLE t SET (LOCK_ESCALATION = DISABLE)`), but batching is almost always the better fix.

### Seeing locks

```sql
SELECT l.request_session_id, l.resource_type, l.request_mode, l.request_status,
       OBJECT_NAME(p.object_id) AS ObjectName
FROM   sys.dm_tran_locks AS l
LEFT JOIN sys.partitions AS p ON p.hobt_id = l.resource_associated_entity_id
WHERE  l.resource_database_id = DB_ID()
ORDER  BY l.request_session_id;
```

---

## Blocking vs deadlock

> **Blocking is traffic. Deadlock is a crash.**

Confusing these sends you down entirely the wrong debugging path.

| | Blocking | Deadlock |
|---|---|---|
| What | One waits for another's lock | Two wait for each other |
| Resolves | Yes, when the blocker commits | No — SQL Server kills one |
| Normal? | Yes, it's how isolation works | No, it's a bug in access patterns |
| Error | None (just slow) | 1205, victim rolled back |
| Fix | Shorten transactions, add indexes, RCSI | Consistent lock ordering, shorter transactions |

### Finding the blocking chain

```sql
SELECT
    blocked.session_id       AS BlockedSpid,
    blocked.blocking_session_id AS BlockerSpid,
    blocked.wait_type, blocked.wait_time,
    blocked_sql.text         AS BlockedSql,
    blocker_sql.text         AS BlockerSql
FROM sys.dm_exec_requests AS blocked
CROSS APPLY sys.dm_exec_sql_text(blocked.sql_handle) AS blocked_sql
LEFT JOIN sys.dm_exec_connections AS blocker_conn
       ON blocker_conn.session_id = blocked.blocking_session_id
OUTER APPLY sys.dm_exec_sql_text(blocker_conn.most_recent_sql_handle) AS blocker_sql
WHERE blocked.blocking_session_id <> 0;
```

**The head of the chain is what matters.** Session C waits on B waits on A — killing C and B changes nothing. Find A.

Common causes of long blocking:
- A transaction left open by an application that didn't commit
- A missing index causing a scan that locks far more than needed
- Someone left a query window open in SSMS with an uncommitted `BEGIN TRAN` *(this happens constantly)*
- Lock escalation from an unbatched bulk operation

---

## Deadlock prevention

### The single most effective rule

**Access objects in a consistent order everywhere.**

```
Proc A: update Orders, then Customers
Proc B: update Customers, then Orders     ← deadlock waiting to happen
```

Pick an order — alphabetical, or parent-before-child — and follow it in every procedure. This eliminates the majority of deadlocks outright.

### Other measures

- **Shorter transactions.** Less time holding locks = less overlap.
- **Cover your indexes.** A scan locks far more than a seek, dramatically widening the window for conflict.
- **`UPDLOCK` on read-then-write.** If you `SELECT` a row and then `UPDATE` it, take the update lock during the select.
- **RCSI**, which removes reader/writer conflicts entirely (writer/writer deadlocks remain).
- **Retry logic.** Deadlocks are normal in busy systems. Error 1205 is retryable — catch it and retry with a small random backoff.

```sql
DECLARE @Attempt int = 0;
WHILE @Attempt < 3
BEGIN
    BEGIN TRY
        BEGIN TRAN;
            -- work
        COMMIT;
        BREAK;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK;
        IF ERROR_NUMBER() = 1205 AND @Attempt < 2
        BEGIN
            SET @Attempt += 1;
            WAITFOR DELAY '00:00:00.250';
        END
        ELSE THROW;
    END CATCH
END
```

### Capturing deadlock details

The `system_health` Extended Events session captures deadlock graphs by default — they're already being recorded, right now, with no setup:

```sql
SELECT XEvent.query('.') AS DeadlockGraph
FROM (
    SELECT CAST(target_data AS xml) AS TargetData
    FROM   sys.dm_xe_session_targets AS st
    JOIN   sys.dm_xe_sessions AS s ON s.address = st.event_session_address
    WHERE  s.name = 'system_health' AND st.target_name = 'ring_buffer'
) AS Data
CROSS APPLY TargetData.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]')
        AS XEventData(XEvent);
```

The graph shows both processes, both resources, and which statements were involved — everything you need to find the ordering conflict.

---

## Application locks

A named mutex you control, unrelated to any table.

```sql
DECLARE @rc int;
EXEC @rc = sp_getapplock
     @Resource    = 'RefreshMasterSales:2025',
     @LockMode    = 'Exclusive',
     @LockOwner   = 'Transaction',
     @LockTimeout = 30000;

IF @rc < 0
BEGIN
    ROLLBACK;
    THROW 50001, 'Another refresh is already running for this slice.', 1;
END
```

**Return codes:** `0` acquired · `1` acquired after waiting · `-1` timeout · `-2` cancelled · `-3` deadlock victim · `-999` parameter error. Anything `< 0` is a failure.

**`@LockOwner = 'Transaction'`** releases automatically at commit or rollback. Prefer this over `'Session'`, which requires an explicit `sp_releaseapplock` and leaks if the connection is pooled and reused.

**Use for:** "only one instance of this job at a time." **Name the lock by slice**, not by procedure — refreshing 2024 and 2025 concurrently should be allowed.

---

## Optimistic concurrency

For "did anyone else change this row while I was looking at it?"

```sql
-- read
SELECT ProductId, Name, Price, RowVer FROM ref.Product WHERE ProductId = @Id;

-- write, only if unchanged
UPDATE ref.Product
SET    Name = @Name, Price = @Price
WHERE  ProductId = @Id AND RowVer = @OriginalRowVer;

IF @@ROWCOUNT = 0
    THROW 50002, 'This record was modified by another user. Reload and retry.', 1;
```

`rowversion` changes automatically on every update, so a mismatch means someone else got there first. No locks held between read and write — which is exactly what you want for a web application where the "transaction" spans a user thinking about a form for four minutes.

The alternative — holding a lock across a user interaction — is pessimistic concurrency, and it is essentially always wrong for web apps.

---

## Practical guidance

**For your refresh procedure:**
- `SET XACT_ABORT ON`, wrap in `TRY/CATCH`, `THROW` on failure
- `sp_getapplock` named by slice
- Do slow preparation *outside* the transaction; open it only for the delete + insert
- If it blocks readers, enable RCSI — that's the fix, not `NOLOCK` on the readers

**For your website:**
- Read from views; RCSI means those reads never block
- Never open a transaction spanning a user interaction
- Use `rowversion` for optimistic concurrency on any editable data
- Parameterize everything (SQL injection, and plan reuse)

**Instance-wide:**
- Turn on RCSI unless you have a specific reason not to
- Turn on Query Store
- Size tempdb for the version store
- Know how to find the head of a blocking chain before you need to at 2am

---

[← Back to index](README.md) · [Next: T-SQL Cookbook →](06-tsql-cookbook.md)
