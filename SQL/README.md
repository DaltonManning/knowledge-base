# SQL Server Engineering Reference

Working notes for designing, building, and running SQL Server data systems — schema design, ETL/ELT pipelines, refresh automation, performance, and operations.

Written for the person who owns the whole stack: the one who writes the procs, sets the backup policy, and has to explain to somebody why a number on a report is wrong.

---

## How to use this

There are two tracks through the same material.

| Track | File | Use it when |
|---|---|---|
| **Fast** | [`10-cheatsheet.md`](10-cheatsheet.md) | You need the answer now. One-liners, syntax, decision tables. |
| **Deep** | Everything else | You're learning it, or about to make a decision you'll live with. |

Every concept in the cheat sheet links back to its full treatment. Read the deep version once, then live in the cheat sheet.

---

## Table of contents

### [00 — Glossary](00-glossary.md)
Every term you'll use forever, defined plainly and grouped by what it's for. Start here if a word in a Stack Overflow answer stopped you cold.

- Correctness under concurrency · Reliability patterns · Modeling · Performance · Query semantics · Pipelines · Operations · Change management

### [01 — Architecture & Layering](01-architecture.md)
Where data lives and why. Schemas, the raw → master → view pipeline, one database vs. many, synonyms, and how to decide what a master table should look like.

- The three-layer model · Schemas · One DB or several · Master table design · Grain · Synonyms

### [02 — Database Objects](02-objects.md)
The full catalog: tables, constraints, indexes, views, procedures, functions, triggers. What each one is *for*, and the traps in each.

- Storage layer · Logic you call · Logic that fires itself · Change-detection features · The object decision table

### [03 — Refresh & Automation Patterns](03-refresh-patterns.md)
The heart of it. Idempotent refresh procedures, full vs. incremental, triggering strategies, queues, watermarks, and error handling that doesn't lie to you.

- Delete-and-reinsert · App locks · Dirty queues · Watermarks · Change Tracking / CDC / temporal · Logging and observability

### [04 — Performance](04-performance.md)
Why things are slow and how to tell. Sargability, indexes, execution plans, statistics, parameter sniffing, and the set-based mindset.

- Sargability · Index anatomy · Reading plans · Statistics & estimates · Parameter sniffing · RBAR · tempdb

### [05 — Concurrency & Transactions](05-concurrency.md)
ACID, isolation levels, blocking vs. deadlock, snapshot isolation, and the locking behavior that makes your refresh proc block your website.

- Transactions · Isolation levels · Lock types & escalation · Blocking vs deadlock · RCSI · App locks

### [06 — T-SQL Cookbook](06-tsql-cookbook.md)
The language patterns worth internalizing. NULL logic, joins, window functions, `APPLY`, date handling, `MERGE` and why to be careful with it.

- Three-valued logic · Join taxonomy · Window functions · APPLY · Dates & time zones · String handling · Error handling

### [07 — Project Layout & Deployment](07-project-layout.md)
Your SQL lives in git. Folder structure, re-runnable scripts, migrations, state-based vs migration-based deployment, environments, drift.

- Folder structure · Re-runnable patterns · Migrations · SSDT vs Flyway · Drift · Code review checklist

### [08 — Operations](08-operations.md)
Keeping it alive. Recovery models, backups, RPO/RTO, restore testing, maintenance, monitoring, and the failure modes that take small shops down.

- Recovery models · Backup strategy · RPO/RTO · Restore testing · Maintenance · Monitoring · Common disasters

### [09 — Integration & External Data](09-integration.md)
Getting data in from elsewhere. SSIS patterns, linked servers, ODBC sources, historians (IP.21), file imports, and the rules for pulling from production systems you don't own.

- SSIS patterns · Linked servers · Historians & time-series sources · File imports · Idempotent loading

### [10 — Cheat Sheet](10-cheatsheet.md)
The simplified track. Everything above compressed to lookup tables, snippets, and rules of thumb.

### [11 — Reading List](11-reading-list.md)
Books, blogs, and tools actually worth your time, with what each one is for.

---

## The five ideas underneath everything

If you carry nothing else out of these notes:

**1. Grain.** What does one row represent? Say it in a sentence. Undeclared grain is the root cause of most wrong numbers.

**2. Idempotence.** Running it twice equals running it once. This converts "the job failed halfway, now what?" into "run it again." It is the single most valuable property in data engineering.

**3. Sargability.** A predicate an index can seek on. `WHERE d >= @start AND d < @end` is sargable. `WHERE YEAR(d) = @y` is not. This one distinction explains most mystery slowness.

**4. Source of truth.** Every fact lives authoritatively in exactly one place. Everything else is derived and disposable. Ambiguity here causes the "which number is right?" meeting.

**5. RPO / RTO.** How much data can you afford to lose, and how long can you afford to be down? These are business decisions that dictate every technical backup choice. As the owner, you set them — nobody else will.

---

## The governing philosophy

**Raw is truth. Everything downstream is derived and rebuildable.**

If you can always reconstruct your master tables by re-running a procedure against raw data, you can be fearless. Refreshes stop being scary. Bugs become fixable retroactively. New requirements become backfills instead of archaeology.

The corollary: **never hand-edit derived data.** The moment somebody types a corrected value directly into a master table, it is no longer derivable, and the next refresh destroys their work. Manual overrides belong in their own table that the refresh joins in.

**Let the database enforce what it can.** You can forget to call a procedure. You cannot forget a constraint. Any rule expressible as a `CHECK`, `UNIQUE`, `FOREIGN KEY`, or `NOT NULL` should be expressed that way rather than as logic in code — constraints are checked on every path into the table, including the one where somebody is poking at data in SSMS at 11pm.

---

## Conventions used in these notes

Examples assume this schema layout throughout:

```
raw.*    landing zone — SSIS writes here, no transformation, dumb and append-only
ref.*    static relational maps / lookup dimensions
stg.*    optional scratch space for multi-step transforms
mart.*   master tables and the views reports read
dbo.*    default — utility objects, logging, anything cross-cutting
```

Running example: sales extract data landing in `raw`, joined against product and region maps in `ref`, aggregated by year into `mart.MasterSales`, exposed to a website through `mart.vw_*`.

```
SSIS ──▶ raw.SalesExtract ──┐
                            ├──[ mart.RefreshMaster ]──▶ mart.MasterSales ──▶ mart.vw_* ──▶ website
        ref.ProductMap ─────┘
        ref.RegionMap
```

---

*Version notes: content targets SQL Server 2016+ unless stated. Version-specific behavior is flagged inline where it matters — 2019's scalar UDF inlining and 2022's parameter-sensitive plans are the big ones.*
