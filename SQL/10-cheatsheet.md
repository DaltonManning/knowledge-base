# 10 — Cheat Sheet

[← Back to index](README.md) · [← Integration](09-integration.md) · [Next: Reading List →](11-reading-list.md)

**The simplified track.** Everything compressed to lookups and snippets. Each section links to the full treatment.

---

## The five ideas

| Idea | One line |
|---|---|
| **Grain** | What does one row represent? Say it in a sentence. Enforce with `UNIQUE`. |
| **Idempotence** | Running twice = running once. Makes retry safe. |
| **Sargable** | Bare column on one side of the comparison, or no index seek. |
| **Source of truth** | Each fact lives authoritatively in one place. Everything else is derived. |
| **RPO / RTO** | How much data can you lose; how long can you be down. Write them down. |

---

## Definitions in one line each

<details>
<summary><b>Concurrency</b> — <a href="05-concurrency.md">full page</a></summary>

| Term | One line |
|---|---|
| Transaction | All-or-nothing unit of work |
| ACID | Atomic, Consistent, Isolated, Durable |
| Isolation level | How much of others' uncommitted work you can see |
| Dirty read | Reading uncommitted data that may vanish |
| Non-repeatable read | Same row, read twice, different values |
| Phantom read | Same range query, read twice, extra rows |
| Blocking | One waits for another's lock. **Normal.** |
| Deadlock | Two wait for each other. SQL kills one. **A bug.** |
| Lock escalation | ~5000 row locks → one table lock. Batch your DML. |
| Optimistic concurrency | Don't lock; detect conflict at write time via `rowversion` |
| App lock | `sp_getapplock` — named mutex, "one run at a time" |
| RCSI | Row versioning for reads. Readers stop blocking. **Turn it on.** |

</details>

<details>
<summary><b>Reliability</b> — <a href="00-glossary.md#reliability-patterns">full page</a></summary>

| Term | One line |
|---|---|
| Idempotent | Twice = once |
| Deterministic | Same input → same output always |
| Side effect | Changes something outside the return value |
| At-least-once | You'll get it, maybe twice. Answer: be idempotent. |
| Watermark | "Processed everything up to here" |
| Backfill | Run over history to fill a gap or fix a bug |
| Replay | Rebuild from raw. Requires immutable raw + deterministic transform. |
| Fail fast | Stop at the first error rather than continuing wrong |
| Poison record | One bad row that kills every retry. Quarantine it. |

</details>

<details>
<summary><b>Modeling</b> — <a href="00-glossary.md#data-modeling">full page</a></summary>

| Term | One line |
|---|---|
| Grain | What one row represents |
| Cardinality | How many — between tables, or distinct values in a column |
| Selectivity | What fraction a predicate returns. High = index useful. |
| Surrogate key | Meaningless generated ID (`IDENTITY`). **Prefer this.** |
| Natural key | Real-world data as the key. Keep as `UNIQUE`, not PK. |
| Normalization | Each fact in exactly one place |
| Denormalization | Duplicating on purpose for read speed. Your mart layer. |
| Star schema | Fact table + dimension tables |
| Fact table | Measurements, many rows |
| Dimension table | Descriptive attributes, few rows |
| SCD Type 1 | Overwrite, lose history |
| SCD Type 2 | New row with dates, keep history. **Decide early.** |
| Cartesian product | Missing join condition. 10k × 10k = 100M. |
| Fan-out | Join one-to-many then aggregate → inflated totals |

</details>

<details>
<summary><b>Performance</b> — <a href="04-performance.md">full page</a></summary>

| Term | One line |
|---|---|
| Sargable | Index can seek on it |
| Seek | Jump to the rows. Cost ∝ rows returned. |
| Scan | Read everything. Cost ∝ table size. |
| Clustered index | The physical table order. One per table. |
| Heap | No clustered index. Usually a mistake. |
| Covering index | Has every column the query needs |
| `INCLUDE` | Columns in the leaf only — available, not seekable |
| Key lookup | Index found the row, jumping to the table for more columns |
| Statistics | Value distribution histograms |
| Cardinality estimate | The optimizer's row-count guess. Bad guess → bad plan. |
| Parameter sniffing | Plan cached for the first parameter value, reused for all |
| Implicit conversion | Silent type coercion. **Kills sargability invisibly.** |
| RBAR | Row By Agonizing Row. Cursors and loops. |
| Spill to tempdb | Ran out of memory for a sort/hash. Symptom of bad estimates. |

</details>

<details>
<summary><b>Pipelines</b> — <a href="00-glossary.md#pipelines--etl">full page</a></summary>

