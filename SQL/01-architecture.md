# 01 — Architecture & Layering

[← Back to index](README.md) · [← Glossary](00-glossary.md) · [Next: Objects →](02-objects.md)

Where data lives, why it lives there, and how to decide.

---

## Contents

- [The three-layer model](#the-three-layer-model)
- [Schemas](#schemas)
- [One database or several](#one-database-or-several)
- [Synonyms](#synonyms)
- [Designing a master table](#designing-a-master-table)
- [Grain, in depth](#grain-in-depth)
- [Keys on a master table](#keys-on-a-master-table)
- [The view layer](#the-view-layer)
- [Manual overrides](#manual-overrides)

---

## The three-layer model

```
┌─────────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐
│  SSIS   │────▶│   raw.*  │────▶│  mart.*   │────▶│ mart.vw_*│────▶ consumers
│  files  │     │  landing │     │  master   │     │  views   │
│  APIs   │     │          │     │           │     │          │
└─────────┘     └──────────┘     └───────────┘     └──────────┘
                      ▲                ▲                 ▲
                   dumb,          derived,          the contract
                 immutable      rebuildable      nothing else is touched
                                     ▲
                              ┌──────┴──────┐
                              │    ref.*    │
                              │ static maps │
                              └─────────────┘
```

**Layer 1 — `raw`: the landing zone**

Dumb by design. No transformation, no cleanup, no joins, no business logic. Its only job is to hold what the source gave you, in a shape close to what the source gave you.

Why this discipline matters: raw is your replay capability. If you clean data on the way in, and later discover the cleaning rule was wrong, the original values are gone. If you land it dirty and clean it downstream, you fix the rule and re-run.

Rules:
- Column types match the source's real types (not everything `nvarchar(255)`)
- Add a load-audit column: `LoadedUtc`, and ideally `LoadBatchId`
- Truncate-and-reload, or append-only. Never in-place update.
- No foreign keys to `ref` — raw must be able to hold a bad ProductId, because that's a fact about your source data you need to see

**Layer 2 — `ref`: the static maps**

Lookup tables, code translations, hierarchies. In dimensional terms, these are your **dimensions**. Small, slow-changing, and — importantly — **part of your schema's meaning**, which is why their contents belong in source control alongside the DDL.

**Layer 3 — `mart`: the master tables**

Derived, denormalized, query-optimized. Built by a procedure from `raw` joined to `ref`. Fully rebuildable at any time. This is what people actually query.

**Layer 4 — `mart.vw_*`: the contract**

The only thing consumers touch. → [details below](#the-view-layer)

**Optional — `stg`: scratch space**

When a transform needs intermediate steps that don't fit in one statement. Contents are disposable between runs. If you don't need it, don't create it.

---

## Schemas

### What a schema actually is

A namespace. A folder for objects. `mart.MasterSales` means "the object named `MasterSales` in the container named `mart`." Without one you get `dbo`, which is just the default container everything lands in.

That's the whole concept. It gets used constantly and explained almost never.

### Why bother

1. **Visual grouping.** With 40 objects, `dbo.` everywhere is one alphabetical wall in SSMS with no indication of what's a landing table versus what reports read. With schemas they group, and the name tells you the layer.
2. **The name carries meaning.** `raw.SalesExtract` tells a reader "this is unprocessed" without a comment.
3. **Permission boundary.** You can grant `SELECT` on the `mart` schema and nothing else. One statement, covers every current and future object in it.
4. **Ownership chaining works cleanly** when objects share a schema owner.

### Cost

`CREATE SCHEMA raw;` — that's it. Do it once at the start.

```sql
CREATE SCHEMA raw   AUTHORIZATION dbo;   -- landing zone
CREATE SCHEMA ref   AUTHORIZATION dbo;   -- static maps
CREATE SCHEMA stg   AUTHORIZATION dbo;   -- optional scratch
CREATE SCHEMA mart  AUTHORIZATION dbo;   -- master + views
GO
```

### Always two-part-name everything

```sql
SELECT * FROM mart.MasterSales;   -- ✅
SELECT * FROM MasterSales;        -- ❌
```

Omitting the schema forces a name-resolution step on every compile and can produce different plan cache entries per user. Two-part naming is free and always correct.

---

## One database or several

### Default: one database, multiple schemas

Reasons, in order of how much they'll bite you:

**Transactions.** Your refresh procedure reads `raw` and writes `mart` atomically. The moment those are on separate *instances*, you need DTC — a separate service, slower, more failure modes. (Separate databases on the *same* instance still handle transactions fine, but it's one more thing.)

**Backup and restore consistency.** One backup, one point in time. Two databases restored to slightly different moments means master disagrees with raw, and you get to explain why.

**Foreign keys don't cross databases.** If `mart` needs to reference `ref`, they must be in the same database for the engine to enforce it.

**Ownership chaining breaks across databases.** Cross-database permission chains require either matching database owners or `TRUSTWORTHY`, which is a security smell. Real friction.

**Simpler everything.** One connection string, one set of permissions, one restore procedure.

### When to actually split

Concrete reasons only:

| Reason | Why it justifies a split |
|---|---|
| **Different retention** | Raw purged at 90 days, master kept forever — different backup and archive policies |
| **Wildly different size** | Raw is 2 TB, master is 3 GB — you may want them on different storage |
| **Different HA/DR requirements** | Master needs an Availability Group, raw doesn't |
| **Different physical server** | Master must live on the reporting server |
| **Different security domain** | Genuinely different audiences with no overlap |
| **Separate product** | It's a different application, not a different layer of the same one |

"It feels cleaner" is not on this list.

### The instance-level view

```
Instance
 ├── MyDataDb           ← your raw + ref + mart, all schemas
 ├── ReportingDb        ← only if you have a reason above
 └── master/msdb/model/tempdb   ← system, don't touch except msdb for Agent
```

---

## Synonyms

A nickname for an object.

```sql
CREATE SYNONYM dbo.MasterSales FOR mart.MasterSales;
```

Now `dbo.MasterSales` and `mart.MasterSales` are the same table. The only reason to care: **if the object ever moves, you repoint the synonym and every query using the nickname keeps working.**

```sql
-- later, when it moves to another database
DROP SYNONYM dbo.MasterSales;
CREATE SYNONYM dbo.MasterSales FOR ReportingDb.mart.MasterSales;
-- every proc that said FROM dbo.MasterSales still works, unchanged
```

**When to actually use them:**
- You genuinely expect a move and want it to be a one-line change
- You're consolidating from a legacy naming scheme without rewriting everything at once
- Cross-database references you want to isolate to one definition

**When to skip them:** Early on, at small scale. They're insurance against a move you may never make, and they add an indirection layer that makes "where does this actually live?" harder to answer. Know the word; use it when the need appears.

---

## Designing a master table

### One table per grain, not one table total

The only question that decides how many master tables you need:

> **Can every row answer the same "one row per what?" question?**

- Sales and inventory both roll up to year + product + region → **one table**, more columns.
- Sales is one-row-per-year-per-product, inventory is one-row-per-month-per-warehouse → **two tables**. Forcing them together produces NULLs everywhere and totals that double-count when someone joins them.

**Wide is fine.** Forty columns at a single consistent grain is completely normal and much easier to work with than four narrow tables you have to keep re-joining. Denormalization is the point of this layer.

### The shape

```sql
CREATE TABLE mart.MasterSales
(
    MasterSalesId   int IDENTITY(1,1)  NOT NULL,

    -- the grain
    DataYear        int                NOT NULL,
    ProductName     varchar(100)       NOT NULL,
    RegionName      varchar(100)       NOT NULL,

    -- the measures
    Qty             decimal(18,2)      NOT NULL,
    Revenue         decimal(18,2)      NOT NULL,
    OrderCount      int                NOT NULL,

    -- audit
    RefreshedUtc    datetime2(0)       NOT NULL
        CONSTRAINT DF_MasterSales_RefreshedUtc DEFAULT SYSUTCDATETIME(),
    SourceRowCount  int                NULL,

    CONSTRAINT PK_MasterSales PRIMARY KEY CLUSTERED (MasterSalesId),

    CONSTRAINT UQ_MasterSales_Grain
        UNIQUE (DataYear, ProductName, RegionName),

    CONSTRAINT CK_MasterSales_Year
        CHECK (DataYear BETWEEN 1900 AND 2999)
);
GO

CREATE NONCLUSTERED INDEX IX_MasterSales_Year_Region
    ON mart.MasterSales (DataYear, RegionName)
    INCLUDE (ProductName, Qty, Revenue);
GO
```

**Why `SourceRowCount`:** so you can sanity-check that a refresh didn't silently lose half the data. A refresh that produces 12 rows instead of 4,000 should be visible without anyone noticing a wrong number on a report first.

**Why `RefreshedUtc`:** "is this data stale?" is the first question anyone asks about a report. Answer it in the data.

**Why UTC:** because your server will eventually move, or DST will bite you at 2am on a Sunday. Store UTC, convert for display.

---

## Grain, in depth

Grain is the most important word in this entire reference. It deserves its own treatment.

### Stating it

Write it as a sentence, in the README and in an extended property on the table:

```sql
EXEC sys.sp_addextendedproperty
     @name = N'Grain',
     @value = N'One row per DataYear, ProductName, RegionName.',
     @level0type = N'SCHEMA', @level0name = N'mart',
     @level1type = N'TABLE',  @level1name = N'MasterSales';
```

### Enforcing it

```sql
CONSTRAINT UQ_MasterSales_Grain UNIQUE (DataYear, ProductName, RegionName)
```

This is the constraint that catches a duplicate-refresh bug *before* it reaches a report. Without it, a race between two refresh runs silently doubles your numbers and nobody finds out until month-end.

### Why violating it is so damaging

Fan-out. If `mart.MasterSales` accidentally has two rows for (2025, Widget, North), then:

```sql
SELECT SUM(Revenue) FROM mart.MasterSales WHERE DataYear = 2025;
```

...is wrong. And every join to that table multiplies rows on the other side too. The error propagates silently, and the only symptom is a number that's too big — which looks like good news until someone checks.

### Changing grain later

You can't, cleanly. Changing grain means rebuilding the table and revalidating every query and report that depends on it. This is why the "will anyone need month-level?" question is worth asking *before* you build year-level.

If you're unsure, **build at the finer grain and aggregate up in a view.** Going from monthly to yearly is a `GROUP BY`. Going from yearly to monthly is impossible without reloading history.

---

## Keys on a master table

Two keys doing genuinely different jobs:

**`MasterSalesId` (surrogate, `IDENTITY`)** — a single stable integer column for front-end binding. Grids, data-bound controls, and ORMs all want exactly this: one column, integer, unique.

**`UQ_..._Grain` (the natural/business key)** — the database refusing to let a bug create two rows for the same logical entity.

### The IDENTITY caveat you must know

If your refresh does delete-and-reinsert, **IDs regenerate on every refresh.** Row 4471 today is not row 4471 tomorrow.

That's fine for grid binding within a page load. It is **not** fine for:
- Bookmarks or saved links pointing at a specific row
- A "favorites" feature storing IDs
- Any external system holding a reference

If you need durable references, key them on the **grain columns**, not the surrogate. Or switch the refresh from delete-and-reinsert to a proper merge that preserves IDs — but that's more complexity, so only do it if the requirement is real.

---

## The view layer

### The rule

**Consumers only ever read from `mart.vw_*`.** No direct table access. No ad-hoc SQL against `raw`.

Once that's true, everything behind the views is yours to change freely. That's the actual payoff — bigger than any single design decision above it.

### What it buys you

| Without views | With views |
|---|---|
| Rename a column → hunt through app code | Rename a column, adjust the view, app unchanged |
| Split a table → rewrite every query | Split a table, view joins them back, app unchanged |
| "What does the app depend on?" → grep the codebase | Read the view definitions |
| Presentation logic in C# | Presentation logic in SQL, where the data is |
| Every consumer sees every column | Views expose only what's needed |

### Writing them

```sql
CREATE OR ALTER VIEW mart.vw_SalesByRegion
AS
SELECT
    ms.MasterSalesId,
    ms.DataYear,
    ms.RegionName,
    ms.ProductName,
    ms.Qty,
    ms.Revenue,
    ms.RefreshedUtc
FROM mart.MasterSales AS ms;
GO
```

**Three things to get right:**

**1. Name columns explicitly. Never `SELECT *` inside a view.**
A `SELECT *` view captures the column list at creation time and doesn't pick up new columns until you run `sp_refreshview`. Worse, it can silently shift column *order* if the underlying table changes, breaking any binding that goes by position. This is a real, common, hard-to-diagnose bug.

**2. Don't nest views more than one level deep.**
View on view on view is the single most reliable way to produce an execution plan nobody can debug. The optimizer expands them all and the result is unreadable. If you need a second layer of logic, write a new *flat* view over the base tables.

**3. A view is not a performance feature.**
Creating a view caches nothing and precomputes nothing. `SELECT * FROM MyView WHERE x = 1` performs exactly the same as pasting the view body inline. Filtering against a view works normally — the optimizer folds your `WHERE` into the underlying query and still uses your indexes.

The exception is an **indexed view** (requires `SCHEMABINDING` and a unique clustered index), which genuinely materializes to disk and auto-maintains. Real tool, real write overhead, many restrictions. Reach for it only with evidence.

### When you need a parameter

Use an **inline table-valued function**, not a stored procedure:

```sql
CREATE OR ALTER FUNCTION mart.fn_SalesForYear (@Year int)
RETURNS TABLE
AS RETURN
(
    SELECT MasterSalesId, ProductName, RegionName, Qty, Revenue
    FROM mart.MasterSales
    WHERE DataYear = @Year
);
GO

SELECT * FROM mart.fn_SalesForYear(2025);
```

An inline TVF is expanded into the calling query and optimized as one unit — it performs like the query you'd have written by hand. Unlike a procedure, you can still join it, filter it further, and compose it into larger queries.

> **Rule:** need a shape → view. Need a shape with a parameter → inline TVF. Need to change data or do multiple steps → stored procedure.

---

## Manual overrides

The instant someone hand-edits a value in a master table, it stops being derivable and the next refresh destroys their work.

But sometimes a human genuinely does know better than the source system. Handle it with a separate table the refresh joins in:

```sql
CREATE TABLE mart.SalesOverride
(
    DataYear      int          NOT NULL,
    ProductName   varchar(100) NOT NULL,
    RegionName    varchar(100) NOT NULL,
    Revenue       decimal(18,2)    NULL,   -- NULL = don't override this measure
    Reason        nvarchar(400) NOT NULL,
    CreatedBy     sysname      NOT NULL DEFAULT SUSER_SNAME(),
    CreatedUtc    datetime2(0) NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_SalesOverride PRIMARY KEY (DataYear, ProductName, RegionName)
);
```

Then in the refresh:

```sql
SELECT ...,
       Revenue = COALESCE(o.Revenue, agg.Revenue)
FROM   aggregated AS agg
LEFT JOIN mart.SalesOverride AS o
       ON  o.DataYear    = agg.DataYear
       AND o.ProductName = agg.ProductName
       AND o.RegionName  = agg.RegionName
```

Now: overrides survive refreshes, they're auditable (who, when, why), and master remains fully derivable — from raw *plus* overrides, both of which are durable inputs.

---

[← Back to index](README.md) · [Next: Objects →](02-objects.md)
