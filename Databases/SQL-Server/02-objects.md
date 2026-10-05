# 02 — Database Objects

[← Back to index](README.md) · [← Architecture](01-architecture.md) · [Next: Refresh Patterns →](03-refresh-patterns.md)

The full catalog: what each object is for, and the trap in each.

---

## Contents

- [The decision table](#the-decision-table)
- [Layer 1 — Storage](#layer-1--storage)
- [Layer 2 — Logic you call](#layer-2--logic-you-call)
- [Layer 3 — Logic that fires itself](#layer-3--logic-that-fires-itself)
- [Layer 4 — Change detection](#layer-4--change-detection)
- [Temp objects](#temp-objects)
- [Odds and ends](#odds-and-ends)

---

## The decision table

| I want to… | Use |
|---|---|
| Store data | **Table** |
| Guarantee a rule about data | **Constraint** (not code) |
| Make a query fast | **Index** |
| Derive a column from other columns | **Computed column** (`PERSISTED` if you'll index it) |
| Name a query shape | **View** |
| Name a query shape that takes a parameter | **Inline TVF** |
| Change data, or do multiple steps | **Stored procedure** |
| React automatically to any write | **Trigger** (keep it tiny) |
| Know what changed since last time | **Change Tracking** / `rowversion` |
| Keep history of value changes | **Temporal table** / **CDC** |
| Scratch space inside a procedure | **`#temp` table** |
| Rename or relocate an object transparently | **Synonym** |
| Share an ID across tables | **Sequence** |

---

## Layer 1 — Storage

### Tables

The clustered index **is** the table — data pages are the index's leaf level. A table without one is a **heap**, where rows sit wherever there was space. Heaps are appropriate for pure staging where you truncate-and-bulk-load and never seek; for anything you query, add a clustered index.

**Choosing the clustered index key** — the three properties that matter:

1. **Narrow** — it's duplicated into every nonclustered index on the table
2. **Ever-increasing** — appends go to the end rather than splitting pages in the middle
3. **Unique** — if it isn't, SQL Server silently adds a 4-byte uniquifier

An `int IDENTITY` satisfies all three, which is why it's the boring default. A random `GUID` violates #2 badly and causes constant page splits — use `NEWSEQUENTIALID()` if you must have GUIDs.

**Data types — get these right at creation:**

| Need | Use | Not |
|---|---|---|
| Money | `decimal(19,4)` | `float` (binary rounding errors) |
| Date only | `date` | `datetime` |
| Timestamp | `datetime2(0..3)` | `datetime` (3.33ms precision, larger) |
| Timezone-aware | `datetimeoffset` | storing local time |
| Non-Unicode text | `varchar(n)` | `varchar(max)` unless truly needed |
| Unicode text | `nvarchar(n)` | mixing with `varchar` in comparisons |
| Yes/no | `bit` | `char(1)` |
| Row change marker | `rowversion` | a manually-maintained column |

> **Never `float` for money.** `0.1 + 0.2 <> 0.3` in binary floating point. Use `decimal`.

> **`varchar(max)` / `nvarchar(max)`** can't be indexed as a key and can push data off-row. Only when you genuinely need >8000 chars.

> **Mixing `varchar` and `nvarchar`** in a comparison causes an implicit conversion that destroys sargability. Pick one convention and stick to it.

### Constraints — the most underrated automation in the product

Constraints enforce rules **without any code running**, are checked on **every** path into the table, and **cannot be forgotten**.

```sql
-- NOT NULL: a guarantee AND an optimizer hint
Qty decimal(18,2) NOT NULL

-- DEFAULT: fills in when the caller doesn't
RefreshedUtc datetime2(0) NOT NULL DEFAULT SYSUTCDATETIME()

-- CHECK: a business rule the engine enforces
CONSTRAINT CK_Sales_QtyNonNeg CHECK (Qty >= 0)

-- UNIQUE: your grain, enforced
CONSTRAINT UQ_MasterSales_Grain UNIQUE (DataYear, ProductName, RegionName)

-- FOREIGN KEY: referential integrity
CONSTRAINT FK_Sales_Product FOREIGN KEY (ProductId)
    REFERENCES ref.ProductMap (ProductId)
```

> **The principle: anything the database can guarantee, let it guarantee.** You can forget to call a procedure. You can write a new code path that skips validation. You cannot forget a constraint — it's checked when SSIS loads, when your proc runs, and when someone types an `UPDATE` in SSMS at 11pm.

**Trusted vs untrusted constraints.** A constraint added `WITH NOCHECK` doesn't validate existing rows and is marked untrusted — and the optimizer then *ignores it* for query simplification. Check whether yours are trusted:

```sql
SELECT name, is_not_trusted FROM sys.foreign_keys   WHERE is_not_trusted = 1
UNION ALL
SELECT name, is_not_trusted FROM sys.check_constraints WHERE is_not_trusted = 1;
```

Fix with `ALTER TABLE t WITH CHECK CHECK CONSTRAINT c;` (yes, `CHECK CHECK` — not a typo).

### Computed columns

A column defined by an expression rather than stored input.

```sql
ALTER TABLE raw.SalesExtract
    ADD LoadYear AS YEAR(LoadDate) PERSISTED;

CREATE INDEX IX_Sales_Year ON raw.SalesExtract (LoadYear, SourceSystem);
```

- **Not persisted** — computed on read. Free storage, costs CPU per access.
- **`PERSISTED`** — physically stored, maintained automatically, **indexable**.

The automation you get free: it can never drift out of sync with its inputs, because there's no separate update step to forget.

Requires a **deterministic** expression — no `GETDATE()`, no `NEWID()`.

**Use case:** making a non-sargable pattern sargable. `WHERE YEAR(LoadDate) = @Y` normally forces a scan; with a persisted computed column plus index, the optimizer matches the expression and seeks.

### Indexes

**Nonclustered index anatomy:**

```sql
CREATE NONCLUSTERED INDEX IX_Sales_Date_Source
    ON raw.SalesExtract (LoadDate, SourceSystem)   -- KEY: seekable, sorted
    INCLUDE (Qty, Revenue, ProductId)              -- INCLUDE: available, not seekable
    WHERE IsDeleted = 0;                           -- FILTER: only these rows
```

**Key column order rules:**
1. Equality predicates first
2. Range predicates last
3. Everything else → `INCLUDE`

An index on `(A, B)` serves `WHERE A = 1`, `WHERE A = 1 AND B = 2`, and `ORDER BY A, B`. It generally does **not** serve `WHERE B = 2` alone.

**Filtered indexes** are excellent for "hot subset" patterns — index only active rows, only the current year, only non-null values. Much smaller, much cheaper to maintain. Caveat: the query's predicate must match the filter for the optimizer to use it, and parameterized queries sometimes won't match.

**Columnstore** — for analytic tables with millions of rows and aggregate-heavy queries. Compresses enormously, scans very fast, poor for single-row lookups. If a master table grows into the tens of millions of rows and everything against it is `GROUP BY` and `SUM`, a clustered columnstore index is a big lever.

**The cost of indexes:** every one makes `INSERT`/`UPDATE`/`DELETE` slower and consumes space and memory. The right number is *as few as possible while queries are fast enough*.

---

## Layer 2 — Logic you call

| Object | What it is | Reach for it when |
|---|---|---|
| **Stored procedure** | Imperative batch. DML, DDL, transactions, loops, multiple result sets | Anything that *changes* data or has steps |
| **View** | A saved `SELECT`. No storage, no precomputation | Naming a join, hiding column sprawl |
| **Inline TVF** | A view that takes parameters. Expanded into the caller | A "view" needing a `@Year` |
| **Scalar function** | Returns one value | Sparingly — see below |
| **Multi-statement TVF** | Builds a table variable, returns it | Rarely — bad estimates |

### Stored procedures

The workhorse. Standard scaffolding:

```sql
CREATE OR ALTER PROCEDURE mart.DoTheThing
    @Year int,
    @SourceSystem varchar(20) = NULL
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    BEGIN TRY
        BEGIN TRAN;
            -- work here
        COMMIT;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK;
        THROW;
    END CATCH
END
GO
```

**`SET NOCOUNT ON`** — suppresses "n rows affected" messages, which are network chatter some clients mistake for result sets.

**`SET XACT_ABORT ON`** — makes any runtime error abort the entire transaction. **Without it, some errors leave the transaction open and half-applied.** Always on when you open a transaction.

**`THROW`** (not `RAISERROR`) — re-raises the original error with its number and severity intact. `RAISERROR` is legacy and loses information.

**Why procedures rather than app-side SQL:**
- Plan reuse
- One place to fix a bug rather than a redeploy
- `EXECUTE` permission without table permission (ownership chaining)
- The SQL is in your repo where it can be reviewed

### Views

Covered fully in [Architecture → The view layer](01-architecture.md#the-view-layer). The three rules:

1. Name columns explicitly, never `SELECT *`
2. Don't nest more than one level
3. It's not a performance feature (unless indexed)

### Functions — the scalar UDF landmine

**Pre-SQL Server 2019**, a scalar UDF in a `SELECT` list:
- Executes **once per row**, serially
- **Silently forces the entire plan single-threaded**
- Is invisible in the execution plan — its cost is hidden

A query goes from 2 seconds to 4 minutes with no obvious cause in the plan. This is one of the most common "why is this so slow?" mysteries in SQL Server.

**SQL Server 2019+** inlines many scalar UDFs automatically (Scalar UDF Inlining), which mostly fixes it. Not all qualify — the ones doing data access or using certain constructs still don't inline.

**The fix, on any version:** rewrite as an inline TVF and use `CROSS APPLY`.

```sql
-- ❌ scalar UDF
SELECT Id, dbo.fn_CalcMargin(Revenue, Cost) AS Margin FROM Sales;

-- ✅ inline TVF + APPLY
CREATE OR ALTER FUNCTION dbo.fn_Margin (@rev decimal(18,2), @cost decimal(18,2))
RETURNS TABLE AS RETURN
    SELECT Margin = CASE WHEN @rev = 0 THEN 0 ELSE (@rev - @cost) / @rev END;
GO

SELECT s.Id, m.Margin
FROM Sales AS s
CROSS APPLY dbo.fn_Margin(s.Revenue, s.Cost) AS m;
```

**Multi-statement TVFs** (`RETURNS @t TABLE (...) AS BEGIN ... END`) get a fixed cardinality estimate — 1 row before 2014, 100 after. Feed 500,000 rows through one and the optimizer builds a plan for 100. Avoid unless you truly need procedural logic in a table-returning object.

---

## Layer 3 — Logic that fires itself

### DML triggers

`AFTER` / `INSTEAD OF` on `INSERT` / `UPDATE` / `DELETE`.

**The one fact that explains everything about triggers:** they run **inside the caller's transaction**. Good — guaranteed to fire, atomic with the write. Bad — the caller waits for them and holds locks the whole time.

**Therefore: keep them tiny.** Mark a flag, write an audit row. Never aggregate, never call an external system, never do anything slow.

### The most common trigger bug

**Triggers fire once per statement, not once per row.** `inserted` and `deleted` are *tables* that may contain 100,000 rows.

```sql
-- ❌ WRONG — silently processes one arbitrary row when a batch arrives
CREATE TRIGGER trg_Bad ON dbo.Orders AFTER INSERT AS
BEGIN
    DECLARE @Id int = (SELECT OrderId FROM inserted);   -- which one?!
    UPDATE dbo.Summary SET Cnt = Cnt + 1 WHERE OrderId = @Id;
END

-- ✅ RIGHT — set-based, handles any batch size
CREATE TRIGGER trg_Good ON dbo.Orders AFTER INSERT AS
BEGIN
    SET NOCOUNT ON;
    UPDATE s SET Cnt = s.Cnt + i.n
    FROM dbo.Summary AS s
    JOIN (SELECT OrderId, COUNT(*) AS n FROM inserted GROUP BY OrderId) AS i
      ON i.OrderId = s.OrderId;
END
```

The wrong version *works in testing* — because you tested with single-row inserts — and corrupts data quietly in production when SSIS loads a batch.

### The trigger philosophy

Triggers are for things that must happen **no matter who writes the data**, including someone poking at it in SSMS.

If all your writes go through procedures you control, **do the work in the procedure instead.** It's visible, debuggable, and doesn't surprise the person reading the code six months later. Invisible action-at-a-distance is the real cost of triggers, more than performance.

**Other trigger types:**
- **`INSTEAD OF`** — replaces the action. Mainly for making a view updatable.
- **DDL triggers** — fire on schema changes. Genuinely useful for logging who altered what.
- **Logon triggers** — exist. You almost certainly don't want one; a bug locks everyone out.

---

## Layer 4 — Change detection

The alternative to full rebuilds. Understand the four; pick one when full loads stop finishing in time.

| Feature | Tells you | Overhead | Use when |
|---|---|---|---|
| **`rowversion` column** | Which rows changed (via a watermark) | Near zero | Simple incremental, you control the schema |
| **Change Tracking** | Which rows changed + what operation | Low | Incremental ETL — usually the right answer |
| **CDC** | Full before/after images | High (log reader agent) | You need the history of values |
| **Temporal tables** | Full history, queryable by time | Moderate (doubles writes) | Audit questions: "what did this look like in March?" |

**`rowversion`** — the poor man's watermark:

```sql
ALTER TABLE raw.SalesExtract ADD RowVer rowversion;

-- store last processed value, then next run:
SELECT * FROM raw.SalesExtract WHERE RowVer > @LastRowVer;
```
Auto-incrementing, database-wide, changes on any update. Not a date — the old name `timestamp` is a historical mistake.

**Change Tracking** — enable at database and table level, then:

```sql
SELECT ct.SalesId, ct.SYS_CHANGE_OPERATION
FROM CHANGETABLE(CHANGES raw.SalesExtract, @LastVersion) AS ct;
```
Lightweight and usually sufficient. **Watch the retention period** — if your job doesn't run within it, you get an error telling you to do a full reload, which is correct but inconvenient at 3am.

**Temporal tables:**

```sql
CREATE TABLE ref.ProductMap
(
    ProductId int PRIMARY KEY,
    Category  varchar(50) NOT NULL,
    ValidFrom datetime2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo   datetime2 GENERATED ALWAYS AS ROW END   HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = ref.ProductMapHistory));

SELECT * FROM ref.ProductMap FOR SYSTEM_TIME AS OF '2025-03-01';
```

This is the easiest way to get Type 2 SCD behavior on your `ref` tables without maintaining it yourself. If "what category was this product in last year?" is a question anyone might ask, turn it on now — you cannot retroactively create history.

---

## Temp objects

| | `#temp` table | `@table` variable | CTE |
|---|---|---|---|
| Statistics | ✅ Yes | ❌ No | n/a — not materialized |
| Estimate | Accurate | 1 row (pre-2019) | Depends on underlying |
| Indexes | ✅ Any | Limited | ❌ |
| In transactions | Rolls back | Survives rollback | n/a |
| Parallelism | ✅ | ❌ for inserts | ✅ |

**Default to `#temp`** for anything over a few hundred rows. The lack of statistics on table variables produces catastrophic plans at scale — the optimizer thinks there's 1 row, chooses nested loops, and there are 2 million.

*(SQL Server 2019+ has deferred compilation for table variables, which improves this considerably. `#temp` is still the safer default.)*

**CTEs are neither** — they're naming, not materialization. Referencing a CTE twice executes it twice. If you need materialization, use `#temp`.

**`##global`** temp tables exist and are visible to all sessions. You almost never want one.

---

## Odds and ends

**`OUTPUT` clause** — any DML can emit affected rows:

```sql
DELETE FROM mart.MasterSales
OUTPUT deleted.MasterSalesId, deleted.DataYear INTO dbo.DeleteAudit
WHERE DataYear = @Year;
```
The clean way to log exactly what a statement touched without a second query.

**Sequences vs `IDENTITY`** — a sequence is a standalone number generator. You can grab a value *before* inserting, and share one sequence across multiple tables. Useful for batch IDs spanning several tables.

```sql
CREATE SEQUENCE dbo.BatchIdSeq AS bigint START WITH 1;
DECLARE @BatchId bigint = NEXT VALUE FOR dbo.BatchIdSeq;
```

**Synonyms** — [see Architecture](01-architecture.md#synonyms).

**Extended properties** — attach documentation to objects in the database itself. Good place for the grain statement.

**Partitioned tables** — split one logical table across multiple physical filegroups by a key range. Enables partition switching (instant load/purge) and partition elimination in queries. Real value at large scale; unnecessary complexity below tens of millions of rows.

**Table types & TVPs** — a user-defined table type lets you pass a whole table as a parameter to a procedure. The right way for an application to send a batch of rows in one call instead of N round trips.

```sql
CREATE TYPE dbo.SalesRowList AS TABLE (ProductId int, Qty decimal(18,2));
GO
CREATE PROCEDURE dbo.InsertSales @Rows dbo.SalesRowList READONLY AS ...
```

---

[← Back to index](README.md) · [Next: Refresh Patterns →](03-refresh-patterns.md)