| Term | One line |
|---|---|
| ETL | Transform then load |
| ELT | Load raw, transform in the database. **Modern default.** |
| Staging / landing | Where raw lands untouched |
| Lineage | Traceable path from a report number back to source rows |
| Full load | Reprocess everything. Simple, idempotent. **Start here.** |
| Incremental | Only what changed. Faster, riskier. |
| Late-arriving data | Rows for a period you already processed |
| Schema drift | Source changed shape without telling you |
| CDC | Full before/after images. Heavy. |
| Change Tracking | Which rows changed. Light. **Usually what you want.** |
| Temporal table | Auto-maintained history, query `AS OF` a time |
| `rowversion` | Auto-incrementing change marker. Poor man's watermark. |
| Partition switching | Metadata-only swap. Instant, any size. |

</details>

<details>
<summary><b>Operations</b> — <a href="08-operations.md">full page</a></summary>

| Term | One line |
|---|---|
| `SIMPLE` recovery | Log auto-truncates, no point-in-time restore |
| `FULL` recovery | Point-in-time restore — **and you MUST back up the log** |
| Full / diff / log backup | Everything / since last full / since last log |
| RPO | How much data you can lose |
| RTO | How long you can be down |
| Transaction log | Write-ahead record. Durability + rollback come from here. |
| tempdb | Shared scratch. Contention affects everything. |
| `DBCC CHECKDB` | Corruption check. **Run weekly.** |
| DTC | Cross-instance transactions. Slow. Avoid. |
| Linked server | Query another server. Often a performance disaster. |

</details>

---

## Decision tables

### Which object?

| Need | Use |
|---|---|
| Change data / multiple steps | Stored procedure |
| Read-only shape | View |
| Read-only shape + parameter | Inline TVF |
| One value per row | Inline TVF + `CROSS APPLY` (**not** scalar UDF) |
| React to any write | Trigger (tiny only) |
| Guarantee a rule | Constraint (**not** code) |
| Scratch space > few hundred rows | `#temp` (**not** `@table`) |

### One table or two?

> Can every row answer the same "one row per what?" question?
> **Yes** → one table, more columns. **No** → separate tables.

### One database or two?

> Default: **one database, multiple schemas.**
> Split only for: different retention, different HA needs, different server, wildly different size, genuinely different application.

### Full or incremental load?

> Start **full**. Go incremental only when full demonstrably stops finishing in time.
> Then in order: better index → narrower slice → RCSI → build-aside → watermark → partition switch → columnstore.

### How to trigger a refresh?

| Situation | Approach |
|---|---|
| You control the loader | **Call the proc when the load finishes** |
| Scheduled batch | Agent job, period parameter, + nightly prior-period pass |
| Writes from anywhere | Trigger marks dirty → job processes queue |
| Ever | ❌ Do the actual work in a trigger |

---

## Snippets

### Sargable date range

```sql
DECLARE @Start date = DATEFROMPARTS(@Year, 1, 1),
        @End   date = DATEFROMPARTS(@Year + 1, 1, 1);
WHERE d >= @Start AND d < @End
```

### Idempotent refresh skeleton

```sql
SET NOCOUNT, XACT_ABORT ON;
BEGIN TRY
    BEGIN TRAN;
        EXEC @rc = sp_getapplock @Resource='Refresh:2025',
             @LockMode='Exclusive', @LockOwner='Transaction', @LockTimeout=30000;
        IF @rc < 0 BEGIN ROLLBACK; THROW 50001,'Already running.',1; END

        DELETE FROM mart.MasterSales WHERE DataYear = @Year;
        INSERT mart.MasterSales (...) SELECT ... OPTION (RECOMPILE);
    COMMIT;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK;
    THROW;
END CATCH
```

### Deduplicate

```sql
WITH R AS (SELECT *, rn = ROW_NUMBER() OVER (
                         PARTITION BY KeyCols ORDER BY LoadedUtc DESC)
           FROM raw.T)
DELETE FROM R WHERE rn > 1;
```

### Find what's missing

```sql
SELECT DISTINCT r.ProductId
FROM   raw.SalesExtract AS r
WHERE  NOT EXISTS (SELECT 1 FROM ref.ProductMap AS p WHERE p.ProductId = r.ProductId);
```

### Top N per group

```sql
SELECT g.RegionName, t.*
FROM   ref.RegionMap AS g
CROSS APPLY (SELECT TOP 3 ProductName, Revenue
             FROM mart.MasterSales AS m
             WHERE m.RegionName = g.RegionName
             ORDER BY Revenue DESC) AS t;
```

### Running total

