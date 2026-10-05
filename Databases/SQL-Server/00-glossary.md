# 00 — Glossary

[← Back to index](README.md)

Every term grouped by what it's for rather than alphabetically, because the grouping *is* part of the learning. Alphabetical index at the bottom.

---

## Contents

- [Correctness under concurrency](#correctness-under-concurrency)
- [Reliability patterns](#reliability-patterns)
- [Data modeling](#data-modeling)
- [Performance](#performance)
- [Query semantics](#query-semantics)
- [Pipelines & ETL](#pipelines--etl)
- [Operations](#operations)
- [Change management](#change-management)
- [Security](#security)
- [Alphabetical index](#alphabetical-index)

---

## Correctness under concurrency

**Transaction**
A unit of work that either fully happens or fully doesn't. `BEGIN TRAN` … `COMMIT` / `ROLLBACK`. Every single statement in SQL Server is already a transaction implicitly; explicit transactions group several statements into one all-or-nothing unit.

**ACID**
The four guarantees a transaction provides:
- **Atomicity** — all or nothing. A failure halfway leaves no partial work.
- **Consistency** — constraints and rules hold at the end of the transaction.
- **Isolation** — concurrent transactions don't see each other's in-progress mess.
- **Durability** — once committed, it survives a power loss. Delivered by the transaction log being written before the data pages.

**Isolation level**
The dial controlling how much of other transactions' uncommitted work you can see, traded against how much blocking you cause. Ordered from least to most isolated: `READ UNCOMMITTED`, `READ COMMITTED` (default), `REPEATABLE READ`, `SERIALIZABLE`. Plus `SNAPSHOT`, which sits outside the ladder by using row versions instead of locks. → [full treatment](05-concurrency.md#isolation-levels)

**Dirty read**
Reading data another transaction has written but not committed — data that may be rolled back and never have existed. Only possible at `READ UNCOMMITTED` / `NOLOCK`.

**Non-repeatable read**
Reading the same row twice within one transaction and getting different values, because someone committed a change in between.

**Phantom read**
Running the same range query twice within one transaction and getting *additional* rows, because someone inserted into the range.

**Lost update**
Two transactions read the same value, both compute a new one from it, both write. The second silently overwrites the first. The classic reason to avoid read-modify-write without proper locking or version checking.

**Blocking**
One transaction waits for a lock another holds. This is **normal and healthy** — it's the mechanism that makes isolation work. Blocking resolves itself when the blocker commits. Only a problem when it lasts a long time.

**Deadlock**
Two transactions each hold a lock the other needs. Neither can proceed and neither will ever release. SQL Server's deadlock monitor detects the cycle and kills one — the **deadlock victim** — with error 1205, rolling it back.

> **Blocking is traffic. Deadlock is a crash.** Confusing them sends you down entirely the wrong debugging path. Blocking → find the long-running transaction. Deadlock → look at lock ordering in your code.

**Lock escalation**
When a statement acquires too many row or page locks (~5,000 by default), SQL Server trades them all for a single table lock to save memory. Efficient for the engine, brutal for everyone else trying to use that table. A reason to batch large deletes.

**Optimistic concurrency**
Assume conflicts are rare. Don't lock; detect at write time whether someone else changed the row (typically via a `rowversion` check) and fail or retry if so.

**Pessimistic concurrency**
Assume conflicts happen. Take locks upfront to prevent them. SQL Server's default behavior.

**Race condition**
Behavior depends on the relative timing of two concurrent processes. In database work, usually: two runs of the same job overlapping, or a check-then-act sequence where something changed between the check and the act.

**Application lock (`sp_getapplock`)**
A named mutex you control, unrelated to any table. The tool for "only one instance of this procedure may run at a time." → [pattern](03-refresh-patterns.md#serializing-with-app-locks)

**Atomic operation**
Indivisible from the outside — observers see it as either not-started or complete, never partial.

---

## Reliability patterns

**Idempotent**
Running it twice produces the same result as running it once. Running it five times, same. This is the property that makes retry safe, and retry is the foundation of every reliable pipeline.

- `DELETE the slice; INSERT the slice;` → idempotent ✅
- `INSERT the slice;` → not idempotent (duplicates on second run) ❌
- `UPDATE t SET counter = counter + 1` → not idempotent ❌
- `UPDATE t SET status = 'Done'` → idempotent ✅

> **The single most valuable property in data engineering.** It converts "the job died halfway through, what state are we in, what do I do now?" into "run it again." That is an enormous reduction in operational anxiety.

**Deterministic**
Same inputs always produce the same output. `GETDATE()`, `NEWID()`, and `@@SPID` are non-deterministic — which is why they can't appear in a persisted computed column, an indexed view, or an index key. Deterministic transforms are what make **replay** possible.

**Side effect**
Anything a piece of code changes outside its return value — rows written, emails sent, files created, APIs called. Side effects are precisely what makes retrying dangerous. Pure computation is always safe to retry; side-effecting code is only safe to retry if it's idempotent.

**At-least-once delivery**
The guarantee most real systems actually provide: your message/row/event will arrive, possibly more than once. Message queues, SSIS retries, and webhook senders all work this way.

**Exactly-once delivery**
The guarantee everyone wants and almost nobody has. Genuinely hard in distributed systems.

> **The practical answer to at-least-once is idempotence.** Stop trying to prevent the duplicate. Make the duplicate harmless. This reframing solves the problem instead of fighting it.

**Watermark**
The marker recording "I have processed everything up to here." A max timestamp, a `rowversion` value, a Change Tracking version number, a max ID. The core of any incremental load. → [pattern](03-refresh-patterns.md#watermark-based-incremental)

**Backfill**
Running a process over historical data to fill a gap or apply a bug fix retroactively. Cheap when your refresh takes a period parameter; a nightmare when it only ever processes "today."

**Replay**
Re-running the transform from raw data to reproduce (or correct) a result. Only possible when raw is immutable and the transform is deterministic. **Preserving the ability to replay is the reason master tables should be fully derivable.**

**Source of truth**
The one place a fact authoritatively lives. Everything else is a derived copy and is disposable. When two systems both claim to be authoritative for the same fact, you get the "which number is right?" meeting, and there's no technical answer to it — only an organizational one.

**Immutable**
Never modified after creation. Append-only. Raw landing tables should be as close to immutable as practical, because immutability is what makes replay trustworthy.

**Fail fast**
Stop at the first error rather than continuing in an unknown state. `SET XACT_ABORT ON` plus `TRY/CATCH` plus `THROW`. The alternative — swallowing errors and continuing — produces silently wrong data, which is far worse than a loud failure.

**Poison record**
A single bad row that kills the whole batch every time you retry it. Handle by quarantining bad rows to a rejects table rather than letting one row block the pipeline forever.

**Circuit breaker**
Stop attempting an operation after repeated failures rather than hammering a failing dependency. Relevant when your pipeline pulls from an external system.

**Dead letter queue / rejects table**
Where rows that couldn't be processed go, so they can be inspected without blocking everything else.

---

## Data modeling

**Grain**
What one row represents, stated as a sentence: *"One row per year, per product, per region."*

> **Undeclared grain is the number-one cause of double-counted totals.** When you join two tables at different grains and sum, you get numbers that are wrong in a way nobody catches for months. Write the grain down. Enforce it with a `UNIQUE` constraint.

**Cardinality**
Two distinct meanings, both common:
1. *Between tables* — one-to-one, one-to-many, many-to-many.
2. *Of a column* — how many distinct values it has. High cardinality = many distinct values (e.g. email address). Low cardinality = few (e.g. a status flag).

**Selectivity**
What fraction of rows a predicate returns. Highly selective = returns few rows = index is worth using. A predicate matching 80% of the table won't use an index even if one exists, because scanning is cheaper than seeking that many times.

**Primary key**
The column(s) uniquely identifying a row. Cannot be NULL. Creates a unique index (clustered by default).

**Surrogate key**
A meaningless generated identifier — `IDENTITY`, a sequence value, a GUID. Carries no business meaning.

**Natural key**
Real-world data used as the key — part number, SSN, email, order number.

> Surrogates win in practice, because natural keys turn out to change (people marry, SKUs get renumbered), repeat (two customers, same name), or arrive typo'd. Keep the natural key as a `UNIQUE` constraint alongside the surrogate PK — you get stable joins *and* enforced business uniqueness.

**Composite key**
A key made of multiple columns together. Column order matters for the index it creates.

**Foreign key**
A column constrained to match a key value in another table.

**Referential integrity**
The guarantee that foreign key relationships hold — no child row points at a parent that doesn't exist.

**Orphan record**
A child row whose parent has vanished. Exactly what a foreign key prevents.

**Normalization**
Organizing data so each fact lives in exactly one place.
- **1NF** — no repeating groups; each cell holds one value. (No `Phone1, Phone2, Phone3`.)
- **2NF** — 1NF plus: no non-key column depends on only *part* of a composite key.
- **3NF** — 2NF plus: no non-key column depends on another non-key column.

In practice the working rule is simply: **don't store the same fact in two places**, because they will eventually disagree and you won't know which is right.

**Denormalization**
Deliberately violating normalization — duplicating data — to make reads faster or simpler. Your master/mart tables are denormalized on purpose. This is fine *specifically because* they're derived and rebuildable; the duplication can't drift from the source since it's regenerated from it.

**Star schema**
The standard reporting shape: a central **fact** table surrounded by **dimension** tables.

**Fact table**
Measurements and metrics. Many rows, mostly numbers and foreign keys. Your `mart.MasterSales`.

**Dimension table**
Descriptive attributes you slice and filter by. Few rows, mostly text. Your `ref.ProductMap`, `ref.RegionMap`.

**Snowflake schema**
A star schema where dimensions are themselves normalized into sub-dimensions. More joins, less duplication. Usually not worth it for reporting.

**Conformed dimension**
A dimension shared consistently across multiple fact tables, so "region" means the same thing everywhere. What lets you compare across subject areas.

**Slowly Changing Dimension (SCD)**
How you handle a dimension attribute changing over time:
- **Type 0** — never changes.
- **Type 1** — overwrite. History is lost.
- **Type 2** — insert a new row with `ValidFrom` / `ValidTo` / `IsCurrent`. History preserved.
- **Type 3** — keep a "previous value" column. Limited, rarely used.

> **Decide early.** Discovering after a year of Type 1 that someone needs "what category was this product in during 2024?" is genuinely painful — the history simply doesn't exist and can't be reconstructed. Ask the question up front: *will anyone ever need to know what this looked like in the past?*

**Junction table / bridge table / associative entity**
Resolves a many-to-many into two one-to-manys. `StudentCourse(StudentId, CourseId)`.

**Cartesian product**
Every row of one table joined to every row of another, because the join condition was missing or wrong. 10,000 × 10,000 = 100 million rows and a server that stops answering.

**Fan-out / fan trap**
Joining a one-to-many relationship then aggregating, causing the "one" side's values to be counted once per matching child row. The most common source of inflated totals.

**Degenerate dimension**
A dimension attribute stored directly on the fact table because it has no other attributes — an order number, an invoice number.

**Surrogate key pipeline**
The lookup step in ETL that translates natural keys from the source into your surrogate keys.

**Entity / attribute / relationship**
The three primitives of data modeling: a thing, a property of that thing, a connection between things. Tables, columns, foreign keys.

**ERD (Entity-Relationship Diagram)**
The picture of your tables and their relationships.

**Data type precedence**
The rules determining which type wins when types mix in an expression. Causes silent implicit conversions — a frequent hidden performance killer. → [see](04-performance.md#implicit-conversion)

**Nullable**
Whether a column permits NULL. `NOT NULL` is both a correctness guarantee and a performance hint. Default to `NOT NULL` and justify exceptions, not the reverse.

---

## Performance

**Sargable** (Search ARGument ABLE)
A predicate that an index can *seek* on, because the indexed column appears bare on one side of the comparison.

```sql
WHERE LoadDate >= @Start AND LoadDate < @End   -- sargable   ✅
WHERE YEAR(LoadDate) = @Year                   -- NOT sargable ❌
WHERE Name LIKE 'Smi%'                         -- sargable   ✅
WHERE Name LIKE '%mith'                        -- NOT sargable ❌
```

> **The single highest-value performance concept.** Wrapping a column in a function, or applying arithmetic to it, forces a full scan no matter what indexes exist. → [full treatment](04-performance.md#sargability)

**Seek**
Navigating the index B-tree directly to the rows you want. Cost proportional to rows *returned*.

**Scan**
Reading everything and filtering. Cost proportional to rows *in the table*. Not always bad — scanning a 200-row lookup table is correct — but a scan on a large table where you expected a seek is the classic red flag.

**Clustered index**
The physical sort order of the table. **The table *is* its clustered index** — the data pages are the index leaf level. Only one per table.

**Nonclustered index**
A separate structure holding the key columns sorted, plus a pointer back to the base row.

**Heap**
A table with no clustered index. Data is unordered. Almost always a mistake for anything you query repeatedly.

**Covering index**
An index containing every column a query needs, so the query never touches the base table. The gold standard for a hot query.

**Included columns (`INCLUDE`)**
Extra columns carried in the index leaf level only — not part of the sort key, not usable for seeking, but available to satisfy the `SELECT` list. How you make an index covering without bloating the key.

**Key lookup / RID lookup**
The index found the row but not every column the query wanted, so it jumps to the base table for the rest. Cheap once. Catastrophic 500,000 times. Usually fixed with `INCLUDE`.

**Composite index column order**
The leading column is the one you can seek on. An index on `(A, B)` helps `WHERE A = 1`, and `WHERE A = 1 AND B = 2`, but generally **not** `WHERE B = 2` alone. Order by: equality predicates first, then range predicates, then includes.

**Execution plan**
The strategy the optimizer chose to run your query. **Estimated** plan is what it intends; **actual** plan includes real row counts. Learning to read these is the highest-leverage SQL Server skill there is.

**Query optimizer**
The cost-based engine that picks a plan. It doesn't find the *best* plan — it finds a good-enough plan quickly, then stops looking.

**Statistics**
Histograms describing the distribution of values in a column, used by the optimizer to guess how many rows a predicate will return.

**Cardinality estimate**
That guess. Stale or missing statistics → bad estimate → bad plan choice → mystery slowness. A large gap between estimated and actual rows in a plan is the number-one diagnostic signal.

**Parameter sniffing**
The optimizer compiles a plan for the *first* parameter value it sees, then caches and reuses it for all subsequent values. Excellent when data is uniformly distributed. Disastrous when one customer has 4 million rows and everyone else has 40 — whoever runs first determines everyone's plan.

Fixes: `OPTION (RECOMPILE)`, `OPTIMIZE FOR`, local variable copies, or splitting into separate procedures. SQL Server 2022's Parameter Sensitive Plan optimization handles some cases automatically.

**Plan cache**
Where compiled plans are stored for reuse. Compilation is expensive, so reuse is good — except when the cached plan is wrong for the current parameters.

**Recompile**
Discarding the cached plan and building a new one for the current parameter values. Costs a few milliseconds of CPU; often saves minutes.

**RBAR** — "Row By Agonizing Row"
Processing one row at a time (cursors, `WHILE` loops, scalar functions per row) when a single set-based statement would do the whole thing at once. The defining beginner-to-intermediate mistake in SQL, and often a 100× difference.

**Set-based thinking**
Describing *what* you want as an operation on entire sets, and letting the engine decide how. The opposite of RBAR, and the actual mental shift that separates people who are fast at SQL from people who aren't.

**Cursor**
The explicit row-at-a-time iteration construct. Occasionally necessary (administrative loops over databases, calling a proc per row). Usually a sign the problem hasn't been thought through set-wise.

**Implicit conversion**
SQL Server silently converting a data type to compare two values — e.g. comparing an `nvarchar` parameter to a `varchar` column. This **destroys sargability** and is invisible unless you look for the warning in the plan. A top-five cause of "the index exists but isn't being used."

**Fragmentation**
Index pages out of physical order due to inserts and updates. Matters far less than the internet implies on modern SSD storage. Don't build a nightly rebuild job before you have a measured problem.

**Fill factor**
How full index pages are packed when built. Lower = more room for inserts before page splits = more space used. Leave at default until you have evidence.

**Page split**
An insert doesn't fit on a page, so the page divides in two. Causes fragmentation and log activity.

**Spill to tempdb**
The engine underestimated memory needed for a sort or hash, ran out, and wrote to disk. Visible as a warning in the actual execution plan. Almost always traces back to a bad cardinality estimate.

**Missing index suggestion**
The optimizer's note that an index might have helped. **Treat as a hint, not an instruction** — it ignores existing indexes, suggests overly wide includes, and doesn't consider write cost. Use it as a starting point for thinking, not a script to run.

**Index maintenance cost**
Every index makes writes slower and takes space. Indexes are not free. The right number is "as few as possible while queries are fast enough."

**Wait statistics**
What the engine spent time waiting *on* — CPU, disk, locks, memory grants. The starting point for "why is the server slow?" as opposed to "why is this query slow?"

**Batching**
Splitting a huge DML statement into chunks to avoid lock escalation, log growth, and long blocking. Delete 100 million rows in 10,000-row batches, not one statement.

---

## Query semantics

**Three-valued logic**
SQL has `TRUE`, `FALSE`, and `UNKNOWN`. Any comparison involving NULL yields UNKNOWN, and `WHERE` only keeps rows evaluating to TRUE.

```sql
NULL = NULL       -- UNKNOWN, not TRUE
NULL <> 'A'       -- UNKNOWN, not TRUE
WHERE Status <> 'Closed'   -- silently EXCLUDES rows where Status IS NULL
```

> This bites everyone, forever. It's the reason `IS NULL` exists as separate syntax, and the reason `NOT IN` with a NULL in the list returns nothing at all.

**`NOT IN` vs `NOT EXISTS`**
`NOT IN (subquery)` returns zero rows if the subquery contains a single NULL. `NOT EXISTS` handles NULLs correctly. **Default to `NOT EXISTS`.**

**Inner join**
Keeps only rows with a match on both sides. Silently drops unmatched rows — the classic way a total quietly comes up short and nobody notices for a month.

**Left / right outer join**
Keeps all rows from one side, filling NULLs where there's no match.

**Full outer join**
Keeps everything from both sides. Useful for reconciliation: "what's in A but not B, and vice versa, in one pass."

**Cross join**
Every combination. Deliberate Cartesian product. Legitimately useful for generating calendar grids or filling gaps in a series.

**Anti-join**
Finding what's *missing*: `NOT EXISTS`, or `LEFT JOIN ... WHERE right.key IS NULL`. Your data-quality workhorse — "which raw rows have no matching entry in the product map?"

**Semi-join**
"Does at least one match exist?" — `EXISTS` / `IN`. Doesn't multiply rows the way a join does, which matters when the other side has duplicates.

**Self-join**
Joining a table to itself. Hierarchies, comparing rows to other rows in the same table.

**CTE (Common Table Expression)**
The `WITH x AS (...)` prefix. **Naming, not materialization** — it's textual substitution. Referencing a CTE twice executes it twice. Not a performance feature.

**Recursive CTE**
A CTE referencing itself — for hierarchies, org charts, bill-of-materials, date series generation.

**Derived table**
A subquery in the `FROM` clause. Functionally like a CTE, different syntax.

**Correlated subquery**
A subquery referencing the outer query's columns, conceptually evaluated per outer row. Often (not always) optimized into a join.

**Window function**
`OVER (PARTITION BY ... ORDER BY ...)`. Aggregates and rankings **without collapsing rows**. Running totals, row numbering, comparing each row to the previous one, percent-of-group.

> Once these click, an entire category of problems stops requiring self-joins or cursors. Worth deliberate study — probably the highest return per hour of any T-SQL topic. → [patterns](06-tsql-cookbook.md#window-functions)

**`ROW_NUMBER` / `RANK` / `DENSE_RANK`**
Ranking functions. `ROW_NUMBER` always unique; `RANK` leaves gaps after ties (1,1,3); `DENSE_RANK` doesn't (1,1,2).

**`APPLY` (`CROSS APPLY` / `OUTER APPLY`)**
Joins to something computed *per row of the left side*. The tool for "top N per group" and for calling a table-valued function with each row's values. `CROSS` behaves like an inner join, `OUTER` like a left join.

**Predicate**
A condition evaluating to true/false/unknown. What lives in `WHERE`, `ON`, and `HAVING`.

**`WHERE` vs `HAVING`**
`WHERE` filters rows before grouping. `HAVING` filters groups after aggregation. Putting a non-aggregate filter in `HAVING` works but processes more rows than necessary.

**Logical query processing order**
The order SQL is *evaluated*, which differs from how it's written:

```
FROM → ON → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → TOP/OFFSET
```

This explains why you can't reference a `SELECT` alias in `WHERE` (SELECT hasn't happened yet) but can in `ORDER BY` (it has).

**DDL / DML / DCL / TCL**
- **DDL** — Data Definition Language: `CREATE`, `ALTER`, `DROP`
- **DML** — Data Manipulation Language: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE`
- **DCL** — Data Control Language: `GRANT`, `REVOKE`, `DENY`
- **TCL** — Transaction Control: `BEGIN`, `COMMIT`, `ROLLBACK`

**`TRUNCATE` vs `DELETE`**

| | `TRUNCATE` | `DELETE` |
|---|---|---|
| Mechanism | Deallocates pages | Removes row by row |
| Logging | Minimal | Full |
| Speed on full table | Very fast | Slow |
| `WHERE` clause | No | Yes |
| Resets `IDENTITY` | Yes | No |
| Fires triggers | No | Yes |
| Rollback-able | Yes (it's transactional) | Yes |

Truncate for "empty this staging table." Delete for anything selective.

**`OUTPUT` clause**
Any `INSERT`/`UPDATE`/`DELETE`/`MERGE` can emit the affected rows into a table or back to the client. The clean way to log exactly what a statement touched without a second query.

**`MERGE`**
Upsert in one statement. Reads elegantly. Has a long tail of documented correctness and concurrency bugs, and requires `HOLDLOCK` to be safe under concurrency. **Fine for small controlled cases (seed data, a dirty-flag queue); avoid for high-volume concurrent loads** where explicit `UPDATE` then `INSERT` is safer.

**Upsert**
Insert if new, update if existing. The operation `MERGE` implements.

**Idempotent DDL**
`CREATE OR ALTER` for procs/views/functions; `IF OBJECT_ID(...) IS NULL` guards for tables. Makes deployment scripts re-runnable.

**Deprecated: `NOLOCK`**
`WITH (NOLOCK)` = `READ UNCOMMITTED` on one table. Can return rows twice, skip rows entirely, and read data that gets rolled back. It is not a performance feature, it's a correctness trade. Almost never the right answer — enable RCSI instead.

---

## Pipelines & ETL

**ETL** — Extract, Transform, Load
Transform data in flight, load the finished result. The classic SSIS data-flow model.

**ELT** — Extract, Load, Transform
Land raw data untouched, then transform inside the database with SQL. **The modern default**, because raw stays available for replay and because SQL engines transform faster than row-by-row pipeline components.

**Staging / landing zone**
Where raw extracted data sits before transformation. Your `raw` schema. Should be dumb: no business logic, no cleanup, no joins.

**Data lineage**
The traceable path from a number on a report back to the source rows that produced it. When someone asks "where did this come from?", lineage is whether you have an answer or a shrug.

**Full load**
Reprocess everything every time. Simple, naturally idempotent, self-healing. **Start here.** Only move on when it demonstrably stops finishing in time.

**Incremental load**
Process only what changed since the last run. Faster, more complex, and vulnerable to missing changes if your change-detection is imperfect.

**Late-arriving data**
Rows for a period you already processed showing up afterward. The reason a "just process yesterday" job eventually produces wrong historical numbers — and the reason your refresh procedure should take a period parameter so you can reprocess any slice on demand.

**Late-arriving dimension / early-arriving fact**
A fact row referencing a dimension member that doesn't exist yet. Handle with an "unknown member" placeholder row rather than dropping the fact.

**Schema drift**
The source system changed shape — new column, renamed column, widened type — and nobody told you. SSIS is particularly brittle here because it binds column metadata at design time.

**Change Data Capture (CDC)**
SQL Server feature capturing full before/after row images into change tables by reading the transaction log. Heavyweight. Use when you need the *history of values*.

**Change Tracking (CT)**
Lightweight sibling of CDC. Tells you *which rows* changed since a version number, not what they changed to. Usually exactly what an incremental ETL needs.

**Temporal table (system-versioned)**
A table that automatically maintains its own history in a paired history table. Query with `FOR SYSTEM_TIME AS OF '2025-03-01'`. Excellent for audit questions.

**`rowversion` (formerly `timestamp`)**
An auto-incrementing binary value, unique database-wide, that changes whenever the row changes. The poor man's watermark, and a good optimistic-concurrency check. Not a date — the name `timestamp` is a historical mistake.

**Slice**
A logical partition of data you process as a unit — a year, a month, a source system, a combination. Your refresh procedure operates on slices.

**Partition switching**
Instantly swapping a prepared table into a partitioned table as a partition. Metadata-only, effectively instant regardless of size. The advanced answer to "the full rebuild takes too long." Requires Enterprise Edition pre-2016 SP1.

**Watermark table**
A small table storing the last-processed position per source. Where you persist your incremental state.

**Reconciliation**
Comparing counts or sums between source and destination to prove the load was complete. The step everyone skips and later wishes they hadn't.

**Data quality dimensions**
Completeness, accuracy, consistency, timeliness, uniqueness, validity. Useful checklist when defining what "good data" means for a given feed.

**Orchestration**
Deciding what runs when, in what order, with what dependencies, and what happens on failure. SQL Agent, SSIS master packages, or a dedicated scheduler.

**Idempotent load key**
The natural identifier that lets you detect you already loaded this batch. Filename, batch ID, source hash. Prevents double-loading the same file.

---

## Operations

**Recovery model**
Determines how the transaction log behaves and what restores are possible.

| Model | Log behavior | Point-in-time restore | Use for |
|---|---|---|---|
| `SIMPLE` | Auto-truncates on checkpoint | ❌ | Dev, staging, anything rebuildable |
| `FULL` | Retained until log backup | ✅ | Production with real RPO needs |
| `BULK_LOGGED` | Minimally logs bulk ops | Partial | Temporary, during huge loads |

> **The most common way a small shop's SQL Server falls over**: someone sets `FULL` recovery (or it's the default from the model database) and never configures log backups. The log grows until the disk fills and the database stops accepting writes. If you're in `FULL`, you *must* back up the log on a schedule.

**Transaction log**
The write-ahead record of every change. Durability, rollback, and point-in-time recovery all come from here. Written before data pages — that's what makes committed transactions survive a crash.

**Full / differential / log backup**
- **Full** — the whole database.
- **Differential** — everything changed since the last *full*.
- **Log** — all transactions since the last *log* backup. Required for point-in-time restore.

A typical chain: weekly full, nightly differential, log every 15 minutes.

**RPO — Recovery Point Objective**
How much data you can afford to lose, measured in time. "15 minutes" means your log backups run every 15 minutes.

**RTO — Recovery Time Objective**
How long you can afford to be down. Determines whether you need failover clustering / Availability Groups or whether restoring from backup is acceptable.

> **These two numbers determine your entire backup and HA strategy, and they're a business decision, not a technical one.** As the owner, you set them — nobody else will do it for you. Write them down.

**Point-in-time restore**
Restoring to an exact moment — e.g. 14:32:07, just before someone ran an `UPDATE` without a `WHERE`. Requires `FULL` recovery plus an unbroken log backup chain.

**A backup you have never restored is not a backup.**
Not a definition, but it belongs here. Schedule a restore test. Put it on the calendar. The number of shops that discover their backups were unusable *during an outage* is not small.

**tempdb**
Shared scratch space for temp tables, table variables, sorts, hash operations, and row versions. System-wide, recreated on restart. A common bottleneck when misconfigured (should be multiple equally-sized data files on fast storage).

**Checkpoint**
Flushing dirty pages from memory to disk. Bounds crash recovery time.

**Buffer pool**
SQL Server's in-memory cache of data pages. Nearly all your RAM. "SQL Server is using all the memory" is usually correct behavior, not a problem.

**Max server memory**
The setting you *must* configure, because the default lets SQL Server take everything and starve the OS.

**DTC — Distributed Transaction Coordinator**
Coordinates transactions spanning multiple instances. Slow, fragile, an extra service to keep alive. A good reason to keep raw and master in the same database.

**Linked server**
A configured connection to another database server, queryable via four-part names. Convenient and frequently a performance disaster, because predicates often don't get pushed to the remote side. → [guidance](09-integration.md#linked-servers)

**High availability (HA) vs Disaster recovery (DR)**
HA = surviving a component failure with minimal downtime (Availability Groups, failover clustering). DR = surviving loss of the whole site (offsite backups, geo-replication). Different problems, different solutions, both driven by RTO.

**Maintenance plan**
Scheduled housekeeping: backups, index maintenance, statistics updates, integrity checks.

**`DBCC CHECKDB`**
Verifies physical and logical integrity of the database. **Run it on a schedule.** Corruption discovered six months late, after every backup containing good data has aged out, is unrecoverable.

**Instance vs database**
An instance is the SQL Server service. A database is one container within it. Settings live at both levels, and confusing which is which causes a lot of wasted troubleshooting.

**Collation**
Rules for character comparison and sorting, including case and accent sensitivity. Set at instance, database, and column level. Mismatched collations in a join cause errors or silent performance loss.

**Edition differences**
Express (free, 10 GB limit, no Agent), Standard, Enterprise, Developer (free, full Enterprise features, non-production only). Use Developer Edition for your dev box — it's free and complete.

---

## Change management

**Migration**
A scripted, ordered, append-only schema change. Once one has run in production, you never edit it — you write another.

**State-based deployment**
You declare the *desired end state* (a set of `CREATE TABLE` scripts) and a tool computes the difference against the target and generates the `ALTER` statements. SSDT / DACPAC, Redgate SQL Compare.

**Migration-based deployment**
You write each change step yourself in order, and the tool tracks which have been applied. Flyway, DbUp, Liquibase.

> Both are legitimate. State-based means less manual work and a diff you should always read before applying. Migration-based means more typing and total control. **Pick one and be consistent** — mixing them is where the pain is.

**Drift**
Production no longer matches source control, because someone changed it by hand. Schema Compare finds it. Culture prevents it.

**DACPAC**
A deployable package of database schema produced by an SSDT project.

**Idempotent script**
Safe to run repeatedly. `CREATE OR ALTER`, `IF NOT EXISTS` guards, `MERGE` for seed data.

**Source of truth (for schema)**
The git repo, not the production database. If a new environment can't be built by running your scripts, the repo isn't actually authoritative and you're one incident away from finding that out.

**Environment promotion**
Dev → Test → Prod, with the same scripts applied in each. If prod gets changes that dev didn't, you have drift by construction.

**Blue-green / rolling deployment**
Deploying without downtime by running old and new side by side. Relevant when schema changes must be backward compatible for a period — add columns before removing old ones, never rename in place.

**Backward-compatible change**
A change that doesn't break existing consumers: adding a nullable column, adding a table, adding an index. Contrast with breaking changes: dropping or renaming a column, narrowing a type, adding a `NOT NULL` column without a default.

**Views as a contract**
Exposing views rather than tables to consumers, so you can restructure underlying storage freely as long as the view's shape stays stable. The main reason the view layer earns its keep.

---

## Security

*(Brief — enough to not create obvious problems.)*

**Principal**
Anything that can be granted permission: a login, a user, a role.

**Login vs user**
A **login** authenticates at the instance level. A **user** is that login's identity inside a specific database. One login maps to a user in each database it can access.

**Role**
A named bundle of permissions. Grant to roles, add principals to roles — never grant directly to individuals, or permissions become impossible to audit.

**Principle of least privilege**
Grant the minimum needed. The website's account should have `SELECT` on `mart.vw_*` and `EXECUTE` on specific procs — not `db_datareader` on everything, and certainly not `db_owner`.

**Ownership chaining**
When objects share an owner, permission is checked only on the object accessed, not the objects it references. This is why granting `EXECUTE` on a procedure works without granting access to underlying tables — and it's the mechanism that makes the proc-and-view layer a genuine security boundary. Breaks across databases.

**SQL injection**
Building SQL by concatenating user input, letting an attacker inject their own statements. Prevented by **parameterized queries**, never by escaping quotes yourself. Inside dynamic SQL, use `sp_executesql` with parameters and `QUOTENAME()` for identifiers.

**Dynamic SQL**
SQL built as a string and executed. Sometimes necessary. Always use `sp_executesql` with parameters rather than `EXEC(@sql)` with concatenation.

**Service account**
The identity a service runs as. Prefer a **gMSA** (group Managed Service Account) — Active Directory rotates the password automatically, so nothing breaks when a password expires.

**Transparent Data Encryption (TDE)**
Encrypts data files at rest. Protects against someone stealing the `.mdf` or a backup file. Doesn't protect against anyone with database access.

**Always Encrypted**
Column-level encryption where the server never sees plaintext. For genuinely sensitive columns.

**Row-level security**
Predicate-based filtering applied automatically so different users see different rows of the same table.

---

## Alphabetical index

**A** — [ACID](#correctness-under-concurrency) · [Anti-join](#query-semantics) · [APPLY](#query-semantics) · [Application lock](#correctness-under-concurrency) · [At-least-once](#reliability-patterns) · [Atomicity](#correctness-under-concurrency)

**B** — [Backfill](#reliability-patterns) · [Batching](#performance) · [Blocking](#correctness-under-concurrency) · [Bridge table](#data-modeling) · [Buffer pool](#operations)

**C** — [Cardinality](#data-modeling) · [Cardinality estimate](#performance) · [Cartesian product](#data-modeling) · [CDC](#pipelines--etl) · [Change Tracking](#pipelines--etl) · [Checkpoint](#operations) · [Circuit breaker](#reliability-patterns) · [Clustered index](#performance) · [Collation](#operations) · [Composite key](#data-modeling) · [Conformed dimension](#data-modeling) · [Covering index](#performance) · [CTE](#query-semantics) · [Cursor](#performance)

**D** — [DACPAC](#change-management) · [DBCC CHECKDB](#operations) · [DDL/DML/DCL](#query-semantics) · [Deadlock](#correctness-under-concurrency) · [Degenerate dimension](#data-modeling) · [Denormalization](#data-modeling) · [Derived table](#query-semantics) · [Deterministic](#reliability-patterns) · [Differential backup](#operations) · [Dimension table](#data-modeling) · [Dirty read](#correctness-under-concurrency) · [Drift](#change-management) · [DTC](#operations) · [Dynamic SQL](#security)

**E** — [ELT](#pipelines--etl) · [ERD](#data-modeling) · [ETL](#pipelines--etl) · [Exactly-once](#reliability-patterns) · [Execution plan](#performance)

**F** — [Fact table](#data-modeling) · [Fail fast](#reliability-patterns) · [Fan-out](#data-modeling) · [Fill factor](#performance) · [Foreign key](#data-modeling) · [Fragmentation](#performance) · [Full load](#pipelines--etl)

**G** — [gMSA](#security) · [Grain](#data-modeling)

**H** — [HA vs DR](#operations) · [HAVING](#query-semantics) · [Heap](#performance)

**I** — [Idempotent](#reliability-patterns) · [Immutable](#reliability-patterns) · [Implicit conversion](#performance) · [Included columns](#performance) · [Incremental load](#pipelines--etl) · [Isolation level](#correctness-under-concurrency)

**J** — [Junction table](#data-modeling)

**K** — [Key lookup](#performance)

**L** — [Late-arriving data](#pipelines--etl) · [Lineage](#pipelines--etl) · [Linked server](#operations) · [Lock escalation](#correctness-under-concurrency) · [Logical query processing order](#query-semantics) · [Lost update](#correctness-under-concurrency)

**M** — [Max server memory](#operations) · [MERGE](#query-semantics) · [Migration](#change-management) · [Missing index suggestion](#performance)

**N** — [Natural key](#data-modeling) · [NOLOCK](#query-semantics) · [Non-repeatable read](#correctness-under-concurrency) · [Normalization](#data-modeling) · [NOT EXISTS](#query-semantics)

**O** — [Optimistic concurrency](#correctness-under-concurrency) · [Orchestration](#pipelines--etl) · [Orphan record](#data-modeling) · [OUTPUT clause](#query-semantics) · [Ownership chaining](#security)

**P** — [Page split](#performance) · [Parameter sniffing](#performance) · [Partition switching](#pipelines--etl) · [Phantom read](#correctness-under-concurrency) · [Plan cache](#performance) · [Point-in-time restore](#operations) · [Poison record](#reliability-patterns) · [Predicate](#query-semantics) · [Primary key](#data-modeling) · [Principle of least privilege](#security)

**R** — [Race condition](#correctness-under-concurrency) · [RBAR](#performance) · [Recompile](#performance) · [Reconciliation](#pipelines--etl) · [Recovery model](#operations) · [Recursive CTE](#query-semantics) · [Referential integrity](#data-modeling) · [Replay](#reliability-patterns) · [Role](#security) · [ROW_NUMBER](#query-semantics) · [rowversion](#pipelines--etl) · [RPO / RTO](#operations)

**S** — [Sargable](#performance) · [SCD](#data-modeling) · [Schema drift](#pipelines--etl) · [Scan](#performance) · [Seek](#performance) · [Selectivity](#performance) · [Self-join](#query-semantics) · [Semi-join](#query-semantics) · [Set-based thinking](#performance) · [Side effect](#reliability-patterns) · [Slice](#pipelines--etl) · [Snowflake schema](#data-modeling) · [Source of truth](#reliability-patterns) · [Spill to tempdb](#performance) · [SQL injection](#security) · [Staging](#pipelines--etl) · [Star schema](#data-modeling) · [State-based deployment](#change-management) · [Statistics](#performance) · [Surrogate key](#data-modeling)

**T** — [tempdb](#operations) · [Temporal table](#pipelines--etl) · [Three-valued logic](#query-semantics) · [Transaction](#correctness-under-concurrency) · [Transaction log](#operations) · [TRUNCATE vs DELETE](#query-semantics)

**U** — [Upsert](#query-semantics)

**W** — [Wait statistics](#performance) · [Watermark](#reliability-patterns) · [Window function](#query-semantics)

---

[← Back to index](README.md) · [Next: Architecture →](01-architecture.md)
