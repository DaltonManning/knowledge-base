# 07 — Project Layout & Deployment

[← Back to index](README.md) · [← T-SQL Cookbook](06-tsql-cookbook.md) · [Next: Operations →](08-operations.md)

Your SQL lives in git. Here's how it's organized and how it gets to production.

---

## Contents

- [The organizing principle](#the-organizing-principle)
- [Folder structure](#folder-structure)
- [Should the tool create your tables](#should-the-tool-create-your-tables)
- [Re-runnable script patterns](#re-runnable-script-patterns)
- [Migrations](#migrations)
- [Deployment approaches](#deployment-approaches)
- [Environments and drift](#environments-and-drift)
- [Naming conventions](#naming-conventions)
- [Code review checklist](#code-review-checklist)
- [Order of operations for a new project](#order-of-operations-for-a-new-project)

---

## The organizing principle

Objects split into two kinds, handled completely differently:

| | Handling | Why |
|---|---|---|
| **Tables** | Created once, then **altered** | You can't re-run `CREATE TABLE` against a live table with data |
| **Procs, views, functions, triggers** | Fully **replaceable** | `CREATE OR ALTER` — re-run any time, no thought |

Internalize that and the folder layout writes itself.

---

## Folder structure

```
/sql
  README.md                       ← grain of each master table, deploy instructions
  deploy.sql                      ← :r every file, in order

  00_schemas/
    schemas.sql

  10_tables/
    raw.SalesExtract.sql
    raw.InventoryExtract.sql
    ref.ProductMap.sql
    ref.RegionMap.sql
    ref.Calendar.sql
    mart.MasterSales.sql
    mart.SalesOverride.sql
    dbo.RefreshLog.sql
    dbo.RefreshQueue.sql
    dbo.LoadRejects.sql

  20_seed/
    ref.ProductMap.data.sql
    ref.RegionMap.data.sql
    ref.Calendar.populate.sql

  30_views/
    mart.vw_SalesByRegion.sql
    mart.vw_DataFreshness.sql
    mart.vw_ReconSales.sql

  40_programmability/
    functions/
      mart.fn_SalesForYear.sql
    procedures/
      mart.RefreshMasterSales.sql
      dbo.ProcessRefreshQueue.sql
    triggers/
      raw.trg_SalesExtract_MarkDirty.sql

  50_security/
    roles.sql
    grants.sql

  90_migrations/
    2026-01-15_add_Revenue_to_MasterSales.sql
    2026-03-02_widen_ProductName.sql
    2026-07-31_add_index_MasterSales_Region.sql

  99_utility/
    healthcheck.sql
    reconcile.sql
```

**Numeric prefixes** exist purely so execution order is obvious: schemas before tables, tables before views, views before things referencing them.

**One file per object.** This is what lets `git log mart.RefreshMasterSales.sql` answer "who changed this and why."

### `deploy.sql`

```sql
:setvar DbName "MyDataDb"
:on error exit

USE [$(DbName)];
GO

:r .\00_schemas\schemas.sql
:r .\10_tables\raw.SalesExtract.sql
:r .\10_tables\ref.ProductMap.sql
-- ... etc
:r .\20_seed\ref.ProductMap.data.sql
:r .\30_views\mart.vw_SalesByRegion.sql
:r .\40_programmability\procedures\mart.RefreshMasterSales.sql
```

Run with `sqlcmd -i deploy.sql -v DbName="MyDataDb"` in SQLCMD mode. `:on error exit` means a failure stops the deploy rather than continuing into an unknown state.

---

## Should the tool create your tables

**No. Write them yourself.** This is the one place worth resisting the easy path.

Four concrete reasons:

**1. Types.** Auto-created tables come out as `nvarchar(255)` for everything, or `float` for numerics. Your entire sargable date-range filter depends on `LoadDate` actually being `date`/`datetime2` — if it lands as a string, that query is a scan forever and you won't know why.

**2. No indexes.** Auto-create never adds them. Your refresh needs an index on `(LoadDate, SourceSystem)` to not be miserable.

**3. Silent drift.** If the tool drops and recreates on each run and the source adds a column, your table shape changes underneath you with nothing in git recording it.

**4. You can't rebuild.** If a new dev box or a test database can't be stood up by running your scripts, **the repo isn't actually the source of truth** — and you're one incident away from discovering that.

### What to do instead

Have SSIS **truncate and insert** into a table *you* defined. Same convenience, none of the above. In SSIS terms: an Execute SQL Task with `TRUNCATE TABLE raw.SalesExtract`, then a Data Flow into the existing table. Never "drop and create."

---

## Re-runnable script patterns

### Tables

```sql
IF OBJECT_ID('mart.MasterSales', 'U') IS NULL
BEGIN
    CREATE TABLE mart.MasterSales
    (
        MasterSalesId int IDENTITY(1,1) NOT NULL,
        DataYear      int               NOT NULL,
        ProductName   varchar(100)      NOT NULL,
        RegionName    varchar(100)      NOT NULL,
        Qty           decimal(18,2)     NOT NULL,
        Revenue       decimal(18,2)     NOT NULL,
        RefreshedUtc  datetime2(0)      NOT NULL
            CONSTRAINT DF_MasterSales_RefreshedUtc DEFAULT SYSUTCDATETIME(),
        CONSTRAINT PK_MasterSales PRIMARY KEY CLUSTERED (MasterSalesId),
        CONSTRAINT UQ_MasterSales_Grain UNIQUE (DataYear, ProductName, RegionName)
    );
END
GO

-- indexes: DROP_EXISTING makes this re-runnable and non-destructive
CREATE NONCLUSTERED INDEX IX_MasterSales_Year_Region
    ON mart.MasterSales (DataYear, RegionName)
    INCLUDE (ProductName, Qty, Revenue)
    WITH (DROP_EXISTING = ON);
GO
```

### Schemas

```sql
IF SCHEMA_ID('mart') IS NULL EXEC('CREATE SCHEMA mart AUTHORIZATION dbo;');
GO
```

`CREATE SCHEMA` must be the first statement in its batch, hence the `EXEC()` wrapper.

### Programmability

```sql
CREATE OR ALTER PROCEDURE mart.RefreshMasterSales ... 
```

`CREATE OR ALTER` (2016 SP1+) preserves permissions and extended properties, unlike drop-and-create. Always use it.

### Seed data

```sql
MERGE ref.RegionMap AS t
USING (VALUES
    (1, 'North'),
    (2, 'South'),
    (3, 'East'),
    (4, 'West')
) AS s (RegionId, RegionName)
   ON t.RegionId = s.RegionId
WHEN MATCHED AND t.RegionName <> s.RegionName
    THEN UPDATE SET RegionName = s.RegionName
WHEN NOT MATCHED BY TARGET
    THEN INSERT (RegionId, RegionName) VALUES (s.RegionId, s.RegionName);
-- deliberately no WHEN NOT MATCHED BY SOURCE: don't delete rows that may be
-- referenced by existing fact data
```

> **Seed data for `ref.*` belongs in source control.** Those static maps are part of your schema's meaning — a fresh database without them is not a working database.
>
> (`MERGE` is fine here: a dozen rows, run manually, no concurrency. → [why to avoid it elsewhere](03-refresh-patterns.md#why-not-merge))

---

## Migrations

**Once the table exists, every change goes in `90_migrations/` as an `ALTER`.**

```sql
-- 90_migrations/2026-01-15_add_Revenue_to_MasterSales.sql
IF NOT EXISTS (
    SELECT 1 FROM sys.columns
    WHERE object_id = OBJECT_ID('mart.MasterSales') AND name = 'Revenue'
)
BEGIN
    ALTER TABLE mart.MasterSales
        ADD Revenue decimal(18,2) NOT NULL
            CONSTRAINT DF_MasterSales_Revenue DEFAULT 0;
END
GO
```

**And you also update the `CREATE TABLE` file** so it stays accurate for fresh builds.

Yes, that's dual maintenance. It's mildly annoying, and it's precisely the annoyance that state-based tools (SSDT) exist to remove — you keep only the desired-state `CREATE TABLE` and the tool computes the `ALTER`. **Worth switching when the manual version starts to grate. Not before.**

### Migration rules

1. **Append-only.** Once a migration has run in production, never edit it. Write another.
2. **Date-prefixed names** so ordering is obvious.
3. **Guarded**, so re-running is safe.
4. **One logical change per file.**
5. **Test the rollback**, or at least know what it would be.

### Breaking vs backward-compatible changes

| Backward compatible | Breaking |
|---|---|
| Add a nullable column | Drop a column |
| Add a table | Rename a column |
| Add an index | Narrow a type (`varchar(100)` → `varchar(50)`) |
| Widen a type | Add `NOT NULL` without a default |
| Add a default | Change a column's meaning |

**For breaking changes, use expand-and-contract:**

1. Add the new column alongside the old
2. Write to both; backfill the new one
3. Migrate readers to the new column
4. Wait a release
5. Drop the old column

Slower, and it's how you change schema without a maintenance window or a broken app.

> **Views make this dramatically easier.** If consumers read `mart.vw_SalesByRegion` rather than the table, you can do all five steps behind the view and nobody outside notices. That's the payoff for the view layer.

---

## Deployment approaches

### State-based (SSDT / DACPAC)

You maintain `CREATE TABLE` scripts describing the **desired end state**. The tool compares against the target and generates the `ALTER` script.

**Pros:** no dual maintenance · schema is always readable as current state · built-in Schema Compare · free with Visual Studio · works with VS Code's MSSQL extension (Schema Designer + Schema Compare are GA there now)

**Cons:** generated scripts occasionally do table rebuilds you didn't expect · data motion needs pre/post-deploy scripts · **you must read the generated diff before applying it**

### Migration-based (Flyway / DbUp / Liquibase)

You write each change step yourself in order. The tool tracks which have been applied.

**Pros:** total control over exactly what runs · obvious ordering · simple mental model · natural fit for app-developer workflows

**Cons:** no single file shows current state · more typing · easy to write a migration that isn't idempotent

### Which

| Situation | Lean toward |
|---|---|
| Solo or small team, database-first | **Hand-written scripts** — start here |
| Multiple environments, tired of hand-carrying | **SSDT** |
| App developers own the schema, CI/CD pipeline | **Flyway / DbUp** |
| Lots of data motion during schema change | **Migration-based** |

**Pick one and be consistent.** Mixing approaches is where the pain lives.

**The actual trigger to graduate from hand-written scripts to SSDT** is having a dev *and* a prod database and being tired of hand-carrying changes between them. Not table count.

---

## Environments and drift

### The minimum

```
Dev   → your machine or a dev instance. SQL Server Developer Edition is free
        and has every Enterprise feature. Use it.
Prod  → the real one.
```

Add **Test/Staging** when you have users who'd notice a bad deploy.

### Drift

Production no longer matches source control, because someone changed it by hand.

**Detecting it:** Schema Compare (SSDT, the VS Code MSSQL extension, or Redgate) run against prod on a schedule. Anything that differs is either drift or an un-deployed change — both worth knowing about.

**Preventing it** is cultural, not technical:
- Nobody has `db_owner` on prod for routine work
- Changes go through the repo
- Emergency hotfixes are backported to the repo **the same day**, not "when things calm down"

**The rule:** if a new environment can't be built by running your scripts, the repo is not the source of truth, and you'll find that out during an incident.

### `.gitignore`

```
*.bak
*.trn
*.mdf
*.ldf
*.dacpac
bin/
obj/
*.user
config.local.*
```

**Never commit connection strings with credentials.** Use a template file plus an ignored local override.

---

## Naming conventions

Pick a set, write it down, apply it consistently. What matters is consistency, not which convention you chose.

```
Schemas          raw, ref, stg, mart, dbo
Tables           PascalCase singular:  MasterSales, ProductMap
Columns          PascalCase:           DataYear, RefreshedUtc
Primary key      PK_TableName
Unique           UQ_TableName_Purpose
Foreign key      FK_ChildTable_ParentTable
Check            CK_TableName_Rule
Default          DF_TableName_ColumnName
Index            IX_TableName_Col1_Col2
Views            vw_Purpose
Procedures       Verb + Noun:          RefreshMasterSales, ProcessRefreshQueue
Functions        fn_Purpose
Triggers         trg_TableName_Action
```

**Explicitly name your constraints.** Auto-generated names (`DF__MasterSa__Refre__3B75D760`) differ between environments, which makes scripted drops impossible and Schema Compare noisy.

**Avoid reserved words** as object names. If you need `[Date]` or `[Year]`, bracket them everywhere — or just pick a different name.

**Avoid the `sp_` prefix** on procedures. SQL Server checks `master` first for anything starting with `sp_`, which is a small performance cost and a real source of confusion.

---

## Code review checklist

For any procedure or query going into the repo:

**Correctness**
- [ ] Is the grain preserved? Could this fan out and double-count?
- [ ] `LEFT JOIN` where dropping rows would be silent data loss?
- [ ] NULL handling: `NOT EXISTS` instead of `NOT IN`? Inequalities accounting for NULL?
- [ ] Half-open date ranges (`>= @start AND < @end`)?
- [ ] Idempotent — safe to run twice?

**Performance**
- [ ] Predicates sargable? No functions wrapping indexed columns?
- [ ] Any implicit conversions? (parameter types matching column types)
- [ ] Supporting indexes exist?
- [ ] `OPTION (RECOMPILE)` where the optional-parameter pattern is used?
- [ ] No scalar UDFs in a `SELECT` list?

**Safety**
- [ ] `SET NOCOUNT, XACT_ABORT ON`?
- [ ] `TRY/CATCH` with `ROLLBACK` and `THROW`?
- [ ] Transaction as short as possible — no slow work inside it?
- [ ] App lock if concurrent runs would conflict?
- [ ] Dynamic SQL parameterized with `sp_executesql` and `QUOTENAME`?

**Maintainability**
- [ ] Two-part names everywhere (`mart.MasterSales`, not `MasterSales`)?
- [ ] `CREATE OR ALTER`?
- [ ] No `SELECT *` in a view or in production code?
- [ ] Comment explaining *why* for anything non-obvious?
- [ ] Logging: does a failure leave a trace someone can find?

---

## Order of operations for a new project

1. **`00_schemas/`** — thirty seconds of work
2. **`10_tables/ref.*` + `20_seed/`** — stable, and everything joins to them
3. **`10_tables/raw.*`** — real types, plus the `(LoadDate, SourceSystem)` index
4. **`10_tables/mart.*`** — with the grain unique constraint
5. **`10_tables/dbo.RefreshLog`** — before you need it, not after
6. **Point SSIS at raw. Load one period. Look at the data.**
7. **Then write `mart.RefreshMasterSales`**
8. **`30_views/`** — the front-end contract
9. **Schedule it, reconcile it, monitor it**

> **Writing the procedure last matters more than it sounds.** Once real data is sitting in `raw`, you'll discover the NULLs, the ProductIds missing from your map, the duplicate rows, the dates that arrived as strings. You'll write a procedure that handles what's *actually there* rather than what you assumed would be there.

**For the README:** write down the grain of each master table in plain English. Future you will thank present you, and it's the first question anyone else will ask.

---

[← Back to index](README.md) · [Next: Operations →](08-operations.md)
