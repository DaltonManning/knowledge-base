# 03 — Refresh & Automation Patterns

[← Back to index](README.md) · [← Objects](02-objects.md) · [Next: Performance →](04-performance.md)

The heart of it. How derived data gets built, rebuilt, and kept honest.

---

## Contents

- [The core pattern: idempotent slice refresh](#the-core-pattern-idempotent-slice-refresh)
- [Serializing with app locks](#serializing-with-app-locks)
- [Why not MERGE](#why-not-merge)
- [Triggering strategies](#triggering-strategies)
- [The dirty queue pattern](#the-dirty-queue-pattern)
- [Watermark-based incremental](#watermark-based-incremental)
- [Handling unmapped and bad data](#handling-unmapped-and-bad-data)
- [Logging & observability](#logging--observability)
- [Reconciliation](#reconciliation)
- [Scaling up when full rebuild gets slow](#scaling-up-when-full-rebuild-gets-slow)

---

## The core pattern: idempotent slice refresh

**Delete the slice, reinsert the slice, in one transaction.**

Idempotent by construction. Run it once, twice, twenty times — same result. That property matters more than raw speed, because it converts "the job died halfway, what state are we in?" into "run it again."

```sql
CREATE OR ALTER PROCEDURE mart.RefreshMasterSales
    @Year          int,
    @SourceSystem  varchar(20) = NULL      -- NULL = all sources
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    -- Half-open date range: sargable, and no end-of-day boundary bug
    DECLARE @Start date = DATEFROMPARTS(@Year, 1, 1),
            @End   date = DATEFROMPARTS(@Year + 1, 1, 1);

    DECLARE @RowsIn int, @RowsOut int, @StartedUtc datetime2(0) = SYSUTCDATETIME();

    BEGIN TRY
        BEGIN TRAN;

            ----------------------------------------------------------
            -- 1. Serialize: one refresh per slice at a time
            ----------------------------------------------------------
            DECLARE @lock nvarchar(255) =
                CONCAT('RefreshMasterSales:', @Year, ':', ISNULL(@SourceSystem, '*'));
            DECLARE @rc int;

            EXEC @rc = sp_getapplock
                 @Resource    = @lock,
                 @LockMode    = 'Exclusive',
                 @LockOwner   = 'Transaction',
                 @LockTimeout = 30000;          -- 30s

            IF @rc < 0
            BEGIN
                ROLLBACK;
                THROW 50001, 'Refresh already running for this slice.', 1;
            END

            ----------------------------------------------------------
            -- 2. Remove the slice
            ----------------------------------------------------------
            DELETE m
            FROM mart.MasterSales AS m
            WHERE m.DataYear = @Year
              AND (@SourceSystem IS NULL OR m.SourceSystem = @SourceSystem);

            ----------------------------------------------------------
            -- 3. Rebuild the slice
            ----------------------------------------------------------
            INSERT mart.MasterSales
                  (DataYear, SourceSystem, ProductName, RegionName,
                   Qty, Revenue, OrderCount, RefreshedUtc)
            SELECT
                   @Year,
                   r.SourceSystem,
                   ISNULL(p.ProductName, '(unmapped)'),
                   ISNULL(g.RegionName,  '(unmapped)'),
                   SUM(r.Qty),
                   SUM(r.Revenue),
                   COUNT_BIG(*),
                   SYSUTCDATETIME()
            FROM      raw.SalesExtract AS r
            LEFT JOIN ref.ProductMap   AS p ON p.ProductId = r.ProductId
            LEFT JOIN ref.RegionMap    AS g ON g.RegionId  = r.RegionId
            WHERE r.LoadDate >= @Start
              AND r.LoadDate <  @End
              AND (@SourceSystem IS NULL OR r.SourceSystem = @SourceSystem)
            GROUP BY r.SourceSystem,
                     ISNULL(p.ProductName, '(unmapped)'),
                     ISNULL(g.RegionName,  '(unmapped)')
            OPTION (RECOMPILE);

            SET @RowsOut = @@ROWCOUNT;

            ----------------------------------------------------------
            -- 4. Log it
            ----------------------------------------------------------
            INSERT dbo.RefreshLog
                  (ProcName, DataYear, SourceSystem, RowsWritten,
                   StartedUtc, FinishedUtc, Status)
            VALUES ('mart.RefreshMasterSales', @Year, @SourceSystem, @RowsOut,
                    @StartedUtc, SYSUTCDATETIME(), 'Success');

        COMMIT;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK;

        INSERT dbo.RefreshLog
              (ProcName, DataYear, SourceSystem, RowsWritten,
               StartedUtc, FinishedUtc, Status, ErrorMessage)
        VALUES ('mart.RefreshMasterSales', @Year, @SourceSystem, NULL,
                @StartedUtc, SYSUTCDATETIME(), 'Failed', ERROR_MESSAGE());

        THROW;
    END CATCH
END
GO
```

### Why each piece is there

**Half-open date range (`>= @Start AND < @End`)**
Sargable — the optimizer can seek on an index over `LoadDate`. Compare to `WHERE YEAR(LoadDate) = @Year`, which wraps the column in a function and forces a full scan.

The `<` upper bound rather than `<= '2025-12-31'` avoids the end-of-day boundary bug: with `datetime`/`datetime2`, rows timestamped `2025-12-31 14:30` are silently dropped by `<= '2025-12-31'` because that literal means midnight.

**`OPTION (RECOMPILE)`**
The `(@Param IS NULL OR col = @Param)` idiom is convenient but poisons plan reuse. SQL Server caches a plan built for whichever value it saw first — and the plan for "all sources" (scan and aggregate) is badly wrong for "one source" (seek), and vice versa. Recompile costs a few milliseconds and saves you from that.

**`LEFT JOIN` on the `ref` tables**
An inner join silently drops rows whose `ProductId` isn't in the map yet. That's how a total quietly comes up short and nobody notices for a month. `LEFT JOIN` plus an `'(unmapped)'` bucket makes the problem *visible in the output* — someone sees a row labeled unmapped and asks about it.

**`COUNT_BIG(*)` not `COUNT(*)`**
`COUNT(*)` returns `int` and overflows at 2.1 billion. `COUNT_BIG` returns `bigint`. Also required for indexed views. Costs nothing.

**Logging inside the transaction on success, outside on failure**
The success log commits with the work. The failure log is written after `ROLLBACK`, so it survives — if it were inside the rolled-back transaction it would vanish along with everything else, which is exactly when you most need it.

**`THROW` at the end of `CATCH`**
Re-raises so the caller (SQL Agent, SSIS) actually sees a failure. Without it, the job reports success while having done nothing — the worst possible outcome, because nobody investigates.

---

## Serializing with app locks

Without serialization, two concurrent refreshes of the same slice can interleave:

```
Session A: DELETE year 2025      ─┐
Session B: DELETE year 2025       │  both deletes complete
Session A: INSERT year 2025       │  both inserts complete
Session B: INSERT year 2025      ─┘  → DUPLICATE ROWS
```

`sp_getapplock` prevents this. Key points:

- **`@LockOwner = 'Transaction'`** — released automatically at commit/rollback. Safer than `'Session'`, which you must release explicitly and which leaks if the connection is reused.
- **Return codes:** `0` = acquired, `1` = acquired after waiting, `-1` = timeout, `-2` = cancelled, `-3` = deadlock victim. Anything `< 0` failed.
- **Take it *inside* the transaction but *before* the work.**
- **Name the lock by slice**, not by procedure — refreshing 2024 and 2025 concurrently is fine and should be allowed.

The `UNIQUE` constraint on the grain is the backstop for when this fails. Belt and braces: app lock prevents the race, constraint catches it if the prevention has a hole.

---

## Why not MERGE

`MERGE` reads beautifully and does upsert in one statement. It also has:

- A long tail of documented correctness bugs (search "MERGE bug site:feedback.azure.com" for the museum)
- Concurrency issues requiring `WITH (HOLDLOCK)` on the target to be safe — which many examples omit
- Confusing behavior when the source contains duplicate keys (error 8672, at runtime, in production)
- Trigger firing semantics that surprise people

**For a full-slice rebuild it buys you nothing over delete-and-insert.** Use it for small, controlled, low-concurrency cases — seed data, a dirty-flag queue — and reach for explicit `UPDATE` then `INSERT` when volume or concurrency is real.

Safe explicit upsert:

```sql
BEGIN TRAN;

    UPDATE t
    SET    t.Value = s.Value, t.UpdatedUtc = SYSUTCDATETIME()
    FROM   mart.Target AS t WITH (UPDLOCK, HOLDLOCK)
    JOIN   #Source      AS s ON s.[Key] = t.[Key]
    WHERE  t.Value <> s.Value OR (t.Value IS NULL) <> (s.Value IS NULL);

    INSERT mart.Target ([Key], Value, UpdatedUtc)
    SELECT s.[Key], s.Value, SYSUTCDATETIME()
    FROM   #Source AS s
    WHERE  NOT EXISTS (SELECT 1 FROM mart.Target AS t WHERE t.[Key] = s.[Key]);

COMMIT;
```

`UPDLOCK, HOLDLOCK` on the update makes the check-then-insert sequence safe. The `<>` guard in the `WHERE` avoids writing rows that didn't change — less log, fewer triggers fired, less blocking.

---

## Triggering strategies

Four options, roughly in order of preference.

### 1. Call it from the loader (best default)

If SSIS or an application controls the load, call the procedure when the load finishes.

**Pros:** fewest moving parts, runs exactly when data is ready, no polling, no lag, easy to reason about.
**Cons:** only works if you control every write path.

This is the right answer more often than people expect. Reach for something cleverer only when writes arrive from places you don't control.

### 2. Scheduled, on a period parameter

SQL Agent runs the proc for the current period every N minutes, plus a wider backfill nightly.

```sql
-- every 15 minutes: current year
EXEC mart.RefreshMasterSales @Year = YEAR(GETDATE());

-- nightly: current + prior year, to catch late-arriving data
EXEC mart.RefreshMasterSales @Year = YEAR(GETDATE());
EXEC mart.RefreshMasterSales @Year = YEAR(GETDATE()) - 1;
```

**Pros:** simple, self-healing, catches everything eventually.
**Cons:** latency; wasted work when nothing changed.

That nightly prior-period pass is the answer to **late-arriving data**, and it's the step people skip. Rows for December that arrive on January 3rd will never be picked up by a job that only processes "the current year."

### 3. Trigger marks dirty, job processes (best for uncontrolled writes)

→ [see the dirty queue pattern below](#the-dirty-queue-pattern)

### 4. Do the work in a trigger (almost never)

**Don't.** A trigger runs inside the caller's transaction, so every bulk load waits for your aggregation to finish while holding locks. A 30-second refresh turns every insert into a 30-second insert.

---

## The dirty queue pattern

Decouples "something changed" from "recompute it." Ten loads into 2026 collapse into one refresh.

### The queue table

```sql
CREATE TABLE dbo.RefreshQueue
(
    DataYear     int          NOT NULL,
    SourceSystem varchar(20)  NOT NULL,
    IsDirty      bit          NOT NULL DEFAULT 1,
    MarkedUtc    datetime2(0) NOT NULL DEFAULT SYSUTCDATETIME(),
    LastRunUtc   datetime2(0)     NULL,
    CONSTRAINT PK_RefreshQueue PRIMARY KEY (DataYear, SourceSystem)
);
```

### The trigger — tiny, set-based

```sql
CREATE OR ALTER TRIGGER raw.trg_SalesExtract_MarkDirty
ON raw.SalesExtract
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    MERGE dbo.RefreshQueue AS q
    USING (
        SELECT DISTINCT YEAR(LoadDate) AS DataYear, SourceSystem
        FROM (
            SELECT LoadDate, SourceSystem FROM inserted
            UNION ALL
            SELECT LoadDate, SourceSystem FROM deleted
        ) AS x
        WHERE LoadDate IS NOT NULL
    ) AS s
      ON q.DataYear = s.DataYear AND q.SourceSystem = s.SourceSystem
    WHEN MATCHED THEN
        UPDATE SET q.IsDirty = 1, q.MarkedUtc = SYSUTCDATETIME()
    WHEN NOT MATCHED THEN
        INSERT (DataYear, SourceSystem) VALUES (s.DataYear, s.SourceSystem);
END
GO
```

Note the `UNION ALL` of `inserted` and `deleted` — an `UPDATE` that moves a row from 2024 to 2025 dirties *both* years. Missing this is a subtle correctness bug.

(`MERGE` is acceptable here: tiny row counts, and the app lock in the refresh proc handles the real concurrency.)

### The worker

```sql
CREATE OR ALTER PROCEDURE dbo.ProcessRefreshQueue
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @Year int, @Src varchar(20);

    -- Claim one slice atomically
    WHILE 1 = 1
    BEGIN
        UPDATE TOP (1) q
        SET    q.IsDirty = 0, q.LastRunUtc = SYSUTCDATETIME()
        OUTPUT deleted.DataYear, deleted.SourceSystem
          INTO #claimed (DataYear, SourceSystem)
        FROM   dbo.RefreshQueue AS q WITH (READPAST, UPDLOCK)
        WHERE  q.IsDirty = 1;

        IF @@ROWCOUNT = 0 BREAK;

        SELECT TOP 1 @Year = DataYear, @Src = SourceSystem FROM #claimed;
        DELETE FROM #claimed;

        EXEC mart.RefreshMasterSales @Year = @Year, @SourceSystem = @Src;
    END
END
```

**The claim pattern** — `UPDATE ... OUTPUT` with `READPAST, UPDLOCK` — atomically marks a row as taken and tells you which one, skipping rows another worker already grabbed. This is the standard queue-in-a-table idiom and it's worth knowing; hand-rolled `SELECT` then `UPDATE` has a race between the two statements.

**Clear the flag before doing the work**, not after. If a change arrives *during* the refresh, the flag gets set again and it runs once more. Slightly wasteful, always correct. The reverse order can lose a change.

---

## Watermark-based incremental

When full-slice rebuild stops finishing in time.

```sql
CREATE TABLE dbo.LoadWatermark
(
    SourceName   varchar(100) NOT NULL PRIMARY KEY,
    LastRowVer   binary(8)        NULL,
    LastRunUtc   datetime2(0)     NULL
);
```

```sql
CREATE OR ALTER PROCEDURE mart.IncrementalLoadSales
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    DECLARE @Last binary(8), @Current binary(8) = MIN_ACTIVE_ROWVERSION();

    SELECT @Last = ISNULL(LastRowVer, 0x0)
    FROM   dbo.LoadWatermark WHERE SourceName = 'raw.SalesExtract';

    BEGIN TRAN;

        -- process rows in (@Last, @Current)
        INSERT stg.SalesChanges (...)
        SELECT ...
        FROM   raw.SalesExtract
        WHERE  RowVer > @Last AND RowVer < @Current;

        -- ... apply to master ...

        UPDATE dbo.LoadWatermark
        SET    LastRowVer = @Current, LastRunUtc = SYSUTCDATETIME()
        WHERE  SourceName = 'raw.SalesExtract';

    COMMIT;
END
```

### The two traps

**Use `MIN_ACTIVE_ROWVERSION()`, not `@@DBTS`.** `@@DBTS` is the latest version issued — but a transaction in flight may have already been *assigned* a lower version and not yet committed. If you set your watermark past it, you skip that row forever. `MIN_ACTIVE_ROWVERSION()` returns the lowest version of any still-open transaction, so anything below it is safely committed.

**Update the watermark in the same transaction as the work.** Otherwise a failure between the two either reprocesses (harmless if idempotent) or skips (data loss, silent).

> **Only build this when you need it.** Full rebuilds are simpler, naturally idempotent, and self-healing — a bug in a full rebuild gets fixed by re-running. A bug in incremental logic can leave gaps you don't discover for months.

---

## Handling unmapped and bad data

### Make gaps visible, not silent

Three strategies, in order of increasing rigor:

**1. Unknown member** — `LEFT JOIN` + `ISNULL(x, '(unmapped)')`. The row still counts toward totals but is visibly attributed to nothing. Best default.

**2. Rejects table** — route unmatched rows aside for inspection:

```sql
INSERT dbo.LoadRejects (SourceTable, SourceRowId, Reason, CapturedUtc)
SELECT 'raw.SalesExtract', r.SalesId, 'ProductId not in ref.ProductMap', SYSUTCDATETIME()
FROM      raw.SalesExtract AS r
LEFT JOIN ref.ProductMap   AS p ON p.ProductId = r.ProductId
WHERE     r.LoadDate >= @Start AND r.LoadDate < @End
  AND     p.ProductId IS NULL;
```

**3. Data quality gate** — refuse to publish if quality is below threshold:

```sql
IF @UnmappedPct > 5.0
    THROW 50010, 'Refresh aborted: more than 5% of rows have unmapped products.', 1;
```

Loud failure beats quiet wrongness. A report that doesn't refresh gets investigated; a report that refreshes with 20% of revenue missing does not.

### The anti-join is your data quality workhorse

```sql
-- what's in raw that has no map entry?
SELECT DISTINCT r.ProductId
FROM   raw.SalesExtract AS r
WHERE  NOT EXISTS (SELECT 1 FROM ref.ProductMap AS p WHERE p.ProductId = r.ProductId);
```

Run this as a standing check. New unmapped values appearing is your early warning that the source system added something.

---

## Logging & observability

```sql
CREATE TABLE dbo.RefreshLog
(
    RefreshLogId  bigint IDENTITY(1,1) PRIMARY KEY,
    ProcName      sysname       NOT NULL,
    DataYear      int               NULL,
    SourceSystem  varchar(20)       NULL,
    RowsRead      int               NULL,
    RowsWritten   int               NULL,
    StartedUtc    datetime2(0)  NOT NULL,
    FinishedUtc   datetime2(0)      NULL,
    Status        varchar(20)   NOT NULL,   -- Success / Failed / Skipped
    ErrorMessage  nvarchar(2000)    NULL,
    HostName      sysname       NOT NULL DEFAULT HOST_NAME(),
    LoginName     sysname       NOT NULL DEFAULT SUSER_SNAME()
);
CREATE INDEX IX_RefreshLog_Started ON dbo.RefreshLog (StartedUtc DESC);
```

### The questions this answers

Every one of these will be asked of you eventually:

```sql
-- Is anything failing?
SELECT TOP 50 * FROM dbo.RefreshLog
WHERE Status = 'Failed' ORDER BY StartedUtc DESC;

-- Is it getting slower over time?
SELECT CAST(StartedUtc AS date) AS RunDate,
       AVG(DATEDIFF(second, StartedUtc, FinishedUtc)) AS AvgSeconds,
       MAX(DATEDIFF(second, StartedUtc, FinishedUtc)) AS MaxSeconds
FROM   dbo.RefreshLog
WHERE  ProcName = 'mart.RefreshMasterSales' AND Status = 'Success'
GROUP  BY CAST(StartedUtc AS date)
ORDER  BY RunDate DESC;

-- Did a refresh silently produce far fewer rows than usual?
SELECT DataYear, RowsWritten, StartedUtc,
       PrevRows = LAG(RowsWritten) OVER (PARTITION BY DataYear ORDER BY StartedUtc)
FROM   dbo.RefreshLog
WHERE  ProcName = 'mart.RefreshMasterSales' AND Status = 'Success';

-- When did this data last update? (the #1 question about any report)
SELECT DataYear, MAX(FinishedUtc) AS LastRefreshed
FROM   dbo.RefreshLog WHERE Status = 'Success' GROUP BY DataYear;
```

That last one is worth exposing as a view the front end can display. "Data as of 2026-07-31 03:15 UTC" on a dashboard prevents an enormous amount of confusion.

---

## Reconciliation

The step everyone skips.

```sql
CREATE OR ALTER VIEW mart.vw_ReconSales
AS
SELECT
    r.DataYear,
    RawRowCount    = r.RawRows,
    RawRevenue     = r.RawRevenue,
    MasterRevenue  = m.MasterRevenue,
    Variance       = m.MasterRevenue - r.RawRevenue
FROM (
    SELECT YEAR(LoadDate) AS DataYear,
           COUNT_BIG(*)   AS RawRows,
           SUM(Revenue)   AS RawRevenue
    FROM   raw.SalesExtract GROUP BY YEAR(LoadDate)
) AS r
LEFT JOIN (
    SELECT DataYear, SUM(Revenue) AS MasterRevenue
    FROM   mart.MasterSales GROUP BY DataYear
) AS m ON m.DataYear = r.DataYear;
```

Any non-zero variance means rows were dropped or double-counted. Check it after every deployment, and consider alerting on it.

---

## Scaling up when full rebuild gets slow

In the order you should try them:

**1. Index it properly.** Most "too slow" is a missing index on `(LoadDate, SourceSystem)` covering the aggregated columns. Check this first; it's often the whole problem.

**2. Narrow the slice.** Refresh by year+month+source rather than year. Same pattern, smaller unit of work.

**3. `READ_COMMITTED_SNAPSHOT`.** Doesn't make the refresh faster, but stops it blocking readers — which is often the actual complaint.

**4. Build aside, then swap.** Populate a staging table outside the transaction, then do a fast delete+insert (or partition switch) inside it. Minimizes lock duration.

**5. Incremental with a watermark.** → [above](#watermark-based-incremental)

**6. Partition switching.** With the master table partitioned by year, build the new partition in a separate table and switch it in — a metadata-only operation, effectively instant regardless of size.

```sql
-- staging table must match schema, filegroup, and have a CHECK constraint
-- restricting it to exactly the target partition's range
ALTER TABLE mart.MasterSales_Staging
    SWITCH TO mart.MasterSales PARTITION 12;
```

**7. Columnstore.** If the master table is tens of millions of rows and every query is `GROUP BY` + `SUM`, a clustered columnstore index is a step-change in scan speed and compression.

> Work down this list in order. Steps 1–3 solve the vast majority of cases, and steps 5–7 add real complexity that you should only take on with evidence that you need it.

---

[← Back to index](README.md) · [Next: Performance →](04-performance.md)
