# 06 — T-SQL Cookbook

[← Back to index](README.md) · [← Concurrency](05-concurrency.md) · [Next: Project Layout →](07-project-layout.md)

The language patterns worth internalizing.

---

## Contents

- [Three-valued logic](#three-valued-logic)
- [Logical query processing order](#logical-query-processing-order)
- [Join taxonomy](#join-taxonomy)
- [Window functions](#window-functions)
- [APPLY](#apply)
- [Dates and time](#dates-and-time)
- [Strings](#strings)
- [Aggregation patterns](#aggregation-patterns)
- [Gaps and islands](#gaps-and-islands)
- [Pivot and unpivot](#pivot-and-unpivot)
- [Error handling](#error-handling)
- [Dynamic SQL](#dynamic-sql)
- [Useful system queries](#useful-system-queries)

---

## Three-valued logic

SQL has `TRUE`, `FALSE`, and `UNKNOWN`. Any comparison involving NULL yields UNKNOWN, and `WHERE` keeps only rows evaluating to TRUE.

```sql
NULL = NULL      -- UNKNOWN
NULL <> 'A'      -- UNKNOWN
NULL + 1         -- NULL
```

### The three that bite everyone

**1. Inequality silently excludes NULLs**

```sql
-- ❌ excludes rows where Status IS NULL
WHERE Status <> 'Closed'

-- ✅ includes them
WHERE (Status <> 'Closed' OR Status IS NULL)
-- or
WHERE ISNULL(Status, '') <> 'Closed'      -- but this is non-sargable
```

**2. `NOT IN` with a NULL returns nothing at all**

```sql
-- if the subquery returns even one NULL, this returns ZERO rows
WHERE ProductId NOT IN (SELECT ProductId FROM ref.ProductMap)

-- ✅ always use NOT EXISTS
WHERE NOT EXISTS (SELECT 1 FROM ref.ProductMap AS p WHERE p.ProductId = t.ProductId)
```

Why: `x NOT IN (1, 2, NULL)` expands to `x <> 1 AND x <> 2 AND x <> NULL`. That last term is UNKNOWN, and `TRUE AND UNKNOWN` is UNKNOWN. Never TRUE. **Default to `NOT EXISTS` and this class of bug disappears.**

**3. Aggregates ignore NULLs, except `COUNT(*)`**

```sql
COUNT(*)        -- all rows
COUNT(col)      -- rows where col IS NOT NULL
AVG(col)        -- excludes NULLs from BOTH numerator and denominator
SUM(col)        -- NULL if every value is NULL (not 0)
```

`AVG` is the dangerous one: `AVG` over (10, 20, NULL) is 15, not 10. If NULL means zero in your domain, write `AVG(ISNULL(col, 0))`.

### NULL-handling functions

```sql
ISNULL(a, b)              -- SQL Server only; return type from first arg
COALESCE(a, b, c, ...)    -- ANSI; multiple args; higher precedence type
NULLIF(a, b)              -- NULL if a = b; classic divide-by-zero guard
IIF(cond, t, f)           -- shorthand CASE

-- safe division
Revenue / NULLIF(Qty, 0)  -- NULL instead of an error when Qty is 0
```

`COALESCE` is evaluated as a `CASE` expression, so a subquery inside it can be evaluated more than once. `ISNULL` evaluates its arguments once. Minor, but real.

### `NOT EXISTS` vs `EXCEPT`

`EXCEPT` treats NULLs as equal (it uses distinct semantics), unlike `=`. Occasionally exactly what you want for comparing row sets:

```sql
-- rows that differ, NULL-safe
SELECT * FROM #New
EXCEPT
SELECT * FROM #Old;
```

---

## Logical query processing order

The order SQL is *evaluated*, which differs from how it's written:

```
1. FROM
2. ON
3. JOIN
4. WHERE
5. GROUP BY
6. HAVING
7. SELECT
8. DISTINCT
9. ORDER BY
10. TOP / OFFSET-FETCH
```

**This explains a lot:**

- You **can't** reference a `SELECT` alias in `WHERE` — SELECT hasn't happened yet
- You **can** reference it in `ORDER BY` — SELECT has happened
- `WHERE` filters rows; `HAVING` filters groups
- `TOP` with `ORDER BY` is applied *last*, which is why `TOP` without `ORDER BY` is non-deterministic

### `WHERE` on the outer table of a `LEFT JOIN` turns it into an inner join

```sql
-- ❌ this is now effectively an INNER JOIN — the WHERE kills the NULL rows
FROM      raw.Sales AS s
LEFT JOIN ref.ProductMap AS p ON p.ProductId = s.ProductId
WHERE     p.Category = 'Widgets'

-- ✅ put the condition in the ON clause
FROM      raw.Sales AS s
LEFT JOIN ref.ProductMap AS p
       ON p.ProductId = s.ProductId AND p.Category = 'Widgets'
```

One of the most common join bugs, and it produces *fewer* rows than expected — which looks like a data problem rather than a query problem.

---

## Join taxonomy

| Join | Keeps | Use for |
|---|---|---|
| `INNER` | Matches only | When unmatched rows genuinely shouldn't count |
| `LEFT` | All left + matches | Preserving all facts; finding gaps |
| `RIGHT` | All right + matches | Rewrite as LEFT for readability |
| `FULL` | Everything | Reconciliation between two sets |
| `CROSS` | Every combination | Generating grids, calendars, filling series |

### Anti-join — finding what's missing

Your data-quality workhorse.

```sql
-- what's in raw with no matching map entry?
SELECT DISTINCT r.ProductId
FROM   raw.SalesExtract AS r
WHERE  NOT EXISTS (SELECT 1 FROM ref.ProductMap AS p WHERE p.ProductId = r.ProductId);

-- equivalent with LEFT JOIN
SELECT DISTINCT r.ProductId
FROM      raw.SalesExtract AS r
LEFT JOIN ref.ProductMap   AS p ON p.ProductId = r.ProductId
WHERE     p.ProductId IS NULL;
```

Both perform similarly. `NOT EXISTS` reads better and is NULL-safe.

### Semi-join — "does at least one match exist?"

```sql
SELECT * FROM ref.ProductMap AS p
WHERE EXISTS (SELECT 1 FROM raw.SalesExtract AS r WHERE r.ProductId = p.ProductId);
```

Key property: **doesn't multiply rows** the way a join does. If you `JOIN` to check existence and the other side has duplicates, you get duplicate rows in your result — a common source of inflated totals.

### Fan-out — the silent total inflator

```sql
-- Order has 1 row, OrderLine has 5.
-- SUM(o.ShippingCost) counts shipping FIVE TIMES.
SELECT SUM(o.ShippingCost), SUM(ol.LineTotal)
FROM Orders AS o JOIN OrderLines AS ol ON ol.OrderId = o.OrderId;
```

**Fix:** aggregate each grain separately, then join the aggregates.

```sql
SELECT o.Shipping, l.LineTotal
FROM (SELECT SUM(ShippingCost) AS Shipping FROM Orders) AS o
CROSS JOIN (SELECT SUM(LineTotal) AS LineTotal FROM OrderLines) AS l;
```

> **This is why grain matters.** Every fan-out bug is a grain mismatch wearing a disguise.

---

## Window functions

The highest return per hour of any T-SQL topic. Aggregates and rankings **without collapsing rows**.

```sql
<function>() OVER (
    PARTITION BY <reset groups>
    ORDER BY     <ordering within group>
    ROWS/RANGE BETWEEN <frame>
)
```

### Ranking

```sql
SELECT ProductName, Revenue,
       ROW_NUMBER() OVER (ORDER BY Revenue DESC) AS RowNum,     -- 1,2,3,4
       RANK()       OVER (ORDER BY Revenue DESC) AS Rnk,        -- 1,1,3,4 (gaps)
       DENSE_RANK() OVER (ORDER BY Revenue DESC) AS DenseRnk,   -- 1,1,2,3 (no gaps)
       NTILE(4)     OVER (ORDER BY Revenue DESC) AS Quartile
FROM mart.MasterSales;
```

### Deduplication — the pattern you'll use constantly

```sql
WITH Ranked AS (
    SELECT *,
           rn = ROW_NUMBER() OVER (
                    PARTITION BY ProductId, LoadDate   -- what makes a duplicate
                    ORDER BY     LoadedUtc DESC        -- which one to keep
                )
    FROM raw.SalesExtract
)
DELETE FROM Ranked WHERE rn > 1;
```

Yes, you can `DELETE` from a CTE. This is the cleanest dedup idiom in T-SQL.

### Running totals

```sql
SELECT DataYear, ProductName, Revenue,
       SUM(Revenue) OVER (
           PARTITION BY ProductName
           ORDER BY     DataYear
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS RunningTotal
FROM mart.MasterSales;
```

> **Always specify `ROWS BETWEEN ...` explicitly.** The default frame when you have `ORDER BY` is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which handles ties differently *and* is significantly slower (it can spool to disk). `ROWS` is what you almost always mean.

### Comparing to other rows

```sql
SELECT DataYear, Revenue,
       PrevYear   = LAG(Revenue)  OVER (ORDER BY DataYear),
       NextYear   = LEAD(Revenue) OVER (ORDER BY DataYear),
       YoYChange  = Revenue - LAG(Revenue) OVER (ORDER BY DataYear),
       YoYPct     = 100.0 * (Revenue - LAG(Revenue) OVER (ORDER BY DataYear))
                    / NULLIF(LAG(Revenue) OVER (ORDER BY DataYear), 0),
       FirstEver  = FIRST_VALUE(Revenue) OVER (ORDER BY DataYear),
       LatestEver = LAST_VALUE(Revenue)  OVER (ORDER BY DataYear
                        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
FROM mart.MasterSales;
```

**`LAST_VALUE` needs the explicit full frame**, or the default frame stops at the current row and it just returns the current value. Classic gotcha.

### Percent of group

```sql
SELECT RegionName, ProductName, Revenue,
       PctOfRegion = 100.0 * Revenue
                   / SUM(Revenue) OVER (PARTITION BY RegionName)
FROM mart.MasterSales;
```

Note: no `ORDER BY` in the `OVER` clause means the frame is the whole partition — exactly right for a denominator.

### Performance note

A window function needs its data sorted by `PARTITION BY, ORDER BY`. An index in that order eliminates the sort operator entirely, which is often the difference between fast and not.

---

## APPLY

Joins to something computed **per row of the left side**.

`CROSS APPLY` ≈ inner join. `OUTER APPLY` ≈ left join.

### Top N per group

The signature use case.

```sql
SELECT r.RegionName, t.ProductName, t.Revenue
FROM   ref.RegionMap AS r
CROSS APPLY (
    SELECT TOP 3 ms.ProductName, ms.Revenue
    FROM   mart.MasterSales AS ms
    WHERE  ms.RegionName = r.RegionName
    ORDER  BY ms.Revenue DESC
) AS t;
```

Compare to the `ROW_NUMBER` version — both work; `APPLY` is often faster when the outer set is small and there's a supporting index, because it seeks per outer row instead of ranking everything.

### Calling an inline TVF per row

```sql
SELECT s.Id, m.Margin
FROM      Sales AS s
CROSS APPLY dbo.fn_Margin(s.Revenue, s.Cost) AS m;
```

The replacement for scalar UDFs. → [why](02-objects.md#functions--the-scalar-udf-landmine)

### Unpivoting inline

```sql
SELECT s.OrderId, v.Kind, v.Amount
FROM   Sales AS s
CROSS APPLY (VALUES
    ('Gross', s.GrossAmount),
    ('Tax',   s.TaxAmount),
    ('Net',   s.NetAmount)
) AS v(Kind, Amount);
```

Much more readable than `UNPIVOT`, and handles different data types and expressions.

### Reusing a calculation

```sql
SELECT s.Id, c.Total, c.Total * 0.1 AS Commission
FROM   Sales AS s
CROSS APPLY (SELECT Total = s.Qty * s.UnitPrice) AS c;
```

Lets you name an expression once and reference it multiple times in the same `SELECT` — something SQL otherwise won't let you do.

---

## Dates and time

### The rules

1. **Store UTC** (`datetime2` or `datetimeoffset`). Convert for display only.
2. **Use `date` when you don't need time.** Smaller, and no boundary bugs.
3. **`datetime2`, not `datetime`.** `datetime` has 3.33ms precision and a weird range.
4. **Never store local time** unless you also store the offset — DST will get you.
5. **Use half-open ranges.** Always.

### Format literals

```sql
'2025-01-15'              -- ✅ unambiguous for date/datetime2
'2025-01-15T14:30:00'     -- ✅ ISO 8601, unambiguous always
'20250115'                -- ✅ unambiguous for datetime
'01/15/2025'              -- ❌ depends on language settings — breaks silently
```

### The building blocks

```sql
SYSUTCDATETIME()                     -- UTC, datetime2 — use this
SYSDATETIME()                        -- server local
GETUTCDATE() / GETDATE()             -- legacy datetime versions
DATEFROMPARTS(2025, 1, 1)            -- build a date safely
DATEADD(month, -1, @d)
DATEDIFF(day, @a, @b)                -- counts BOUNDARIES crossed, not full units
EOMONTH(@d)                          -- last day of month
DATETRUNC(month, @d)                 -- SQL Server 2022+
```

> **`DATEDIFF` counts boundaries crossed, not elapsed units.** `DATEDIFF(year, '2025-12-31', '2026-01-01')` = 1, despite being one day apart. Never use it for age or duration without care.

### Period boundaries

```sql
-- year
DECLARE @Start date = DATEFROMPARTS(@Year, 1, 1),
        @End   date = DATEFROMPARTS(@Year + 1, 1, 1);

-- month (pre-2022)
DECLARE @MStart date = DATEADD(month, DATEDIFF(month, 0, @AnyDate), 0),
        @MEnd   date = DATEADD(month, DATEDIFF(month, 0, @AnyDate) + 1, 0);

-- month (2022+)
DECLARE @MStart2 date = DATETRUNC(month, @AnyDate);

WHERE d >= @Start AND d < @End      -- always half-open
```

### Time zones (2016+)

```sql
SELECT @utc AT TIME ZONE 'UTC' AT TIME ZONE 'Eastern Standard Time';
```

Handles DST correctly using the OS time zone database. The double `AT TIME ZONE` is required: the first tells SQL Server what the value *is*, the second converts it.

### Calendar table

For anything with fiscal periods, business days, or holidays, build a date dimension:

```sql
CREATE TABLE ref.Calendar
(
    [Date]        date PRIMARY KEY,
    [Year]        int  NOT NULL,
    [Quarter]     int  NOT NULL,
    [Month]       int  NOT NULL,
    MonthName     varchar(10) NOT NULL,
    [DayOfWeek]   int  NOT NULL,
    IsWeekend     bit  NOT NULL,
    IsHoliday     bit  NOT NULL,
    FiscalYear    int  NOT NULL,
    FiscalQuarter int  NOT NULL,
    FiscalPeriod  int  NOT NULL
);
```

Populate once for 50 years. Now every fiscal rule lives in one table instead of scattered through expressions, joins are sargable, and changing the fiscal calendar is a data change rather than a code change. **Worth doing early** if fiscal periods are in your future at all.

---

## Strings

```sql
CONCAT(a, b, c)              -- NULL-safe; treats NULL as ''
CONCAT_WS('-', a, b, c)      -- with separator (2017+)
STRING_AGG(col, ', ')        -- aggregate rows into one string (2017+)
    WITHIN GROUP (ORDER BY col)
STRING_SPLIT(@csv, ',')      -- string to rows (2016+)
TRIM(col)                    -- both ends (2017+)
REPLACE / LEFT / RIGHT / SUBSTRING / CHARINDEX / LEN / DATALENGTH
FORMAT(@d, 'yyyy-MM-dd')     -- flexible, SLOW — avoid in large result sets
```

**`LEN` vs `DATALENGTH`:** `LEN` ignores trailing spaces and counts characters; `DATALENGTH` counts bytes (so `nvarchar` returns double). For "is this empty?", `DATALENGTH(col) = 0` is the reliable check.

**`FORMAT` is slow** — it calls into .NET CLR per row. Fine for a handful of rows, genuinely painful over a million. Use `CONVERT` with a style code for bulk work.

**`QUOTENAME`** for dynamic SQL identifiers — the injection-safe way to embed a table or column name.

---

## Aggregation patterns

### Conditional aggregation — better than PIVOT

```sql
SELECT DataYear,
       NorthRevenue = SUM(CASE WHEN RegionName = 'North' THEN Revenue END),
       SouthRevenue = SUM(CASE WHEN RegionName = 'South' THEN Revenue END),
       ActiveCount  = COUNT(CASE WHEN Status = 'Active' THEN 1 END)
FROM   mart.MasterSales
GROUP  BY DataYear;
```

Note the `CASE` without `ELSE` — it returns NULL, and aggregates skip NULLs. More readable and more flexible than `PIVOT`, and you can mix different aggregate functions in one pass.

### `GROUPING SETS` — multiple grain levels in one query

```sql
SELECT DataYear, RegionName, SUM(Revenue) AS Revenue,
       IsSubtotal = GROUPING(RegionName)
FROM   mart.MasterSales
GROUP  BY GROUPING SETS ( (DataYear, RegionName), (DataYear), () );
```

Detail rows, per-year subtotals, and a grand total in a single scan. `GROUPING()` returns 1 for the aggregated-away column so you can tell a subtotal row from a detail row with a NULL in it.

`ROLLUP` and `CUBE` are shorthand for common grouping-set combinations.

### `HAVING` vs `WHERE`

```sql
SELECT ProductName, SUM(Revenue) AS Total
FROM   mart.MasterSales
WHERE  DataYear = 2025          -- filters rows (before grouping) — do this when you can
GROUP  BY ProductName
HAVING SUM(Revenue) > 10000;    -- filters groups (after aggregation) — only for aggregates
```

---

## Gaps and islands

Finding consecutive runs and the holes between them. Comes up constantly: uptime windows, date continuity, missing sequence numbers.

```sql
-- islands: group consecutive dates into runs
WITH Numbered AS (
    SELECT [Date],
           grp = DATEADD(day,
                    -ROW_NUMBER() OVER (ORDER BY [Date]),
                    [Date])
    FROM   dbo.ActivityDates
)
SELECT StartDate = MIN([Date]),
       EndDate   = MAX([Date]),
       DayCount  = COUNT(*)
FROM   Numbered
GROUP  BY grp
ORDER  BY StartDate;
```

**The trick:** for consecutive dates, `date - row_number` is constant. That constant becomes the group key. Elegant once you've seen it; impossible to derive from scratch.

```sql
-- gaps: find missing values in a sequence
SELECT GapStart = t.Id + 1,
       GapEnd   = (SELECT MIN(Id) FROM dbo.T AS x WHERE x.Id > t.Id) - 1
FROM   dbo.T AS t
WHERE  NOT EXISTS (SELECT 1 FROM dbo.T AS n WHERE n.Id = t.Id + 1)
  AND  t.Id < (SELECT MAX(Id) FROM dbo.T);
```

---

## Pivot and unpivot

```sql
-- PIVOT (rigid: column list must be known at write time)
SELECT * FROM (
    SELECT DataYear, RegionName, Revenue FROM mart.MasterSales
) AS src
PIVOT (SUM(Revenue) FOR RegionName IN ([North],[South],[East],[West])) AS pvt;
```

> **Conditional aggregation is usually better** — more flexible, mixes aggregates, and doesn't require the rigid `IN` list. Reach for `PIVOT` only when it genuinely reads more clearly.

**Dynamic pivot** (unknown columns) requires dynamic SQL, and is usually a sign the work belongs in the presentation layer instead. A reporting tool pivots better than SQL does.

```sql
-- UNPIVOT — but CROSS APPLY (VALUES ...) is more readable and more capable
SELECT OrderId, Kind, Amount
FROM Sales
UNPIVOT (Amount FOR Kind IN (GrossAmount, TaxAmount, NetAmount)) AS u;
```

---

## Error handling

```sql
CREATE OR ALTER PROCEDURE dbo.Example
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;

    BEGIN TRY
        BEGIN TRAN;
            -- work
        COMMIT;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK;

        INSERT dbo.ErrorLog (ProcName, ErrNum, ErrMsg, ErrLine, OccurredUtc)
        VALUES (OBJECT_NAME(@@PROCID), ERROR_NUMBER(), ERROR_MESSAGE(),
                ERROR_LINE(), SYSUTCDATETIME());

        THROW;   -- re-raise with original number and severity
    END CATCH
END
```

### Error functions (valid only inside `CATCH`)

`ERROR_NUMBER()` · `ERROR_MESSAGE()` · `ERROR_SEVERITY()` · `ERROR_STATE()` · `ERROR_LINE()` · `ERROR_PROCEDURE()`

### Custom errors

```sql
THROW 50001, 'Refresh already running for this slice.', 1;
```

User-defined error numbers must be ≥ 50000. `THROW` requires the preceding statement to end with a semicolon — a genuinely annoying parser quirk that produces confusing errors.

### What `TRY/CATCH` won't catch

- Compile errors (syntax, invalid object names in the same batch)
- Severity 20+ errors that terminate the connection
- Attention/timeout from the client

**Log the error, then re-throw.** Swallowing errors produces silently wrong data, which is far worse than a loud failure — a job that fails gets investigated; a job that "succeeds" while doing nothing does not.

---

## Dynamic SQL

Sometimes necessary. Do it safely.

```sql
DECLARE @sql nvarchar(max), @TableName sysname = 'MasterSales';

SET @sql = N'SELECT DataYear, SUM(Revenue) AS Revenue
             FROM mart.' + QUOTENAME(@TableName) + N'
             WHERE DataYear = @Year
             GROUP BY DataYear;';

EXEC sys.sp_executesql @sql, N'@Year int', @Year = 2025;
```

**Two rules, no exceptions:**

1. **`QUOTENAME()` for identifiers** (table/column names) — they can't be parameters, so they must be escaped.
2. **`sp_executesql` with parameters for values** — never string-concatenate a value into SQL.

```sql
-- ❌ SQL injection, and no plan reuse
EXEC('SELECT * FROM t WHERE Name = ''' + @Name + '''');

-- ✅ safe and cacheable
EXEC sp_executesql N'SELECT * FROM t WHERE Name = @Name', N'@Name nvarchar(100)', @Name;
```

Beyond injection, parameterized dynamic SQL gets **plan reuse**; concatenated SQL produces a new plan for every distinct string, bloating the plan cache.

---

## Useful system queries

```sql
-- find every object referencing a table/column
SELECT DISTINCT o.name, o.type_desc
FROM   sys.sql_expression_dependencies AS d
JOIN   sys.objects AS o ON o.object_id = d.referencing_id
WHERE  d.referenced_entity_name = 'MasterSales';

-- search all procedure/view/function definitions for a string
SELECT OBJECT_SCHEMA_NAME(object_id) + '.' + OBJECT_NAME(object_id) AS ObjName,
       o.type_desc
FROM   sys.sql_modules AS m
JOIN   sys.objects AS o ON o.object_id = m.object_id
WHERE  m.definition LIKE '%SearchTerm%';

-- table sizes
SELECT s.name + '.' + t.name AS TableName,
       p.rows,
       SizeMB = SUM(a.total_pages) * 8 / 1024
FROM   sys.tables AS t
JOIN   sys.schemas AS s ON s.schema_id = t.schema_id
JOIN   sys.indexes AS i ON i.object_id = t.object_id
JOIN   sys.partitions AS p ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN   sys.allocation_units AS a ON a.container_id = p.partition_id
WHERE  i.index_id IN (0,1)
GROUP  BY s.name, t.name, p.rows
ORDER  BY SizeMB DESC;

-- all indexes on a table, with their key and included columns
SELECT i.name AS IndexName, i.type_desc, i.is_unique, i.filter_definition,
       KeyCols = STUFF((SELECT ', ' + c.name
                        FROM sys.index_columns AS ic
                        JOIN sys.columns AS c
                          ON c.object_id = ic.object_id AND c.column_id = ic.column_id
                        WHERE ic.object_id = i.object_id AND ic.index_id = i.index_id
                          AND ic.is_included_column = 0
                        ORDER BY ic.key_ordinal
                        FOR XML PATH('')), 1, 2, ''),
       InclCols = STUFF((SELECT ', ' + c.name
                        FROM sys.index_columns AS ic
                        JOIN sys.columns AS c
                          ON c.object_id = ic.object_id AND c.column_id = ic.column_id
                        WHERE ic.object_id = i.object_id AND ic.index_id = i.index_id
                          AND ic.is_included_column = 1
                        FOR XML PATH('')), 1, 2, '')
FROM   sys.indexes AS i
WHERE  i.object_id = OBJECT_ID('mart.MasterSales');

-- currently running queries
SELECT r.session_id, r.status, r.command, r.wait_type, r.blocking_session_id,
       r.percent_complete, r.total_elapsed_time, t.text
FROM   sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE  r.session_id <> @@SPID;
```

---

[← Back to index](README.md) · [Next: Project Layout →](07-project-layout.md)
