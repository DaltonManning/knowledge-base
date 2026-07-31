# 04 — Performance

[← Back to index](README.md) · [← Refresh Patterns](03-refresh-patterns.md) · [Next: Concurrency →](05-concurrency.md)

Why things are slow, and how to tell rather than guess.

---

## Contents

- [Sargability](#sargability)
- [Implicit conversion](#implicit-conversion)
- [Index anatomy](#index-anatomy)
- [Reading execution plans](#reading-execution-plans)
- [Statistics & cardinality estimates](#statistics--cardinality-estimates)
- [Parameter sniffing](#parameter-sniffing)
- [RBAR and set-based thinking](#rbar-and-set-based-thinking)
- [Batching large operations](#batching-large-operations)
- [tempdb](#tempdb)
- [Where to start when something is slow](#where-to-start-when-something-is-slow)

---

## Sargability

**S**earch **ARG**ument **ABLE** — a predicate an index can *seek* on.

The rule: **the indexed column must appear bare on one side of the comparison.** Wrap it in a function, apply arithmetic to it, or force a type conversion on it, and the optimizer can no longer use the index's sort order.

| ❌ Not sargable | ✅ Sargable |
|---|---|
| `WHERE YEAR(LoadDate) = 2025` | `WHERE LoadDate >= '2025-01-01' AND LoadDate < '2026-01-01'` |
| `WHERE CAST(Id AS varchar) = '5'` | `WHERE Id = 5` |
| `WHERE Price * 1.1 > 100` | `WHERE Price > 100 / 1.1` |
| `WHERE ISNULL(Status,'') = 'A'` | `WHERE Status = 'A'` |
| `WHERE Name LIKE '%smith'` | `WHERE Name LIKE 'smith%'` |
| `WHERE DATEDIFF(day, Dt, GETDATE()) < 7` | `WHERE Dt > DATEADD(day, -7, GETDATE())` |
| `WHERE LEFT(Code, 3) = 'ABC'` | `WHERE Code LIKE 'ABC%'` |

**Leading-wildcard `LIKE` is genuinely unfixable** with a normal index — nothing about a B-tree helps you find strings ending in something. If you need it, that's Full-Text Search's job.

### The date-range pattern in full

```sql
DECLARE @Start date = DATEFROMPARTS(@Year, 1, 1),
        @End   date = DATEFROMPARTS(@Year + 1, 1, 1);

WHERE LoadDate >= @Start AND LoadDate < @End
```

**Why half-open (`< @End`) and not `BETWEEN`:** with `datetime`/`datetime2`, `BETWEEN '2025-01-01' AND '2025-12-31'` silently excludes everything on December 31st after midnight, because the literal means `2025-12-31 00:00:00`. That's a whole day of data, missing, with no error. Half-open ranges are correct for every temporal type and every precision, and you never have to think about it again.

### When you can't avoid the function

Persist it and index it:

```sql
ALTER TABLE raw.SalesExtract ADD LoadYear AS YEAR(LoadDate) PERSISTED;
CREATE INDEX IX_Sales_LoadYear ON raw.SalesExtract (LoadYear, SourceSystem);
```

Now `WHERE YEAR(LoadDate) = @Year` becomes seekable — the optimizer matches the expression to the computed column.

### Fiscal years

Shift, then extract. For a July 1 start where FY is named for the ending year:

```sql
YEAR(DATEADD(month, 6, LoadDate))
```

Same sargability caveat, so a persisted computed column is usually the right move. Or better: a **calendar/date dimension table** with `FiscalYear`, `FiscalQuarter`, `FiscalPeriod` columns, joined on the date. Then every fiscal rule lives in one table instead of scattered through expressions, and changing the fiscal calendar is a data change rather than a code change.

---

## Implicit conversion

The invisible sargability killer.

```sql
-- Column is varchar(20). Parameter is nvarchar (the default from .NET!)
WHERE AccountCode = @AccountCode
```

`nvarchar` has higher data type precedence than `varchar`, so SQL Server converts **the column** — every row of it — to `nvarchar`. That's a function on the column. Scan.

**This is one of the most common causes of "the index exists but isn't being used,"** and it's especially common with .NET applications, because ADO.NET sends string parameters as `nvarchar` by default.

**How to spot it:** the execution plan shows a `CONVERT_IMPLICIT` in the predicate, and there's a warning on the SELECT operator. Look for it whenever an obviously-indexed column is scanning.

**Fixes:**
- Match your parameter types to your column types (in .NET: set `SqlDbType.VarChar` explicitly)
- Or use `nvarchar` columns consistently and stop mixing
- Never fix it by casting the column — that makes it worse

Same problem class: comparing `int` to `varchar`, `date` to `varchar`, or joining columns with different collations.

---

## Index anatomy

```sql
CREATE NONCLUSTERED INDEX IX_Sales_Date_Source
    ON raw.SalesExtract (LoadDate, SourceSystem)   -- KEY
    INCLUDE (ProductId, RegionId, Qty, Revenue)    -- LEAF ONLY
    WHERE IsDeleted = 0;                           -- FILTER
```

**Key columns** — sorted, seekable, usable for `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`.
**Included columns** — carried in the leaf level only. Not seekable, but available to the `SELECT` list, which is what makes an index *covering*.
**Filter** — index only a subset of rows. Smaller and cheaper.

### Column order

1. Equality predicates first (`WHERE A = @a`)
2. Range predicates last (`WHERE B > @b`)
3. Everything else the query needs → `INCLUDE`

An index on `(A, B)` serves:
- ✅ `WHERE A = 1`
- ✅ `WHERE A = 1 AND B = 2`
- ✅ `ORDER BY A, B`
- ❌ `WHERE B = 2` alone (can scan the index, can't seek)

Think of a phone book sorted by (LastName, FirstName). Finding "Smith" is instant. Finding everyone named "John" means reading the whole book.

### Key lookups

The index found the row but didn't have every column the query wanted, so it jumps to the base table for the rest.

Cheap once. **Catastrophic 500,000 times** — and the optimizer knows this, which is why past a certain estimated row count it abandons the index entirely and scans instead. That's the "why isn't it using my index?" answer more often than people expect.

Fix: add the missing columns to `INCLUDE`.

### Covering index

Contains everything a query needs. The query never touches the base table. This is the target for any hot query.

### Costs — indexes are not free

Every index:
- Slows every `INSERT`, `UPDATE` (of its columns), and `DELETE`
- Consumes disk and buffer pool memory
- Must be maintained, backed up, and checked

**Duplicate and unused indexes are pure cost.** Find them:

```sql
-- unused, or write-only, indexes
SELECT OBJECT_SCHEMA_NAME(i.object_id) + '.' + OBJECT_NAME(i.object_id) AS TableName,
       i.name, s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM   sys.indexes AS i
LEFT JOIN sys.dm_db_index_usage_stats AS s
       ON s.object_id = i.object_id AND s.index_id = i.index_id
      AND s.database_id = DB_ID()
WHERE  i.type_desc = 'NONCLUSTERED'
  AND  OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND  ISNULL(s.user_seeks,0) + ISNULL(s.user_scans,0) + ISNULL(s.user_lookups,0) = 0
ORDER  BY s.user_updates DESC;
```

**Caveat:** these stats reset on service restart, and they don't know about the quarterly report that runs once. Check uptime before dropping anything, and prefer disabling over dropping so it's easy to reverse.

### Missing index suggestions

The optimizer's hints, visible in plans and in `sys.dm_db_missing_index_details`.

**Treat as a starting point for thinking, never as a script to run.** They:
- Ignore indexes that already exist (suggesting near-duplicates)
- Suggest absurdly wide `INCLUDE` lists
- Don't consider write cost
- Are per-query, with no view of the whole workload

A suggestion of "97% improvement" on a query that runs twice a year is not worth an index that slows down every insert.

---

## Reading execution plans

The highest-leverage SQL Server skill there is. Learn to read them and most performance mysteries stop being mysteries.

**Actual plan** (Ctrl+M in SSMS) includes real row counts. **Estimated plan** (Ctrl+L) doesn't run the query. Use actual whenever you can.

### What to look for, in order

**1. Estimated vs actual rows.** Hover the operators. A large gap (say 100× or more) means the optimizer was working from bad information, and every choice downstream of that gap is suspect. **This is the single most useful diagnostic in the plan.**

**2. Scans where you expected seeks.** On a large table, ask why. Usually: non-sargable predicate, implicit conversion, missing index, or a key lookup so expensive the optimizer gave up on the index.

**3. Warnings (yellow triangles).** Implicit conversions, spills to tempdb, missing join predicates, excessive memory grants. These are the engine telling you exactly what's wrong.

**4. Thick arrows.** Arrow width is proportional to row count. A thick arrow early in the plan that becomes thin later means you're moving a lot of rows just to throw them away — push the filter earlier.

**5. The expensive operator.** Percentages are *estimates* and can lie badly when estimates are wrong. Use them as a hint, not gospel.

### Operators worth recognizing

| Operator | Means | Concern when |
|---|---|---|
| Clustered Index Seek | Direct navigation | Rarely |
| Clustered Index Scan | Full table read | Large table, selective predicate expected |
| Index Seek + Key Lookup | Found rows, fetching more columns | High row count → add `INCLUDE` |
| Nested Loops | Loop outer, seek inner | Outer row count is large |
| Hash Match | Build hash table, probe | Memory-hungry; spills are bad |
| Merge Join | Both inputs sorted, zip together | Usually fine |
| Sort | Ordering data | Expensive; often removable with the right index |
| Table Spool / Eager Spool | Materializing intermediate results | Often a sign of a rewriteable query |
| Parallelism (Gather Streams) | Multi-threaded | Fine, unless skewed |

### Where to find plans after the fact

```sql
-- top queries by total CPU, with their plans
SELECT TOP 20
       qs.total_worker_time / 1000 AS TotalCPU_ms,
       qs.execution_count,
       qs.total_worker_time / qs.execution_count / 1000 AS AvgCPU_ms,
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
           ((CASE qs.statement_end_offset WHEN -1
             THEN DATALENGTH(st.text) ELSE qs.statement_end_offset END
             - qs.statement_start_offset)/2)+1) AS StatementText,
       qp.query_plan
FROM   sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)      AS st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle)   AS qp
ORDER  BY qs.total_worker_time DESC;
```

**Query Store** (2016+) is better — turn it on. It persists plans and runtime stats across restarts, shows plan changes over time, and lets you force a good plan when a regression happens.

```sql
ALTER DATABASE MyDb SET QUERY_STORE = ON
    (OPERATION_MODE = READ_WRITE, QUERY_CAPTURE_MODE = AUTO);
```

> **Note on "total CPU" vs "average CPU":** a query taking 50ms but running 2 million times a day costs far more than one taking 30 seconds nightly. Sort by total, not average, when hunting for what to fix.

---

## Statistics & cardinality estimates

**Statistics** are histograms of value distribution. The optimizer uses them to guess how many rows a predicate will return — the **cardinality estimate**. Every plan decision (seek vs scan, loop vs hash, memory grant) flows from that guess.

Bad guess → bad plan → mystery slowness.

### Why estimates go wrong

- **Stale statistics.** Auto-update triggers after roughly 20% of rows change (better in 2016+ with the dynamic threshold). On a 100-million-row table that's 20 million changes — a long time to wait.
- **Ascending key problem.** New rows have dates beyond the histogram's last step, so the optimizer estimates 1 row for "today's data" when it's actually 50,000.
- **Skew.** One value appears 4 million times, others appear 40. Averages mislead.
- **Multi-column correlation.** SQL Server assumes independence between predicates. `WHERE City = 'Seattle' AND State = 'WA'` gets multiplied down to a much smaller estimate than reality.
- **Table variables.** No statistics at all.
- **Multi-statement TVFs.** Fixed guess of 100 rows.

### Maintenance

```sql
UPDATE STATISTICS raw.SalesExtract WITH FULLSCAN;

-- see when stats were last updated and how stale they are
SELECT OBJECT_NAME(s.object_id) AS TableName, s.name AS StatName,
       sp.last_updated, sp.rows, sp.rows_sampled, sp.modification_counter
FROM   sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE  OBJECTPROPERTY(s.object_id,'IsUserTable') = 1
ORDER  BY sp.modification_counter DESC;
```

**Updating statistics is usually more valuable than rebuilding indexes**, and much cheaper. If you're going to have one maintenance job, make it a statistics job before an index-rebuild job.

---

## Parameter sniffing

The optimizer compiles a plan for the **first** parameter value it sees, caches it, and reuses it for everyone.

Fine when data is uniform. Disastrous when it isn't:

```sql
EXEC GetOrders @CustomerId = 42;    -- 40 rows  → nested loops + seek. Good.
EXEC GetOrders @CustomerId = 7;     -- 4M rows  → same plan. 4 million seeks. Terrible.
```

Whoever runs first determines everyone's plan for the rest of the day. The classic symptom: "it's fast normally but slow after the server restarts" (or vice versa).

### Fixes

| Fix | Cost | Use when |
|---|---|---|
| `OPTION (RECOMPILE)` | Compile every run (~ms) | Skewed data, infrequent execution. **Best default.** |
| `OPTIMIZE FOR (@p = value)` | None | You know the typical value |
| `OPTIMIZE FOR UNKNOWN` | None | Use average density; mediocre for everyone but never terrible |
| Local variable copy | None | Same effect as UNKNOWN, older idiom |
| Separate procedures | Maintenance | Two genuinely different workloads |
| Query Store plan forcing | None | A specific known-good plan exists |

```sql
-- the blunt, effective option
SELECT ... FROM ... WHERE CustomerId = @CustomerId OPTION (RECOMPILE);
```

**Always use `OPTION (RECOMPILE)` with the `(@p IS NULL OR col = @p)` optional-parameter pattern.** That pattern's plans are never reusable across parameter combinations, so caching them is actively harmful.

**SQL Server 2022+** has Parameter Sensitive Plan optimization, which caches multiple plans for different value ranges automatically. It helps, and it doesn't cover everything.

---

## RBAR and set-based thinking

**RBAR** — "Row By Agonizing Row." Cursors, `WHILE` loops, scalar UDFs per row. Often a 100× difference against the set-based equivalent.

```sql
-- ❌ RBAR: 10,000 round trips through the engine
DECLARE cur CURSOR FOR SELECT Id, Qty FROM Orders;
OPEN cur; FETCH NEXT FROM cur INTO @Id, @Qty;
WHILE @@FETCH_STATUS = 0
BEGIN
    UPDATE Orders SET Total = @Qty * 10 WHERE Id = @Id;
    FETCH NEXT FROM cur INTO @Id, @Qty;
END

-- ✅ Set-based: one statement
UPDATE Orders SET Total = Qty * 10;
```

### The mental shift

Stop thinking "for each row, do X." Start thinking "here is a set; here is the transformation."

The engine can then parallelize, choose join algorithms, and optimize the whole operation. It cannot do any of that with a loop, because a loop dictates the *how* and leaves the optimizer nothing to decide.

**When a cursor is actually legitimate:**
- Iterating over databases or tables to run maintenance
- Calling a procedure once per row where the procedure has side effects
- Genuinely sequential logic that can't be expressed as a window function (rare — running totals, gaps-and-islands, and "compare to previous row" are all window function territory)

If you must use one: `DECLARE cur CURSOR LOCAL FAST_FORWARD FOR ...` is much lighter than the default.

---

## Batching large operations

A single `DELETE` of 50 million rows will: escalate to a table lock, block everyone, grow the transaction log enormously, and take forever to roll back if it fails at 90%.

```sql
SET NOCOUNT ON;
DECLARE @BatchSize int = 10000, @Rows int = 1;

WHILE @Rows > 0
BEGIN
    BEGIN TRAN;
        DELETE TOP (@BatchSize) FROM raw.SalesExtract
        WHERE LoadDate < DATEADD(year, -3, GETDATE());
        SET @Rows = @@ROWCOUNT;
    COMMIT;

    WAITFOR DELAY '00:00:00.100';   -- let other work through
END
```

Each batch is its own short transaction: locks released between batches, log can truncate (in `SIMPLE`) or be backed up (in `FULL`), progress is preserved if you have to stop.

**Batch size:** 5,000–10,000 is a reasonable starting point. Below ~5,000 lock escalation is unlikely to trigger. Measure rather than guess.

**Make sure the `WHERE` clause is indexed**, or each batch scans the whole table to find its 10,000 rows and you've made things worse.

---

## tempdb

Shared scratch for temp tables, table variables, sorts, hashes, and row versions (RCSI/snapshot). System-wide, so tempdb contention affects *everything*.

Configuration that matters:
- **Multiple equally-sized data files** — 4 to 8, or one per core up to 8. Reduces allocation page contention.
- **Same initial size, same autogrowth** on all files, or the allocation algorithm skews toward the largest.
- **Fast storage.** It's the hottest thing on the instance.
- **Pre-size it** so it doesn't spend production time growing.

*(SQL Server 2016+ configures multiple files at install and enables the relevant trace flags by default. On older installs, check.)*

**Spills to tempdb** show as warnings in actual execution plans — a sort or hash needed more memory than granted. The root cause is nearly always a bad cardinality estimate, so fix the estimate rather than the memory grant.

---

## Where to start when something is slow

Work top-down. Most people start at step 4 and waste hours.

**1. Is it the server or the query?**
```sql
SELECT TOP 10 wait_type, wait_time_ms, waiting_tasks_count
FROM   sys.dm_os_wait_stats
WHERE  wait_type NOT IN ('CLR_SEMAPHORE','SLEEP_TASK','BROKER_TASK_STOP',
       'XE_TIMER_EVENT','SQLTRACE_INCREMENTAL_FLUSH_SLEEP','WAITFOR',
       'REQUEST_FOR_DEADLOCK_SEARCH','LAZYWRITER_SLEEP','CHECKPOINT_QUEUE')
ORDER  BY wait_time_ms DESC;
```
Rough guide: `CXPACKET`/`CXCONSUMER` → parallelism (often a symptom, not a cause). `PAGEIOLATCH_*` → disk reads / missing indexes. `LCK_M_*` → blocking. `RESOURCE_SEMAPHORE` → memory pressure. `WRITELOG` → log disk.

**2. Is something blocking right now?**
```sql
SELECT r.session_id, r.blocking_session_id, r.wait_type, r.wait_time,
       r.status, t.text
FROM   sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE  r.blocking_session_id <> 0;
```

**3. Which query?** Query Store, or the `dm_exec_query_stats` query above. Sort by *total* CPU/reads, not average.

**4. Why is that query slow?** Get the actual plan. Check estimated vs actual rows first, then warnings, then scans.

**5. Fix.** In order of likelihood: sargability → implicit conversion → missing/wrong index → stale statistics → parameter sniffing → rewrite.

> **Measure before and after.** `SET STATISTICS IO, TIME ON` gives you logical reads and CPU time. Logical reads are the most honest metric — they're not affected by caching or by what else the server is doing.

---

[← Back to index](README.md) · [Next: Concurrency →](05-concurrency.md)