```sql
SUM(Revenue) OVER (PARTITION BY ProductName ORDER BY DataYear
                   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

### Year over year

```sql
Revenue - LAG(Revenue) OVER (PARTITION BY ProductName ORDER BY DataYear)
```

### Safe division

```sql
Revenue / NULLIF(Qty, 0)
```

### Conditional aggregation (better than PIVOT)

```sql
SUM(CASE WHEN RegionName = 'North' THEN Revenue END) AS NorthRevenue
```

### Batched delete

```sql
WHILE 1=1
BEGIN
    DELETE TOP (10000) FROM raw.T WHERE LoadDate < @Cutoff;
    IF @@ROWCOUNT = 0 BREAK;
    WAITFOR DELAY '00:00:00.100';
END
```

### Safe dynamic SQL

```sql
SET @sql = N'SELECT * FROM mart.' + QUOTENAME(@Table) + N' WHERE DataYear=@Y';
EXEC sys.sp_executesql @sql, N'@Y int', @Y = @Year;
```

### Re-runnable DDL

```sql
IF SCHEMA_ID('mart') IS NULL EXEC('CREATE SCHEMA mart;');
IF OBJECT_ID('mart.T','U') IS NULL CREATE TABLE mart.T (...);
CREATE INDEX IX_T_X ON mart.T (X) WITH (DROP_EXISTING = ON);
CREATE OR ALTER PROCEDURE mart.P AS ...
```

---

## Anti-patterns

| ❌ Don't | ✅ Do |
|---|---|
| `WHERE YEAR(d) = @y` | `WHERE d >= @s AND d < @e` |
| `BETWEEN` on datetime | Half-open range |
| `NOT IN (subquery)` | `NOT EXISTS` |
| `WITH (NOLOCK)` | Enable RCSI |
| Scalar UDF in `SELECT` | Inline TVF + `CROSS APPLY` |
| `SELECT *` in a view | Explicit column list |
| Views nested 3 deep | One flat view over tables |
| `@table` variable, large | `#temp` table |
| Cursor / `WHILE` loop | Set-based statement |
| Aggregation inside a trigger | Trigger marks dirty; job aggregates |
| `MERGE` under concurrency | `UPDATE` then `INSERT` with `UPDLOCK, HOLDLOCK` |
| `INNER JOIN` to lookups | `LEFT JOIN` + `'(unmapped)'` |
| `float` for money | `decimal(19,4)` |
| Auto-created ETL tables | Tables you defined, in git |
| Personal account as a service | gMSA |
| `AUTO_SHRINK ON` | Never |
| `FULL` recovery, no log backups | Log backups, or `SIMPLE` |
| Hand-editing master data | Overrides table joined in |
| Unnamed constraints | `CONSTRAINT DF_Table_Col DEFAULT ...` |
| `WHERE` filter on outer table of `LEFT JOIN` | Put it in the `ON` clause |

---

## Diagnostic quick reference

| Symptom | Check |
|---|---|
| Query suddenly slow | Query Store plan regression; stale statistics |
| Fast for me, slow for them | Parameter sniffing → `OPTION (RECOMPILE)` |
| Index exists but scanning | Non-sargable predicate; implicit conversion; key lookup too costly |
| Everything slow | `sys.dm_os_wait_stats`, then the blocking chain |
| Something stuck | `blocking_session_id` — find the **head** of the chain |
| Deadlocks | `system_health` XE session already has the graphs |
| Log file huge | `log_reuse_wait_desc` in `sys.databases` |
| Disk filling | tempdb version store; log; unbatched DML |
| Wrong totals | Grain violation / fan-out / inner join dropping rows |
| Job "succeeded", no data | Check step-level results; alert on backup/refresh **age** |

```sql
-- what's blocking right now
SELECT session_id, blocking_session_id, wait_type, wait_time
FROM sys.dm_exec_requests WHERE blocking_session_id <> 0;

-- why can't the log truncate
SELECT name, recovery_model_desc, log_reuse_wait_desc FROM sys.databases;

-- when was everything last backed up
SELECT database_name, type, MAX(backup_finish_date)
FROM msdb.dbo.backupset GROUP BY database_name, type;
```

---

## Setup checklist for a new instance

```sql
sp_configure 'max server memory (MB)', <RAM - 4GB>
sp_configure 'cost threshold for parallelism', 50
sp_configure 'max degree of parallelism', <cores per NUMA node, max 8>
sp_configure 'optimize for ad hoc workloads', 1
sp_configure 'backup compression default', 1

ALTER DATABASE X SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
ALTER DATABASE X SET QUERY_STORE = ON (OPERATION_MODE = READ_WRITE);
ALTER DATABASE X SET AUTO_SHRINK OFF;
ALTER DATABASE X SET PAGE_VERIFY CHECKSUM;
```

Plus: tempdb multi-file · Instant File Initialization · Ola Hallengren's scripts · Database Mail + operator · alerts on severity 19–25 and errors 823/824/825 · **write down RPO and RTO**.

---

[← Back to index](README.md) · [Next: Reading List →](11-reading-list.md)
