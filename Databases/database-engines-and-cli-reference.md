# Relational Databases, SQL Dialects, and Command-Line Clients

[← Back to index](README.md)

**A technical reference covering Oracle, Microsoft SQL Server, PostgreSQL, and AspenTech InfoPlus.21**

---

## Table of Contents

1. [Core Architecture: Engine, Language, Client, Driver](#1-core-architecture-engine-language-client-driver)
2. [The SQL Language and Its Dialects](#2-the-sql-language-and-its-dialects)
3. [Database Engine Comparison](#3-database-engine-comparison)
4. [Computer Science Foundations](#4-computer-science-foundations)
5. [Oracle Database In Depth](#5-oracle-database-in-depth)
6. [SQL*Plus Complete Reference](#6-sqlplus-complete-reference)
7. [SQLcl: The Modern Oracle CLI](#7-sqlcl-the-modern-oracle-cli)
8. [SQL Server and sqlcmd](#8-sql-server-and-sqlcmd)
9. [PostgreSQL and psql](#9-postgresql-and-psql)
10. [InfoPlus.21 and Aspen SQLplus](#10-infoplus21-and-aspen-sqlplus)
11. [Side-by-Side CLI Comparison](#11-side-by-side-cli-comparison)
12. [Cross-Dialect SQL Translation Table](#12-cross-dialect-sql-translation-table)
13. [Connectivity Layers: ODBC, JDBC, OLE DB, Native Drivers](#13-connectivity-layers-odbc-jdbc-ole-db-native-drivers)
14. [Automation Capabilities](#14-automation-capabilities)
15. [Practical Walkthrough: First Oracle Session](#15-practical-walkthrough-first-oracle-session)
16. [Safety Rules for Production Systems](#16-safety-rules-for-production-systems)

---

## 1. Core Architecture: Engine, Language, Client, Driver

Every database platform consists of four distinct layers. Confusion between these layers is the most common source of misunderstanding.

| Layer | Definition | Oracle | SQL Server | PostgreSQL | IP.21 |
|---|---|---|---|---|---|
| **Engine** | Server software that stores data, manages memory, enforces transactions, and executes queries | Oracle Database | SQL Server Database Engine | PostgreSQL server (`postgres`) | InfoPlus.21 |
| **Language** | The syntax used to express queries and logic | SQL + PL/SQL | SQL + T-SQL | SQL + PL/pgSQL | Aspen SQLplus |
| **Client** | Program a human uses to send statements to the engine | SQL\*Plus, SQLcl, SQL Developer | sqlcmd, SSMS, Azure Data Studio | psql, pgAdmin | Aspen SQLplus Query Writer |
| **Driver** | Library a program uses to send statements to the engine | OCI, ODBC, JDBC, ODP.NET | ODBC, OLE DB, JDBC, SqlClient | libpq, ODBC, JDBC | AspenTech ODBC, Aspen APIs |

### Request lifecycle

```
You type a query
      │
      ▼
Client (SQL*Plus / sqlcmd / psql / Excel)
      │   uses a driver / network protocol
      ▼
Network listener (Oracle: TNS listener :1521, SQL Server :1433, Postgres :5432)
      │
      ▼
Engine: parse → optimize → execute → return rows
      │
      ▼
Client formats and displays rows
```

**Key point:** SQL\*Plus is not a language. It is a client program. The language typed into it is Oracle SQL and PL/SQL. SQL\*Plus also has its own small set of *client commands* (such as `SET LINESIZE`) that are never sent to the server.

---

## 2. The SQL Language and Its Dialects

### 2.1 Origin

- **Relational model:** Proposed by Edgar F. Codd at IBM in 1970. Data is represented as relations (tables) of tuples (rows) with attributes (columns).
- **SQL:** Developed at IBM in the 1970s (originally named SEQUEL). Standardized by ANSI in 1986 and ISO in 1987. Revised many times since (SQL-92, SQL:1999, SQL:2003, SQL:2011, SQL:2016, SQL:2023).
- No vendor implements the full standard, and every vendor adds proprietary extensions.

### 2.2 Inheritance model

```
ISO/ANSI SQL Standard (base definition)
 ├── Oracle SQL ─────── extended by PL/SQL  (Procedural Language/SQL)
 ├── Microsoft SQL ──── extended by T-SQL   (Transact-SQL, shared lineage with Sybase)
 ├── PostgreSQL SQL ─── extended by PL/pgSQL (modelled closely on PL/SQL)
 ├── MySQL SQL ──────── stored-procedure syntax based on SQL/PSM
 └── Aspen SQLplus ──── SQL-like language adapted to a historian data model
```

Each dialect **inherits** the core grammar, **adds** functions, data types, and procedural constructs, and **overrides** certain behaviors (row limiting, string concatenation, date handling, NULL semantics).

### 2.3 SQL sub-languages

| Category | Name | Statements | Effect |
|---|---|---|---|
| DQL | Data Query Language | `SELECT` | Reads data. Safe. |
| DML | Data Manipulation Language | `INSERT`, `UPDATE`, `DELETE`, `MERGE` | Changes rows. Transactional. |
| DDL | Data Definition Language | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME` | Changes structure. In Oracle, commits implicitly. |
| DCL | Data Control Language | `GRANT`, `REVOKE` | Changes permissions. |
| TCL | Transaction Control Language | `COMMIT`, `ROLLBACK`, `SAVEPOINT` | Ends or partially undoes transactions. |

### 2.4 Declarative vs procedural

Plain SQL is **declarative**: the statement describes the desired result, and the engine's optimizer decides how to obtain it.

Procedural extensions add **imperative** constructs: variables, conditionals, loops, exception handling, and reusable units stored in the database (procedures, functions, triggers, packages).

**PL/SQL block (Oracle):**
```sql
DECLARE
  v_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_count FROM sample;
  IF v_count > 1000 THEN
    DBMS_OUTPUT.PUT_LINE('Count: ' || v_count);
  END IF;
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

**T-SQL batch (SQL Server):**
```sql
DECLARE @count INT;
SELECT @count = COUNT(*) FROM sample;
IF @count > 1000
    PRINT 'Count: ' + CAST(@count AS VARCHAR(20));
GO
```

**PL/pgSQL block (PostgreSQL):**
```sql
DO $$
DECLARE
  v_count INTEGER;
BEGIN
  SELECT COUNT(*) INTO v_count FROM sample;
  IF v_count > 1000 THEN
    RAISE NOTICE 'Count: %', v_count;
  END IF;
END $$;
```

---

## 3. Database Engine Comparison

| Characteristic | Oracle | SQL Server | PostgreSQL | InfoPlus.21 |
|---|---|---|---|---|
| **Vendor / governance** | Oracle Corporation | Microsoft | Open source (PostgreSQL Global Development Group) | AspenTech |
| **Engine type** | Relational (RDBMS), multi-model | Relational (RDBMS) | Object-relational (ORDBMS) | Real-time process historian |
| **Optimized for** | High concurrency, large transactional and mixed workloads, clustering (RAC) | Transactional and analytic workloads tightly integrated with Windows and Microsoft tooling | Standards compliance, extensibility, correctness | High-frequency time-stamped sensor data |
| **Data shape** | Tables, rows, columns; also JSON, XML, spatial, graph | Tables, rows, columns; also JSON, XML, spatial, graph | Tables, rows, columns; also JSONB, arrays, ranges, custom types | Records (tags) with fields, plus compressed time-series history |
| **Runs on** | Linux, Unix (Solaris, AIX), Windows | Windows, Linux, containers | Linux, Unix, macOS, Windows | Windows |
| **Procedural extension** | PL/SQL | T-SQL | PL/pgSQL (plus PL/Python, PL/Perl, etc.) | Aspen SQLplus procedural syntax |
| **Primary CLI** | SQL\*Plus, SQLcl | sqlcmd | psql | None comparable (GUI Query Writer) |
| **Default port** | 1521 (TNS listener) | 1433 | 5432 | Varies by Aspen service |
| **Namespace hierarchy** | Database (CDB) → Pluggable DB (PDB) → Schema (= user) → Object | Instance → Database → Schema → Object | Cluster → Database → Schema → Object | Database → Record (tag) → Field / Repeat area |
| **Autocommit default in CLI** | Off | On | On | N/A |
| **Unquoted identifier case** | Folded to UPPERCASE | Case-insensitive by default collation | Folded to lowercase | Case-insensitive |
| **Empty string** | Treated as NULL | Distinct from NULL | Distinct from NULL | N/A |
| **Concurrency model** | MVCC using undo segments | Locking by default; optional row versioning (RCSI / snapshot) | MVCC using row versions (requires VACUUM) | Real-time in-memory database + history files |
| **Durability log** | Redo log | Transaction log (.ldf) | Write-Ahead Log (WAL) | History file sets |

### Where PostgreSQL fits

PostgreSQL is the third member of the "big relational" group alongside Oracle and SQL Server. Technically:

- Its SQL dialect is the closest of the three to the ISO standard.
- PL/pgSQL was deliberately modelled on PL/SQL, so Oracle skills transfer well.
- It is extensible at the engine level: custom data types, index types, operators, and extensions (e.g., `postgis`, `timescaledb`, `pg_cron`).
- **Foreign Data Wrappers** allow PostgreSQL to query other databases as if their tables were local (`oracle_fdw` for Oracle, `tds_fdw` for SQL Server). This makes PostgreSQL a strong integration and staging target.

### Where a historian differs

A relational database is optimized for records that are inserted, updated, and joined. A historian is optimized for append-only readings (tag, timestamp, value, quality) at very high rates, and applies compression (such as deadband or swinging-door algorithms) to store only significant changes. Queries center on time ranges, interpolation, and aggregation rather than joins.

---

## 4. Computer Science Foundations

### 4.1 ACID transactions

| Property | Meaning |
|---|---|
| **Atomicity** | A transaction completes entirely or not at all. |
| **Consistency** | A transaction moves the database from one valid state to another (constraints hold). |
| **Isolation** | Concurrent transactions do not see each other's uncommitted work (degree depends on isolation level). |
| **Durability** | Once committed, data survives crashes (guaranteed by the redo/transaction log/WAL). |

### 4.2 Concurrency control

- **MVCC (Multi-Version Concurrency Control):** Readers see a consistent snapshot; writers create new versions. Readers do not block writers and writers do not block readers. Used by Oracle (via undo) and PostgreSQL (via row versions).
- **Lock-based:** Readers take shared locks and may wait on writers. SQL Server's default `READ COMMITTED` uses locking; enabling `READ_COMMITTED_SNAPSHOT` switches it to row versioning.

**Practical consequence for reporting:** A long `SELECT` against Oracle does not block the LIMS application's writes. On SQL Server without RCSI, it can.

### 4.3 Isolation levels (ISO standard)

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible |
| Serializable | Prevented | Prevented | Prevented |

Oracle supports Read Committed (default) and Serializable. Every query in Oracle is statement-level read-consistent.

### 4.4 Query processing

1. **Parse:** Check syntax and resolve object names and privileges.
2. **Optimize:** A cost-based optimizer uses table statistics to choose join order, join method (nested loop, hash, merge), and access paths (full scan vs index).
3. **Execute:** Run the plan and fetch rows.

Oracle caches parsed statements in the **shared pool**. Using bind variables (`:id`) instead of literal values lets the same plan be reused.

### 4.5 Indexes

- **B-tree:** Default in all three engines. Efficient for equality and range lookups.
- **Bitmap (Oracle):** Efficient for low-cardinality columns in read-heavy data.
- **Columnstore (SQL Server), In-Memory column store (Oracle):** Column-oriented storage for analytic scans.
- **GIN / GiST / BRIN (PostgreSQL):** Specialized indexes for JSON, full-text, spatial, and very large ordered tables.

### 4.6 Normalization

Relational schemas (including LIMS schemas) are normalized: data is split into many related tables linked by keys to avoid duplication. As a result, a single business concept (such as "a sample with its test results") usually requires joining several tables. **Views** are used to present joined, reader-friendly shapes without duplicating data.

---

## 5. Oracle Database In Depth

### 5.1 Instance vs database

- **Instance:** Memory structures (SGA, PGA) and background processes running on a server.
- **Database:** Physical files on disk (data files, control files, redo logs).
- One instance mounts and opens one database (or, with RAC, multiple instances open one database).

### 5.2 Multitenant architecture (12c and later)

```
Container Database (CDB)
 ├── CDB$ROOT        ← Oracle's internal metadata; not for application data
 ├── PDB$SEED        ← Template used to create new PDBs
 └── Pluggable DBs   ← Where application schemas (e.g., SampleManager) live
```

Connecting `/ as sysdba` on the server lands in `CDB$ROOT`. Application tables will not be visible until switching into the correct PDB.

```sql
SHOW CON_NAME
SELECT name, open_mode FROM v$pdbs;
ALTER SESSION SET CONTAINER = MYPDB;
```

Databases created before 12c, or created as non-CDB, have no PDBs. In that case `SHOW CON_NAME` returns an error or `v$pdbs` is empty, and the data is directly accessible.

### 5.3 Schemas and users

In Oracle, **a schema and a user are the same thing**. A user named `SMOWNER` owns a schema named `SMOWNER` containing its tables, views, and code. Objects are referenced as `OWNER.OBJECT_NAME`.

Other users can access those objects only if granted privileges.

### 5.4 Storage

- **Tablespace:** A logical storage container made of one or more data files.
- **Segment:** Storage allocated to one object (table, index).
- Common tablespaces: `SYSTEM`, `SYSAUX`, `UNDOTBS1`, `TEMP`, `USERS`, plus application-specific ones.

### 5.5 Administrative privileges

| Connection | Meaning |
|---|---|
| `AS SYSDBA` | Full control, including startup, shutdown, and recovery. Connects as `SYS`. |
| `AS SYSOPER` | Startup, shutdown, and backup without viewing application data. |
| Normal user | Limited to granted privileges. |

`sqlplus / as sysdba` uses **operating system authentication**: if the OS account belongs to the `ORA_DBA` group (Windows) or `dba` group (Linux), no password is required.

### 5.6 Data dictionary

Oracle describes itself through read-only views. Three prefixes exist:

| Prefix | Scope |
|---|---|
| `USER_` | Objects owned by the current user |
| `ALL_` | Objects the current user can access |
| `DBA_` | All objects in the database (requires DBA privileges) |

| View | Contents |
|---|---|
| `DBA_USERS` | All users/schemas |
| `DBA_TABLES` | All tables |
| `DBA_TAB_COLUMNS` | All columns, data types, lengths |
| `DBA_VIEWS` | All views and their SQL text |
| `DBA_CONSTRAINTS` | Primary keys (`P`), foreign keys (`R`), unique (`U`), check (`C`) |
| `DBA_CONS_COLUMNS` | Columns belonging to each constraint |
| `DBA_INDEXES`, `DBA_IND_COLUMNS` | Indexes and their columns |
| `DBA_OBJECTS` | Every object with type, status, and timestamps |
| `DBA_SOURCE` | PL/SQL source code |
| `DBA_SYNONYMS` | Aliases pointing to other objects |
| `DBA_TAB_PRIVS`, `DBA_SYS_PRIVS`, `DBA_ROLE_PRIVS` | Granted privileges |
| `DBA_SEGMENTS` | Space consumed per object |
| `DICTIONARY` | List of all dictionary views with descriptions |

**Dynamic performance views (`V$`)** reflect live instance state:

| View | Contents |
|---|---|
| `V$VERSION` | Oracle version |
| `V$INSTANCE` | Instance name, host, status, startup time |
| `V$DATABASE` | Database name, log mode, CDB status |
| `V$SESSION` | Connected sessions |
| `V$SQL` | Cached SQL statements and execution statistics |
| `V$PARAMETER` | Initialization parameters |

### 5.7 Oracle-specific SQL behavior

| Behavior | Detail |
|---|---|
| `DUAL` | One-row dummy table used to evaluate expressions: `SELECT SYSDATE FROM dual;` |
| Row limiting | `WHERE ROWNUM <= 10` (all versions); `FETCH FIRST 10 ROWS ONLY` (12c+) |
| `ROWNUM` with `ORDER BY` | `ROWNUM` is assigned before sorting. Wrap in a subquery to get "top N after sort." |
| String concatenation | `'a' \|\| 'b'` |
| Null handling | `NVL(x, default)`, `NVL2`, `COALESCE`, `NULLIF` |
| Empty string | `''` is NULL. `WHERE col = ''` never matches; use `IS NULL`. |
| `DATE` type | Always includes time to the second. `TIMESTAMP` adds fractional seconds. |
| Date conversion | `TO_DATE('2026-09-29','YYYY-MM-DD')`, `TO_CHAR(dt,'YYYY-MM-DD HH24:MI:SS')` |
| Current time | `SYSDATE` (server time), `SYSTIMESTAMP`, `CURRENT_DATE` (session time zone) |
| Conditional | `CASE WHEN ... THEN ... END`, `DECODE(expr, v1, r1, v2, r2, default)` |
| Case folding | `select * from sample` resolves to `SAMPLE`. Quoted `"Sample"` is case-sensitive. |
| String literals | Single quotes only. Embedded quote: `'O''Brien'` or `q'[O'Brien]'` |
| Sequences | `CREATE SEQUENCE s;` then `s.NEXTVAL`, `s.CURRVAL` |
| Identity columns | `GENERATED AS IDENTITY` (12c+) |
| Hierarchical queries | `CONNECT BY PRIOR child = parent START WITH ...` |
| Transactions | Begin implicitly with the first DML. Must `COMMIT` or `ROLLBACK`. |
| Implicit commit | Every DDL statement commits any pending transaction before and after it. |

### 5.8 Execution plans

```sql
EXPLAIN PLAN FOR
  SELECT * FROM smowner.sample WHERE status = 'A';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### 5.9 Creating a read-only reporting account

```sql
-- Run as SYSDBA inside the correct PDB
CREATE USER lims_report IDENTIFIED BY "StrongPassword#1";
GRANT CREATE SESSION TO lims_report;

-- Per-table read access (READ, 12c+, does not allow SELECT ... FOR UPDATE row locking)
GRANT READ ON smowner.sample TO lims_report;

-- Generate grant statements for every table in a schema
SELECT 'GRANT READ ON ' || owner || '.' || table_name || ' TO lims_report;'
FROM dba_tables
WHERE owner = 'SMOWNER';
```

On Oracle 23ai and later, schema-level grants are available:
```sql
GRANT SELECT ANY TABLE ON SCHEMA smowner TO lims_report;
```

Private synonyms or `ALTER SESSION SET CURRENT_SCHEMA = smowner;` allow the report user to omit the `SMOWNER.` prefix.

### 5.10 Connection naming

| Term | Meaning |
|---|---|
| **SID** | Name of an instance (older style) |
| **Service name** | Logical name clients connect to; a PDB exposes its own service name |
| **Easy Connect** | `host:port/service_name`, e.g., `dbserver01:1521/SMPDB` |
| **TNS alias** | Short name defined in `tnsnames.ora`, e.g., `SMPROD` |
| **`TNS_ADMIN`** | Environment variable pointing to the folder containing `tnsnames.ora` and `sqlnet.ora` |

**Example `tnsnames.ora` entry:**
```
SMPROD =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = dbserver01)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = SMPDB)
    )
  )
```

Listing available services on the server:
```
lsnrctl status
```

---

## 6. SQL*Plus Complete Reference

SQL\*Plus is Oracle's original command-line client, installed with every Oracle Database and Oracle Client (including Instant Client with the SQL\*Plus package).

### 6.1 Launching

```
sqlplus / as sysdba                              # OS authentication on the server
sqlplus username                                 # prompts for password, uses default/local DB
sqlplus username@SMPROD                          # TNS alias
sqlplus username@//dbserver01:1521/SMPDB         # Easy Connect
sqlplus username/password@SMPROD                 # inline password (visible in process list; avoid)
sqlplus /nolog                                   # start without connecting
sqlplus -S username@SMPROD @script.sql           # silent mode, run a script
sqlplus -L username@SMPROD                       # attempt login once only (no re-prompt)
sqlplus -V                                       # print version and exit
```

Inside SQL\*Plus:
```
CONNECT username@SMPROD
CONNECT / AS SYSDBA
DISCONNECT
EXIT              -- commits pending work by default, then quits
QUIT              -- same as EXIT
EXIT ROLLBACK     -- rolls back pending work, then quits
```

### 6.2 Two kinds of input

| Input | Terminator | Sent to server? | Examples |
|---|---|---|---|
| SQL statements | `;` or `/` on a new line | Yes | `SELECT`, `INSERT`, `CREATE` |
| PL/SQL blocks | `/` on a new line (the `;` inside blocks ends statements, not the block) | Yes | `BEGIN ... END;` then `/` |
| SQL\*Plus commands | Enter key (no terminator required) | No, handled by the client | `SET`, `COLUMN`, `DESC`, `SPOOL`, `SHOW` |

A statement without a terminator stays in the **buffer**. Pressing Enter on a blank line stops entry without running.

### 6.3 Buffer commands

The last SQL statement or PL/SQL block is stored in the SQL buffer.

| Command | Action |
|---|---|
| `/` | Execute the buffer |
| `RUN` or `R` | Display and execute the buffer |
| `LIST` or `L` | Display the buffer |
| `L n` | Display line *n* and make it current |
| `CHANGE /old/new/` or `C /old/new/` | Replace text on the current line |
| `APPEND text` or `A text` | Append text to the current line |
| `INPUT` or `I` | Add lines after the current line |
| `DEL` | Delete the current line |
| `CLEAR BUFFER` | Empty the buffer |
| `EDIT` or `ED` | Open the buffer in an external editor (Notepad on Windows, `$EDITOR`/vi on Linux) |
| `SAVE file.sql` | Save the buffer to a file |
| `GET file.sql` | Load a file into the buffer without running |

Set the editor:
```
DEFINE _EDITOR = notepad
DEFINE _EDITOR = vi
```

### 6.4 Keyboard behavior

| Platform | Behavior |
|---|---|
| **Windows console** | Up/Down arrows recall previous lines; F7 opens a history list; F8 searches history by prefix. Provided by the Windows console, not SQL\*Plus. |
| **Linux/Unix** | No arrow-key history or tab completion. Wrap with `rlwrap sqlplus ...` to add readline history and editing. |
| **Ctrl+C** | Cancels the running statement. |
| **Tab completion** | Not supported in SQL\*Plus (use SQLcl). |

### 6.5 Display formatting

```
SET LINESIZE 250          -- characters per line
SET PAGESIZE 100          -- rows per page before headers repeat; 0 = no headers
SET PAGESIZE 50000        -- effectively one header block
SET WRAP OFF              -- truncate long lines instead of wrapping
SET TRIMOUT ON            -- remove trailing spaces on screen
SET TRIMSPOOL ON          -- remove trailing spaces in spooled files
SET FEEDBACK OFF          -- hide "n rows selected"
SET HEADING OFF           -- hide column headers
SET NULL '(null)'         -- display text for NULL values
SET NUMWIDTH 15           -- default width for numbers
SET LONG 100000           -- characters shown for LONG/CLOB columns
SET TAB OFF               -- use spaces instead of tab characters in output
SET UNDERLINE =           -- character under headers
SET COLSEP '|'            -- column separator
```

**Column formatting:**
```
COLUMN table_name FORMAT A30          -- text column 30 characters wide
COLUMN amount FORMAT 999,999.99       -- numeric mask
COLUMN description FORMAT A40 WORD_WRAPPED
COLUMN owner HEADING 'Schema'         -- rename header
COLUMN internal_id NOPRINT            -- hide a column
COLUMN table_name CLEAR               -- remove formatting for one column
CLEAR COLUMNS                         -- remove all column formatting
```

Short forms: `COL` for `COLUMN`, `FOR` for `FORMAT`, `HEA` for `HEADING`.

**Report breaks and totals:**
```
BREAK ON owner SKIP 1
COMPUTE SUM OF num_rows ON owner
```

### 6.6 Timing and diagnostics

```
SET TIMING ON              -- show elapsed time per statement
SET AUTOTRACE ON           -- show results, execution plan, and statistics
SET AUTOTRACE TRACEONLY    -- plan and statistics only (rows fetched, not displayed)
SET AUTOTRACE OFF
SET SERVEROUTPUT ON SIZE UNLIMITED   -- display DBMS_OUTPUT from PL/SQL
SHOW ERRORS                -- compilation errors for the last PL/SQL object
SHOW ERRORS PROCEDURE smowner.my_proc
```

### 6.7 Information commands

```
DESCRIBE smowner.sample        -- columns, NULL constraint, data types (short: DESC)
SHOW USER                      -- current user
SHOW CON_NAME                  -- current container (CDB/PDB)
SHOW PARAMETER db_name         -- initialization parameters matching a pattern
SHOW ALL                       -- all SQL*Plus settings
SHOW SGA                       -- memory areas (DBA)
SHOW RELEASE                   -- SQL*Plus release number
HELP INDEX                     -- list help topics (if help installed)
```

### 6.8 Scripts

```
@script.sql                   -- run a script (path relative to working directory)
@C:\scripts\export.sql
@@child.sql                   -- run a script relative to the calling script's folder
START script.sql              -- same as @
@script.sql arg1 arg2         -- arguments available as &1, &2
```

**Example script `list_tables.sql`:**
```sql
SET PAGESIZE 50000 LINESIZE 200 FEEDBACK OFF TRIMSPOOL ON
COLUMN table_name FORMAT A40

SELECT table_name, num_rows
FROM dba_tables
WHERE owner = UPPER('&1')
ORDER BY table_name;

EXIT
```
Run: `sqlplus -S / as sysdba @list_tables.sql SMOWNER`

### 6.9 Substitution and bind variables

**Substitution variables** are replaced by SQL\*Plus before the text is sent:
```
DEFINE owner = SMOWNER
SELECT COUNT(*) FROM dba_tables WHERE owner = '&owner';
SELECT COUNT(*) FROM dba_tables WHERE owner = '&&owner';   -- && prompts once, then remembers
UNDEFINE owner
ACCEPT owner PROMPT 'Schema name: '
SET VERIFY OFF        -- hide the old/new substitution lines
SET DEFINE OFF        -- treat & as a literal character (e.g., 'R&D')
```

**Bind variables** are sent to the server as parameters, enabling plan reuse:
```
VARIABLE v_status VARCHAR2(10)
EXEC :v_status := 'A'
SELECT COUNT(*) FROM smowner.sample WHERE status = :v_status;
PRINT v_status
```

### 6.10 Spooling output to files

```
SPOOL C:\temp\output.txt
SELECT ...;
SPOOL OFF

SPOOL output.txt APPEND        -- append instead of overwrite
```

**CSV export (SQL\*Plus 12.2 and later):**
```
SET MARKUP CSV ON QUOTE ON
SET FEEDBACK OFF
SPOOL C:\temp\samples.csv
SELECT * FROM smowner.sample WHERE ROWNUM <= 1000;
SPOOL OFF
SET MARKUP CSV OFF
```

**CSV export (older versions):**
```
SET COLSEP ',' PAGESIZE 0 TRIMSPOOL ON FEEDBACK OFF HEADSEP OFF LINESIZE 32767
SPOOL C:\temp\samples.csv
SELECT id || ',' || status || ',' || TO_CHAR(login_date,'YYYY-MM-DD') FROM smowner.sample;
SPOOL OFF
```

**HTML export:**
```
SET MARKUP HTML ON SPOOL ON
SPOOL report.html
SELECT ...;
SPOOL OFF
SET MARKUP HTML OFF
```

### 6.11 Operating system commands

```
HOST dir                 -- run an OS command (Windows)
HOST ls -l               -- run an OS command (Linux)
!ls                      -- Linux shorthand
$dir                     -- Windows shorthand
HOST                     -- open a shell; type exit to return
```

### 6.12 Error handling for automation

```
WHENEVER SQLERROR EXIT FAILURE ROLLBACK     -- stop script on any SQL error
WHENEVER OSERROR EXIT FAILURE
WHENEVER SQLERROR CONTINUE                  -- restore default
```

The exit code of `sqlplus` can then be checked by a batch file or shell script.

### 6.13 Prompt customization

```
SET SQLPROMPT "_USER'@'_CONNECT_IDENTIFIER> "
```
Place this and other preferred settings in `login.sql` (in the working directory or the `ORACLE_PATH`/`SQLPATH` folder) or in `glogin.sql` (`$ORACLE_HOME/sqlplus/admin/`) to apply automatically at every connection.

**Example `login.sql`:**
```
SET LINESIZE 250 PAGESIZE 100 TRIMSPOOL ON TAB OFF
SET SERVEROUTPUT ON SIZE UNLIMITED
SET SQLPROMPT "_USER'@'_CONNECT_IDENTIFIER> "
DEFINE _EDITOR = notepad
```

---

## 7. SQLcl: The Modern Oracle CLI

SQLcl (`sql`) is Oracle's newer command-line client. It is a free Java-based download, accepts nearly all SQL\*Plus commands and scripts, and adds modern features.

| Feature | Command / Key |
|---|---|
| Tab completion | Tab on table and column names |
| Command history | Up/Down arrows; `HISTORY` lists; `HISTORY 5` recalls item 5 |
| Multi-line editing | Arrow keys move within the statement buffer |
| Object information | `INFO smowner.sample` (columns, comments, indexes, keys) |
| Extended info | `INFO+ smowner.sample` (adds column statistics) |
| Generate DDL | `DDL smowner.sample` |
| Output formats | `SET SQLFORMAT csv`, `json`, `html`, `xml`, `insert`, `loader`, `ansiconsole`, `default` |
| Aliases | `ALIAS tabs=SELECT table_name FROM user_tables;` then `tabs` |
| Load CSV into table | `LOAD tablename file.csv` |
| Unload table to file | `UNLOAD tablename` |
| Change directory | `CD C:\scripts` |
| Repeat a statement | `REPEAT 10 2` (run the buffer 10 times, 2 seconds apart) |
| Scripting | JavaScript via `SCRIPT` command |

**Launch:**
```
sql username@//dbserver01:1521/SMPDB
sql / as sysdba
```

SQLcl is generally more pleasant for interactive exploration; SQL\*Plus is universally present and preferred in legacy automation scripts.

---

## 8. SQL Server and sqlcmd

### 8.1 Architecture

- **Instance:** A running SQL Server service. Default instance is addressed by host name; named instances as `HOST\INSTANCE`.
- **Database:** Many databases per instance, each with its own files and transaction log.
- **Schema:** Namespace inside a database (default `dbo`). Unlike Oracle, schema and user are separate.
- **Object naming:** `server.database.schema.object` (four-part), usually `database.schema.object` or `schema.object`.

### 8.2 Authentication

- **Windows Authentication (Integrated):** Uses the logged-in Windows/Active Directory identity. `-E` in sqlcmd.
- **SQL Server Authentication:** Username and password stored in SQL Server. `-U` and `-P`.
- **Microsoft Entra ID (Azure AD):** For cloud and hybrid deployments.

### 8.3 Launching sqlcmd

```
sqlcmd -S dbserver01 -E                              # Windows auth, default instance
sqlcmd -S dbserver01\SQLEXPRESS -E                   # named instance
sqlcmd -S dbserver01,1433 -U report -P secret        # SQL auth with port
sqlcmd -S dbserver01 -E -d LabDB                     # choose database
sqlcmd -S dbserver01 -E -Q "SELECT @@VERSION"        # run one query and exit
sqlcmd -S dbserver01 -E -q "SELECT @@VERSION"        # run one query and stay interactive
sqlcmd -S dbserver01 -E -i script.sql -o out.txt     # run a file, write output
sqlcmd -S dbserver01 -E -Q "SELECT * FROM t" -s "," -W -o out.csv   # CSV-style export
sqlcmd -S dbserver01 -E -h -1                        # no column headers
sqlcmd -S dbserver01 -E -b                           # exit with error code on failure
sqlcmd -S dbserver01 -E -v owner="dbo"               # define a scripting variable
sqlcmd -?                                            # list switches
```

Two implementations exist: the original ODBC-based `sqlcmd` shipped with SQL Server tools, and the newer Go-based `go-sqlcmd`. Switches are largely compatible.

### 8.4 Batches and GO

Statements are collected until `GO` is entered on its own line. `GO` is a client command, not T-SQL; it tells sqlcmd to send the batch.

```
1> SELECT name FROM sys.databases;
2> GO
```

`GO 5` runs the preceding batch five times.

### 8.5 sqlcmd commands

| Command | Action |
|---|---|
| `GO` | Execute the batch |
| `RESET` | Clear the batch buffer |
| `:ED` | Edit the batch in an external editor |
| `:!! command` | Run an OS command |
| `:r file.sql` | Include and run a script file |
| `:out file.txt` | Redirect output to a file (`:out stdout` to restore) |
| `:error file.txt` | Redirect error output |
| `:setvar name value` | Define a variable, referenced as `$(name)` |
| `:listvar` | Show variables |
| `:connect server` | Connect to another server |
| `:on error exit` | Stop on error (automation) |
| `:XML ON` | XML output mode |
| `:help` | List commands |
| `EXIT` / `QUIT` | Leave sqlcmd |
| `EXIT(SELECT 1)` | Exit and return a value as the exit code |

Keyboard: Up/Down arrow and F7 history are provided by the Windows console. No tab completion.

### 8.6 Exploring a SQL Server instance

```sql
SELECT @@VERSION;
SELECT @@SERVERNAME;
SELECT name FROM sys.databases;
USE LabDB;
SELECT DB_NAME();                                   -- current database
SELECT name FROM sys.schemas;
SELECT s.name AS schema_name, t.name AS table_name
FROM sys.tables t JOIN sys.schemas s ON t.schema_id = s.schema_id
ORDER BY 1, 2;
SELECT * FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'Sample';
EXEC sp_help 'dbo.Sample';                          -- equivalent of DESC
EXEC sp_columns 'Sample';
SELECT TOP 10 * FROM dbo.Sample;
```

### 8.7 T-SQL specifics

| Behavior | Detail |
|---|---|
| Row limiting | `SELECT TOP 10 ...`, or `ORDER BY ... OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY` |
| Concatenation | `'a' + 'b'` or `CONCAT('a','b')` |
| Null handling | `ISNULL(x, default)`, `COALESCE` |
| Current time | `GETDATE()`, `SYSDATETIME()` |
| Variables | `DECLARE @x INT = 5;` |
| Autocommit | On by default. Use `BEGIN TRAN` ... `COMMIT` / `ROLLBACK` for explicit transactions. |
| Row count | `@@ROWCOUNT` |
| Quiet counts | `SET NOCOUNT ON;` |
| Identifier quoting | `[Column Name]` or `"Column Name"` |

---

## 9. PostgreSQL and psql

### 9.1 Architecture

- **Cluster:** One running server instance managing many databases.
- **Database:** Isolated; a connection targets exactly one database (cross-database queries require extensions such as `dblink` or `postgres_fdw`).
- **Schema:** Namespace inside a database (default `public`). Schemas and roles are separate.
- **`search_path`:** Ordered list of schemas searched for unqualified names.
- **Roles:** Users and groups are both "roles."

### 9.2 Launching psql

```
psql -h dbserver01 -p 5432 -U report -d labdb
psql "postgresql://report@dbserver01:5432/labdb"
psql -d labdb -c "SELECT version();"            # run one statement and exit
psql -d labdb -f script.sql                     # run a file
psql -d labdb -A -t -F "," -c "SELECT ..." -o out.csv   # unaligned, tuples only, comma separated
psql -d labdb -v ON_ERROR_STOP=1 -f script.sql  # stop on first error
```

Passwords can be supplied through `~/.pgpass` (`%APPDATA%\postgresql\pgpass.conf` on Windows) or the `PGPASSWORD` environment variable. Environment variables `PGHOST`, `PGPORT`, `PGUSER`, `PGDATABASE` set defaults.

### 9.3 psql meta-commands

Meta-commands begin with a backslash and are handled by psql.

| Command | Action |
|---|---|
| `\?` | Help on meta-commands |
| `\h SELECT` | Help on SQL syntax |
| `\l` | List databases |
| `\c dbname` | Connect to another database |
| `\conninfo` | Current connection details |
| `\dn` | List schemas |
| `\dt` | List tables (in search path) |
| `\dt schema.*` | List tables in a schema |
| `\d tablename` | Describe table (columns, indexes, constraints) |
| `\d+ tablename` | Describe with storage and comments |
| `\dv`, `\di`, `\ds`, `\df` | List views, indexes, sequences, functions |
| `\du` | List roles |
| `\dp` | List privileges |
| `\x` | Toggle expanded (vertical) display |
| `\x auto` | Expanded display only when rows are too wide |
| `\timing` | Toggle execution timing |
| `\e` | Edit the query buffer in an editor |
| `\i file.sql` | Run a script |
| `\o file.txt` | Send output to a file (`\o` alone to stop) |
| `\copy (SELECT ...) TO 'out.csv' CSV HEADER` | Client-side CSV export |
| `\pset format csv` | CSV output format |
| `\set VAR value` | Define a variable, used as `:VAR` or `:'VAR'` |
| `\watch 5` | Re-run the last query every 5 seconds |
| `\! command` | Run an OS command |
| `\q` | Quit |

Keyboard: full readline support on Linux/macOS (arrow history, Ctrl+R reverse search, tab completion of table and column names). Tab completion is also available in the Windows build.

Settings for every session go in `~/.psqlrc` (`%APPDATA%\postgresql\psqlrc.conf` on Windows).

### 9.4 PostgreSQL specifics

| Behavior | Detail |
|---|---|
| Row limiting | `LIMIT 10 OFFSET 20`, or standard `FETCH FIRST 10 ROWS ONLY` |
| Concatenation | `'a' \|\| 'b'` |
| Case-insensitive match | `ILIKE` |
| Casts | `'2026-09-29'::date` or `CAST(... AS date)` |
| Current time | `now()`, `CURRENT_TIMESTAMP` |
| Autocommit | On by default. `BEGIN;` ... `COMMIT;` for explicit transactions. |
| Upsert | `INSERT ... ON CONFLICT (key) DO UPDATE ...` |
| Returning rows from DML | `INSERT ... RETURNING id` |
| Catalogs | `information_schema.*` (standard) and `pg_catalog.*` (native) |

---

## 10. InfoPlus.21 and Aspen SQLplus

> **Name collision:** *Aspen SQLplus* is an AspenTech product and language. It is unrelated to Oracle's *SQL\*Plus* client. The similarity is only in the name.

### 10.1 Data model

- **Records:** Each tag or configuration object is a record, defined by a **definition record** (e.g., `IP_AnalogDef`, `IP_DiscreteDef`) that determines its fields.
- **Fields:** Attributes of a record, such as `IP_INPUT_VALUE`, `IP_INPUT_TIME`, `IP_DESCRIPTION`, `IP_ENG_UNITS`.
- **Repeat areas:** Arrays within a record, including the history repeat area that holds time-stamped values.
- **History:** Values are archived to history file sets on disk, subject to compression settings.

### 10.2 Language characteristics

- Standard-looking `SELECT ... FROM ... WHERE` where definition records act as tables and fields act as columns.
- Built-in handling of time ranges, time periods, and interpolation for history retrieval.
- Procedural statements (variables, loops, conditionals, local procedures) in the same script.
- Ability to write to records and fields, and to schedule query records to run automatically.

**Illustrative queries (field and table names depend on site configuration):**
```sql
-- Current values of analog tags
SELECT name, ip_input_value, ip_input_time, ip_eng_units
FROM ip_analogdef
WHERE name LIKE 'TIC101%';

-- Historical values for one tag over a time range
SELECT ts, value
FROM history
WHERE name = 'TIC101.PV'
  AND ts BETWEEN '28-SEP-26 00:00' AND '29-SEP-26 00:00';
```

The `history` pseudo-table supports additional columns controlling retrieval (such as request type and period) for actual versus interpolated or aggregated values. The exact options are documented in AspenTech's Aspen SQLplus reference for the installed version.

### 10.3 Tools

- **Aspen SQLplus Query Writer:** The primary interactive client (Windows GUI).
- **AspenTech SQLplus ODBC driver:** Allows Excel and other ODBC applications to query IP.21 using SQLplus syntax.
- **Aspen Process Explorer / aspenONE Process Explorer:** Trend visualization.
- **Aspen Excel add-in:** Tag data retrieval directly into worksheets.

---

## 11. Side-by-Side CLI Comparison

| Task | SQL\*Plus (Oracle) | sqlcmd (SQL Server) | psql (PostgreSQL) |
|---|---|---|---|
| Connect | `sqlplus user@//host:1521/svc` | `sqlcmd -S host -U user` | `psql -h host -U user -d db` |
| Connect with OS/Windows auth | `sqlplus / as sysdba` | `sqlcmd -S host -E` | Peer/SSPI auth when configured |
| Run one query and exit | `echo "select 1 from dual;" \| sqlplus -S user@db` | `sqlcmd -Q "SELECT 1"` | `psql -c "SELECT 1"` |
| Run a script | `@script.sql` / `sqlplus user@db @script.sql` | `:r script.sql` / `-i script.sql` | `\i script.sql` / `-f script.sql` |
| Statement terminator | `;` or `/` | `GO` (sends batch) | `;` |
| Rerun last statement | `/` | `GO` again (buffer persists until reset) | `\g` |
| Show buffer | `L` | (none; use `:ED`) | `\p` |
| Edit buffer | `ED` | `:ED` | `\e` |
| Describe a table | `DESC owner.table` | `EXEC sp_help 'schema.table'` | `\d schema.table` |
| List tables | `SELECT table_name FROM user_tables;` | `SELECT name FROM sys.tables;` | `\dt` |
| List schemas | `SELECT username FROM dba_users;` | `SELECT name FROM sys.schemas;` | `\dn` |
| List databases | `SELECT name FROM v$pdbs;` | `SELECT name FROM sys.databases;` | `\l` |
| Switch database | `ALTER SESSION SET CONTAINER = pdb;` | `USE dbname` | `\c dbname` |
| Current user | `SHOW USER` | `SELECT SUSER_NAME();` | `SELECT current_user;` |
| Output to file | `SPOOL file` ... `SPOOL OFF` | `:out file` | `\o file` |
| CSV output | `SET MARKUP CSV ON` | `-s "," -W` | `\pset format csv` |
| Vertical row display | (none; use `COLUMN` formatting) | (none) | `\x` |
| Timing | `SET TIMING ON` | `SET STATISTICS TIME ON` | `\timing` |
| Variables | `DEFINE x = 5` / `&x` | `:setvar x 5` / `$(x)` | `\set x 5` / `:x` |
| Stop on error | `WHENEVER SQLERROR EXIT FAILURE` | `:on error exit` / `-b` | `\set ON_ERROR_STOP on` |
| OS command | `HOST cmd` | `:!! cmd` | `\! cmd` |
| Help | `HELP INDEX` | `:help` / `-?` | `\?`, `\h` |
| Quit | `EXIT` | `EXIT` | `\q` |
| Autocommit default | Off | On | On |
| History / tab completion | Windows console only; use SQLcl or `rlwrap` | Windows console only | Built in |

---

## 12. Cross-Dialect SQL Translation Table

| Operation | Oracle | SQL Server | PostgreSQL |
|---|---|---|---|
| First 10 rows | `FETCH FIRST 10 ROWS ONLY` / `WHERE ROWNUM <= 10` | `TOP 10` | `LIMIT 10` |
| Current date/time | `SYSDATE`, `SYSTIMESTAMP` | `GETDATE()`, `SYSDATETIME()` | `now()` |
| Null replacement | `NVL(a,b)` | `ISNULL(a,b)` | `COALESCE(a,b)` |
| Concatenation | `a \|\| b` | `a + b`, `CONCAT(a,b)` | `a \|\| b` |
| String length | `LENGTH(s)` | `LEN(s)` | `LENGTH(s)` |
| Substring | `SUBSTR(s,1,5)` | `SUBSTRING(s,1,5)` | `SUBSTRING(s,1,5)` / `SUBSTR` |
| Date to text | `TO_CHAR(d,'YYYY-MM-DD')` | `FORMAT(d,'yyyy-MM-dd')` / `CONVERT` | `TO_CHAR(d,'YYYY-MM-DD')` |
| Text to date | `TO_DATE(s,'YYYY-MM-DD')` | `CONVERT(date, s)` | `TO_DATE(s,'YYYY-MM-DD')` / `s::date` |
| Add days | `d + 7` | `DATEADD(day,7,d)` | `d + INTERVAL '7 days'` |
| Expression without table | `SELECT 1 FROM dual` | `SELECT 1` | `SELECT 1` |
| Auto-increment | `GENERATED AS IDENTITY` / sequence | `IDENTITY(1,1)` | `GENERATED AS IDENTITY` / `SERIAL` |
| Conditional | `CASE`, `DECODE` | `CASE`, `IIF` | `CASE` |
| Case-insensitive search | `UPPER(col) LIKE 'X%'` | Depends on collation | `ILIKE` |
| Regular expressions | `REGEXP_LIKE` | Limited (`LIKE` patterns; `REGEXP_LIKE` in SQL Server 2025) | `~`, `~*` |
| List tables | `user_tables`, `all_tables`, `dba_tables` | `sys.tables` | `pg_tables`, `information_schema.tables` |
| Describe | `DESC t` | `sp_help 't'` | `\d t` |
| Procedural language | PL/SQL | T-SQL | PL/pgSQL |
| Anonymous block | `BEGIN ... END;` + `/` | Batch + `GO` | `DO $$ ... $$;` |
| Print debug output | `DBMS_OUTPUT.PUT_LINE` | `PRINT` | `RAISE NOTICE` |

---

## 13. Connectivity Layers: ODBC, JDBC, OLE DB, Native Drivers

### 13.1 Driver types

| Interface | Origin | Used by |
|---|---|---|
| **ODBC** (Open Database Connectivity) | Microsoft, 1992; based on the SQL Call Level Interface standard | Excel, Power BI, Access, many Windows tools, Python `pyodbc` |
| **JDBC** | Sun/Oracle, Java | Java applications, SQL Developer, DBeaver, SQLcl |
| **OLE DB** | Microsoft, COM-based | Legacy Windows applications, SSIS, linked servers |
| **ADO.NET providers** | Microsoft .NET | C#/.NET applications (`Oracle.ManagedDataAccess`, `Microsoft.Data.SqlClient`, `Npgsql`) |
| **Native client libraries** | Vendor | OCI (Oracle Call Interface), TDS (SQL Server wire protocol), libpq (PostgreSQL) |

ODBC remains the most widely supported universal interface on Windows. Most ODBC drivers are thin wrappers around the vendor's native library.

### 13.2 ODBC components

```
Application (Excel)
   │ ODBC API calls
   ▼
ODBC Driver Manager (odbc32.dll on Windows; unixODBC on Linux)
   │ loads driver named in DSN / connection string
   ▼
ODBC Driver (e.g., Oracle in instantclient_19)
   │ calls native client library (OCI)
   ▼
Network → Database listener
```

- **DSN (Data Source Name):** A saved connection definition.
  - **User DSN:** Visible only to one Windows user.
  - **System DSN:** Visible to all users on the machine.
  - **File DSN:** Stored in a `.dsn` file, portable.
- **DSN-less connection string:** All settings supplied inline, e.g.:
  ```
  Driver={Oracle in instantclient_19_20};DBQ=dbserver01:1521/SMPDB;Uid=lims_report;Pwd=...;
  ```

### 13.3 Bitness (32-bit vs 64-bit)

A process can only load drivers that match its own architecture.

| Application | Driver required |
|---|---|
| 64-bit Excel / Power BI | 64-bit ODBC driver and 64-bit Oracle client |
| 32-bit Excel or legacy 32-bit tools | 32-bit ODBC driver and 32-bit Oracle client |

On 64-bit Windows, two separate ODBC administrators exist:

| Path | Manages |
|---|---|
| `C:\Windows\System32\odbcad32.exe` | **64-bit** drivers and DSNs |
| `C:\Windows\SysWOW64\odbcad32.exe` | **32-bit** drivers and DSNs |

The folder names are counterintuitive but correct. Both 32-bit and 64-bit Oracle clients can be installed side by side on one machine.

### 13.4 Oracle client options

| Option | Contents | Notes |
|---|---|---|
| **Full Oracle Client** | OCI, SQL\*Plus, ODBC, tools, Net configuration assistant | Large installer |
| **Instant Client Basic** | OCI libraries only | Unzip-and-go, no installer |
| **Instant Client ODBC package** | ODBC driver + `odbc_install.exe` to register it | Requires Basic |
| **Instant Client SQL\*Plus package** | SQL\*Plus executable | Requires Basic |
| **Instant Client Tools package** | Data Pump, SQL\*Loader, etc. | Requires Basic |
| **ODP.NET Managed Driver** | Pure .NET provider | No Oracle client needed |
| **python-oracledb (thin mode)** | Pure Python driver | No Oracle client needed |
| **JDBC thin driver (`ojdbc*.jar`)** | Pure Java driver | No Oracle client needed |

"Thin" or "managed" drivers implement the Oracle network protocol directly and avoid bitness and client installation issues entirely.

### 13.5 Vendor-layer drivers

Application-specific drivers (such as a LIMS ODBC driver or the AspenTech SQLplus ODBC driver) sit on top of or beside the database connection and expose the application's logical model, security, or query language instead of raw tables. They are useful when application-level rules must be respected; direct database drivers are faster and more flexible when reading raw data.

---

## 14. Automation Capabilities

### 14.1 Oracle

| Mechanism | Description |
|---|---|
| **PL/SQL packages, procedures, functions, triggers** | Server-side logic compiled and stored in the database |
| **DBMS_SCHEDULER** | Built-in job scheduler with calendars, chains, and external jobs |
| **SQL\*Plus / SQLcl scripts** | Batch scripts driven by the OS scheduler, with `WHENEVER SQLERROR` for exit codes |
| **Materialized views** | Pre-computed query results refreshed on a schedule or on commit |
| **Database links** | Query other Oracle databases (or other engines via Heterogeneous Services) |
| **External tables** | Read flat files as if they were tables |
| **SQL\*Loader, Data Pump (`expdp`/`impdp`)** | Bulk load and bulk export/import |
| **ORDS (Oracle REST Data Services)** | Expose tables, views, and PL/SQL as REST endpoints |

### 14.2 SQL Server

| Mechanism | Description |
|---|---|
| **T-SQL stored procedures, triggers** | Server-side logic |
| **SQL Server Agent** | Job scheduler with steps, schedules, alerts |
| **SSIS (Integration Services)** | Visual ETL pipelines |
| **PowerShell `SqlServer` module and `dbatools`** | Scripted administration and data movement |
| **Linked servers** | Query Oracle and other sources through OLE DB/ODBC |
| **bcp** | Bulk copy utility for import/export |

### 14.3 PostgreSQL

| Mechanism | Description |
|---|---|
| **PL/pgSQL and other procedural languages** | Server-side logic, including Python via PL/Python |
| **pg_cron extension** | In-database job scheduler |
| **psql scripting** | Rich variables, conditionals (`\if`), and `ON_ERROR_STOP` |
| **Foreign Data Wrappers** | Live querying of Oracle, SQL Server, files, and other sources |
| **`COPY`** | High-speed bulk load/unload |
| **Logical replication** | Stream changes to other PostgreSQL databases |

### 14.4 InfoPlus.21

| Mechanism | Description |
|---|---|
| **Aspen SQLplus query records** | Stored queries that execute on a schedule or on an event (such as a tag value change) |
| **External tasks** | Custom programs using AspenTech APIs |
| **ODBC / OLE DB access** | Pull historian data into other systems |

### 14.5 General-purpose tooling

| Tool | Relevant capability |
|---|---|
| **Python** (`oracledb`, `pyodbc`, `psycopg`, `pandas`, `SQLAlchemy`) | Cross-database extraction, transformation, and export to Excel/CSV |
| **PowerShell** | Windows-native scheduling and ODBC/.NET database access |
| **Excel Power Query** | Parameterized, refreshable queries through ODBC or native connectors |
| **Windows Task Scheduler / cron** | Trigger CLI scripts on a schedule |

---

## 15. Practical Walkthrough: First Oracle Session

This sequence explores an unfamiliar Oracle database safely using only read operations.

**Step 1: Connect**
```
sqlplus / as sysdba
```

**Step 2: Configure the display**
```
SET LINESIZE 250 PAGESIZE 100 TRIMSPOOL ON TAB OFF
SET TIMING ON
```

**Step 3: Identify version and instance**
```sql
SELECT banner FROM v$version;
SELECT instance_name, host_name, status FROM v$instance;
SELECT name, cdb, log_mode FROM v$database;
```

**Step 4: Locate the application container**
```sql
SHOW CON_NAME
SELECT con_id, name, open_mode FROM v$pdbs;
ALTER SESSION SET CONTAINER = <PDB_NAME>;
SHOW CON_NAME
```

**Step 5: Identify application schemas**
```sql
COLUMN owner FORMAT A30
SELECT owner, COUNT(*) AS tables
FROM dba_tables
WHERE owner NOT IN ('SYS','SYSTEM','XDB','MDSYS','CTXSYS','ORDSYS','OUTLN',
                    'DBSNMP','APPQOSSYS','WMSYS','OJVMSYS','LBACSYS','DVSYS',
                    'GSMADMIN_INTERNAL','AUDSYS','OLAPSYS','ORDDATA')
GROUP BY owner
ORDER BY tables DESC;
```

**Step 6: List the largest tables in the application schema**
```sql
COLUMN table_name FORMAT A40
SELECT table_name, num_rows, last_analyzed
FROM dba_tables
WHERE owner = 'SMOWNER'
ORDER BY num_rows DESC NULLS LAST
FETCH FIRST 50 ROWS ONLY;
```
(`num_rows` comes from optimizer statistics and may be approximate.)

**Step 7: Search for tables or columns by name**
```sql
SELECT table_name FROM dba_tables
WHERE owner = 'SMOWNER' AND table_name LIKE '%SAMPLE%';

COLUMN column_name FORMAT A30
SELECT table_name, column_name, data_type
FROM dba_tab_columns
WHERE owner = 'SMOWNER' AND column_name LIKE '%STATUS%'
ORDER BY table_name;
```

**Step 8: Describe and sample a table**
```sql
DESC smowner.sample
SELECT * FROM smowner.sample FETCH FIRST 10 ROWS ONLY;
```

**Step 9: Discover relationships (foreign keys)**
```sql
COLUMN child_table FORMAT A30
COLUMN parent_table FORMAT A30
SELECT c.table_name  AS child_table,
       cc.column_name AS child_column,
       p.table_name  AS parent_table
FROM dba_constraints c
JOIN dba_cons_columns cc ON cc.owner = c.owner AND cc.constraint_name = c.constraint_name
JOIN dba_constraints p   ON p.owner = c.r_owner AND p.constraint_name = c.r_constraint_name
WHERE c.owner = 'SMOWNER' AND c.constraint_type = 'R'
ORDER BY child_table;
```
Many application schemas enforce relationships in application code rather than foreign keys. If this returns few rows, infer relationships from matching column names instead.

**Step 10: Review existing views and comments**
```sql
SELECT view_name FROM dba_views WHERE owner = 'SMOWNER';
SELECT table_name, comments FROM dba_tab_comments
WHERE owner = 'SMOWNER' AND comments IS NOT NULL;
```

**Step 11: Export results**
```
SET MARKUP CSV ON
SPOOL C:\temp\sm_tables.csv
SELECT table_name, num_rows FROM dba_tables WHERE owner = 'SMOWNER' ORDER BY 1;
SPOOL OFF
SET MARKUP CSV OFF
```

**Step 12: Exit cleanly**
```
ROLLBACK;
EXIT
```

---

## 16. Safety Rules for Production Systems

1. **Use SYSDBA only for administration.** For browsing and reporting, create and use a least-privilege read-only account.
2. **Restrict exploration to `SELECT`, `DESC`, and dictionary views.** These never modify data.
3. **Do not issue `UPDATE`, `DELETE`, `INSERT`, `MERGE`, `TRUNCATE`, `DROP`, or `ALTER` against application schemas.** Application data integrity, audit trails, and validation status may depend on changes going through the application.
4. **Remember Oracle transaction rules.** DML is not committed until `COMMIT`; `EXIT` commits by default; DDL commits implicitly and cannot be rolled back. Use `ROLLBACK` before exiting if anything was changed accidentally.
5. **Limit result sizes** with `FETCH FIRST n ROWS ONLY` or `ROWNUM` when exploring large tables.
6. **Avoid heavy queries during peak laboratory hours.** Full scans of large tables consume shared resources.
7. **Avoid inline passwords** on the command line; they can be visible in process listings and shell history.
8. **In regulated (GxP) environments,** confirm that direct database access for reporting is permitted by quality and validation procedures, and document the reporting account and views.

---

*End of document.*
