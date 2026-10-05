# PCN SQL Server: build, permissions and file ingestion guide

[← Back to index](README.md)

Oct 2, 2026 · @Dalto

## At a glance

Build one SQL Server instance on the OPC/comm server, holding one database, `OT_Staging`. A PowerShell loader reads each file and bulk-copies it into staging tables. Task Scheduler runs the loader under a new, dedicated PCN domain account. Level 3 pulls from staging over one encrypted port.

| Question | Short answer |
| --- | --- |
| One database or many? | One instance per server and one database for your work on each: `OT_Staging` on the PCN, `OT_Hub` on Level 3. Separate things with schemas, not databases. Split only when retention, backups or ownership differ (section 2). |
| Which account runs SQL Server? | Its own virtual service account, `NT SERVICE\MSSQL$OTSTAGE`. Not the 800xA service account. |
| Which account loads the files? | A new PCN domain account, `PCN\svc_sqlload`, with rights to the data folders and a loader role in SQL. |
| The 800xA and IP21 service accounts? | Leave the 800xA account to 800xA. The IP21 account belongs to the Level 3 domain, which the PCN does not trust, so it cannot log on to PCN servers at all. |
| How does Level 3 get in? | A read-only SQL login, `l3_reader`, over TCP 1433 with encryption. This needs mixed-mode authentication. |
| Where does the scripting live? | File work (copy, read, parse, archive) lives in one PowerShell script on the comm server. Batch bookkeeping, validation and purge live in stored procedures in `OT_Staging`. Merge and publish logic lives on Level 3. |
| What schedules it? | Task Scheduler on the PCN, because Express has no SQL Agent. SQL Agent on Level 3. |
| Easiest way to ingest? | PowerShell reads the file with .NET's built-in parser and streams rows into SQL with SqlBulkCopy. No installs, no file rights for SQL Server itself. It handles CSV, delimited .txt and .dat, and fixed width. |
| Which file types are easy? | Anything that is text: CSV, tab or pipe delimited, fixed-width .dat or .txt, JSON, XML. Excel needs a conversion step. A binary .dat needs the vendor's export tool. |

The sections follow build order. Section 14 is the step-by-step checklist; section 15 lists the errors you are most likely to hit.

## 1. What lives where

Files become rows on the comm server, and Level 3 pulls those rows across the firewall. Scripts handle the file work, stored procedures handle the database work, and each server has one scheduler.

&#91;embedded content: what lives where · PCN comm server and Level 3 server\]

Read it top to bottom, the way data moves. The live-tag path through Cim-IO stays exactly as it is today.

| Work | Lives in | Runs as | Started by |
| --- | --- | --- | --- |
| Copy files from sources | `Collect-SourceFiles.ps1` on the comm server | `PCN\svc_sqlload` | Task Scheduler, every 5 minutes |
| Parse files and bulk-load rows | `Load-StagingFiles.ps1` on the comm server | `PCN\svc_sqlload` | The same task, straight after collecting |
| Batch log, duplicate check, purge | Stored procedures in `OT_Staging.etl` | Whoever calls them: the loader | The loader; the purge runs nightly |
| Backups and trimming | `Nightly-Maintenance.ps1` | `PCN\svc_sqlload`; SQL Server writes the .bak files itself | Task Scheduler, 02:10 |
| Pull, merge, reconcile, publish | Stored procedures in `OT_Hub` on Level 3 | SQL Agent's service account | SQL Agent, every 15 minutes |
| Alerts | Agent job and Database Mail on Level 3 | SQL Agent's service account | Hourly |
| The source code itself | Version control; deployed copies in `D:\OTData\scripts` and in each database | Not applicable | You, when you change something |

## 2. One database or several

Use one database per server for your own work and separate its parts with schemas. A second database earns its place only when its data lives by different rules.

Three levels, so the words stay straight:

- **Instance**: one SQL Server installation, one Windows service. One per server.
- **Database**: its own data and log files, backups, recovery model and list of users.
- **Schema**: a named folder inside a database (`etl`, `stg`, `master`, `pub`) that you can grant permissions on as a unit.

Split into a second database only when one of these is true:

| Reason to split | Example here |
| --- | --- |
| Different backup or recovery needs | Years of analyzer results backed up weekly, beside master data backed up nightly |
| Different retention | Raw results kept five years; staging kept 14 days |
| Different owner or security boundary | A database another team or a vendor manages |
| Restores get too slow | A results database too big to restore inside your outage window |

Otherwise stay in one. Foreign keys cannot cross databases, one backup is consistent across every table, and there is one set of users and roles to manage.

| Server | Databases | Inside your database |
| --- | --- | --- |
| PCN comm server, instance `OTSTAGE` | `OT_Staging`; optionally a small `DBA` database for maintenance scripts | Schemas `etl` (control) and `stg` (one table per source) |
| Level 3 server (IP21 and SQL Server together) | `OT_Hub`; later `OT_Results` if analyzer history grows large; Aspen's own databases left alone | Schemas `etl`, `stg`, `master`, `hist`, `pub` |

Naming that keeps the next person oriented:

- Databases start with `OT_`.
- Schemas are lowercase role names: `etl`, `stg`, `master`, `pub`.
- Staging tables are `stg.<source>_<thing>`, such as `stg.gc01_result` or `stg.abb_objects`.
- Procedures are `etl.usp_<Verb><Object>`, such as `etl.usp_BatchStart`. Views are `pub.v_<thing>`.

## 3. Accounts and permissions

Give each job its own identity with only the rights that job needs. Four identities do all the work on the PCN, and neither the 800xA nor the IP21 service account is one of them.

| Identity | Kind | Used for | Rights it needs |
| --- | --- | --- | --- |
| `NT SERVICE\MSSQL$OTSTAGE` | Virtual account the installer creates; no password | Runs the SQL Server engine | Its data, log and backup folders (the installer grants these). Read on `D:\OTData\work` only if you use BULK INSERT |
| `PCN\svc_sqlload` | New PCN domain service account | Runs the collector and loader scripts from Task Scheduler | Read on source shares; Modify on the `D:\OTData` working folders; Read on the scripts folder; Log on as a batch job; a Windows login in SQL with `role_loader` |
| `l3_reader` | SQL login | Level 3 pulling staged rows | `SELECT` on schema `stg` and on one view in `etl`. Nothing else |
| `PCN\SQL_Admins` | PCN domain group | You and anyone else who administers the instance | `sysadmin` |

### Why not the 800xA service account

- It is the identity your control system runs as. Anything running under it can act as 800xA, so a bug or a compromised script would have that reach.
- Its password changes follow ABB's procedure. Tie SQL Server to it and every password change becomes a SQL outage as well.
- Logs can no longer tell your activity apart from 800xA's.

The IP21 service account belongs to the Level 3 domain. The PCN domain does not trust it, so it cannot authenticate to PCN servers at all. On Level 3, run SQL Server under its own virtual accounts there too, not under the IP21 account.

If the PCN domain supports it, make `svc_sqlload` a group managed service account (gMSA): Windows rotates its password and Task Scheduler can run as it (section 11). If you cannot get any new domain account, use the push case in section 6: each source writes into a share on the comm server with its existing account, and the loader runs as a local account on the comm server.

### How SQL Server decides who reads a file

This is the rule behind most "access denied" errors. It depends on which process opens the file.

| Method | Who opens the file | What that identity needs |
| --- | --- | --- |
| PowerShell + SqlBulkCopy (recommended) | The script's own Windows account. SQL Server never touches the file. | NTFS Read on the file; `INSERT` on the staging table |
| `bcp.exe` | The account running bcp, the same way | NTFS Read; `INSERT` |
| `BULK INSERT` run by a Windows login | SQL Server, impersonating that login | NTFS Read for that login; `ADMINISTER BULK OPERATIONS`; `INSERT`. A path on another server also needs Kerberos delegation, so keep files local |
| `BULK INSERT` run by a SQL login | SQL Server's own service account | NTFS Read for `NT SERVICE\MSSQL$OTSTAGE`; `ADMINISTER BULK OPERATIONS`; `INSERT` |

`ADMINISTER BULK OPERATIONS` is a server-wide permission. BULK INSERT also needs `ALTER` on the table when it has CHECK or foreign-key constraints or triggers, or when you use KEEPIDENTITY. Staging tables have none of those, so `INSERT` is enough.

### Inside SQL Server

- Grant to roles, never to individual users. Two roles cover the PCN: `role_loader` and `role_l3_pull`.
- A user with `EXECUTE` on a procedure needs no rights on the tables it touches, as long as the procedure and tables share an owner (`dbo`). This is called ownership chaining.
- Dynamic SQL breaks that chain. Bulk copy and the purge both touch tables directly, so `role_loader` also gets `INSERT` and `DELETE` on `stg`.

### Windows rights for `svc_sqlload`

- Log on as a batch job on the comm server, so scheduled tasks can run as it.
- Deny log on locally and deny log on through Remote Desktop Services. Nobody should sign in as it.
- Not a local administrator anywhere.
- On source servers, Read on the share and folder holding the files. Modify only if you collect by moving files (section 6).

The logins, users and roles are created by the security script in section 5.

## 4. Install and configure the PCN instance

Install the Database Engine only, as a named instance `OTSTAGE`, with mixed-mode authentication, a fixed TCP port and forced encryption. Allow about an hour, including restarts.

### Before you install

- If the comm server is a node of the 800xA system, ABB's rules for third-party software apply, so check with ABB first. If it is a standalone OPC and Cim-IO box you own, your own policy applies.
- A SQL Server is already on that box. Find out what it is with `SELECT @@VERSION, SERVERPROPERTY('Edition'), SERVERPROPERTY('InstanceName');`. If another application owns it, leave it alone and install `OTSTAGE` beside it. If it is yours and unused, you can apply the settings below to it instead.
- The PCN has no internet. Download the installer and the latest cumulative update on a connected machine, scan them under your site's media policy, and carry them over. The Express installer has a Download Media option for an offline package.

### Edition

|  | Express | Standard |
| --- | --- | --- |
| Cost | Free | Licensed |
| Size per database | 10 GB (SQL Server 2022) | No practical limit |
| Memory for data cache | About 1.4 GB | Up to 128 GB |
| SQL Agent, Database Mail, backup compression | No | Yes |
| Fit here | Staging. Recommended | Only if you want SQL Agent on the PCN |

These limits change between releases, so check them for the version you download. Use the same major version as Level 3 where you can.

### Installer choices

| Setting | Choose | Why |
| --- | --- | --- |
| Features | Database Engine Services only | Less to patch |
| Instance | Named instance `OTSTAGE` | Clear name; no clash with the SQL Server already there |
| Engine service account | `NT SERVICE\MSSQL$OTSTAGE` (the default), startup Automatic | No password to manage |
| SQL Server Browser | Disabled | A fixed port makes it unnecessary |
| Perform Volume Maintenance Task privilege | Granted (installer checkbox) | Data files grow faster |
| Collation | The same as Level 3, usually `SQL_Latin1_General_CP1_CI_AS` | Different collations break joins across servers. Check Level 3 with `SELECT SERVERPROPERTY('Collation');` |
| Authentication | Mixed Mode, with a long `sa` password (you disable `sa` later) | Level 3 cannot use Windows authentication across the two domains |
| Administrators | The `PCN\SQL_Admins` group | Not individual people |
| Directories | Data `D:\SQLData`, logs `D:\SQLLogs`, TempDB `D:\SQLTempDB`, backups `D:\SQLBackup` | Keeps SQL Server off C: |
| TempDB | The installer's default number of files | Fine for this load |

Apply the latest cumulative update straight after setup.

### Instance settings

Run this as a sysadmin once setup is done.

```sql
EXEC sys.sp_configure N'show advanced options', 1;  RECONFIGURE;
EXEC sys.sp_configure N'max server memory (MB)', 2048;      -- size to the box; leave room for Windows, Cim-IO and OPC
EXEC sys.sp_configure N'cost threshold for parallelism', 50;
EXEC sys.sp_configure N'max degree of parallelism', 2;
EXEC sys.sp_configure N'optimize for ad hoc workloads', 1;
EXEC sys.sp_configure N'remote admin connections', 1;       -- emergency admin connection
EXEC sys.sp_configure N'xp_cmdshell', 0;                    -- no Windows shell from inside SQL
EXEC sys.sp_configure N'Ole Automation Procedures', 0;
EXEC sys.sp_configure N'Ad Hoc Distributed Queries', 0;
EXEC sys.sp_configure N'clr enabled', 0;
RECONFIGURE;

-- Keep 30 error logs instead of 6
EXEC master.dbo.xp_instance_regwrite N'HKEY_LOCAL_MACHINE',
     N'Software\Microsoft\MSSQLServer\MSSQLServer', N'NumErrorLogs', REG_DWORD, 30;

-- Admin group, then sa off (only after you have signed in through the group)
IF NOT EXISTS (SELECT 1 FROM sys.server_principals WHERE name = N'PCN\SQL_Admins') CREATE LOGIN [PCN\SQL_Admins] FROM WINDOWS;
ALTER SERVER ROLE sysadmin ADD MEMBER [PCN\SQL_Admins];
ALTER LOGIN sa DISABLE;
```

Express caps its data cache at about 1.4 GB whatever you set. Set the memory value anyway so it is right if you ever move to Standard. In SSMS, also set Server Properties > Security > Login auditing to "Both failed and successful logins".

### Network

1. In SQL Server Configuration Manager, open Protocols for OTSTAGE. Enable TCP/IP, disable Named Pipes, leave Shared Memory on.
2. In TCP/IP > IP Addresses > IPAll, clear TCP Dynamic Ports and set TCP Port to `1433`. If the SQL Server already on the box holds 1433, use another fixed port such as `14330` everywhere this guide says 1433.
3. Restart the SQL Server (OTSTAGE) service.
4. Open the port to the Level 3 server only:

```powershell
New-NetFirewallRule -DisplayName 'SQL OTSTAGE from Level 3' -Direction Inbound -Protocol TCP `
  -LocalPort 1433 -RemoteAddress <Level 3 server IP> -Action Allow -Profile Domain
```

5. Test locally with `sqlcmd -S tcp:localhost,1433 -E -Q "SELECT @@SERVERNAME"`. From Level 3, test with `Test-NetConnection <comm server> -Port 1433`.

### Encryption

1. Get a server certificate for the comm server's name: from a PCN certificate authority if you have one, otherwise self-signed:

```powershell
New-SelfSignedCertificate -DnsName 'commsrv.pcn.local','commsrv' -CertStoreLocation 'Cert:\LocalMachine\My' `
  -KeySpec KeyExchange -KeyExportPolicy Exportable -NotAfter (Get-Date).AddYears(5) -FriendlyName 'SQL OTSTAGE'
```

2. In `certlm.msc` > Personal > Certificates, right-click the certificate > All Tasks > Manage Private Keys, and give `NT SERVICE\MSSQL$OTSTAGE` Read.
3. In Configuration Manager > Protocols for OTSTAGE > Properties, pick the certificate on the Certificate tab and set Force Encryption to Yes on the Flags tab. Restart the service.
4. Export the certificate without its private key (.cer) and import it on the Level 3 server into Local Computer > Trusted Root Certification Authorities. Level 3 then trusts the connection without `TrustServerCertificate`.

### Hardening check

- `sa` disabled; the only SQL login is `l3_reader`.
- `xp_cmdshell`, CLR and OLE Automation off.
- Browser off, Named Pipes off, fixed port, firewall rule scoped to the Level 3 address.
- Service accounts are not local administrators.
- Cumulative updates go in with your normal PCN patch window.

## 5. Build the staging database

The scripts below create the database, two schemas, three control tables, the procedures the loader calls, the view Level 3 reads, and the logins. Run them in order as a sysadmin.

### Database and schemas

```sql
CREATE DATABASE OT_Staging
ON PRIMARY (NAME = N'OT_Staging',     FILENAME = N'D:\SQLData\OT_Staging.mdf',     SIZE = 1GB,   FILEGROWTH = 256MB)
LOG ON     (NAME = N'OT_Staging_log', FILENAME = N'D:\SQLLogs\OT_Staging_log.ldf', SIZE = 512MB, FILEGROWTH = 256MB);
GO
ALTER DATABASE OT_Staging SET RECOVERY SIMPLE;       -- staging can be rebuilt from the archived files
ALTER DATABASE OT_Staging SET PAGE_VERIFY CHECKSUM;
ALTER AUTHORIZATION ON DATABASE::OT_Staging TO sa;   -- owned by a disabled login, not a person
GO
USE OT_Staging;
GO
CREATE SCHEMA etl AUTHORIZATION dbo;   -- control: sources, batches, rejects, procedures
GO
CREATE SCHEMA stg AUTHORIZATION dbo;   -- landing tables, one per source
GO
```

### Control tables

`etl.Source` is the registry: one row per feed, telling the loader where files are and how to read them. `etl.Batch` logs every file. `etl.RejectRow` keeps lines that could not be read.

```sql
CREATE TABLE etl.Source (
    source_code    VARCHAR(30)    NOT NULL CONSTRAINT PK_Source PRIMARY KEY,  -- also the folder name: inbound\gc01
    description    NVARCHAR(200)  NULL,
    file_pattern   NVARCHAR(100)  NOT NULL,                     -- '*.csv', 'GC01_*.dat'
    file_format    VARCHAR(20)    NOT NULL CONSTRAINT CK_Source_format
                   CHECK (file_format IN ('delimited','fixed','lines','json','xml')),
    delimiter      NVARCHAR(5)    NULL,                         -- ',' ';' '|' or CHAR(9) for tab
    field_widths   VARCHAR(400)   NULL,                         -- fixed width only: '10,19,12,12'
    has_header     BIT            NOT NULL CONSTRAINT DF_Source_header DEFAULT 1,
    code_page      INT            NOT NULL CONSTRAINT DF_Source_cp DEFAULT 65001,  -- 65001 UTF-8, 1252 ANSI, 1200 UTF-16
    target_table   NVARCHAR(256)  NOT NULL,                     -- 'stg.gc01_result'
    load_proc      NVARCHAR(256)  NULL,                         -- json and xml only: 'etl.usp_Load_lab_json'
    load_pattern   VARCHAR(10)    NOT NULL CONSTRAINT CK_Source_pattern
                   CHECK (load_pattern IN ('snapshot','append','change')),
    retention_days INT            NOT NULL CONSTRAINT DF_Source_ret DEFAULT 14,
    is_active      BIT            NOT NULL CONSTRAINT DF_Source_active DEFAULT 1,
    owner_name     NVARCHAR(100)  NULL
);

CREATE TABLE etl.Batch (
    batch_id       INT IDENTITY(1,1) NOT NULL CONSTRAINT PK_Batch PRIMARY KEY,
    source_code    VARCHAR(30)    NOT NULL CONSTRAINT FK_Batch_Source REFERENCES etl.Source (source_code),
    file_name      NVARCHAR(260)  NOT NULL,
    file_hash      CHAR(64)       NOT NULL,                     -- SHA-256 of the file's bytes
    file_bytes     BIGINT         NOT NULL,
    file_time_utc  DATETIME2(0)   NOT NULL,                     -- the file's last-modified time
    status         VARCHAR(10)    NOT NULL CONSTRAINT CK_Batch_status
                   CHECK (status IN ('Loading','Loaded','Failed','Duplicate')),
    rows_loaded    INT            NULL,
    rows_rejected  INT            NULL,
    started_utc    DATETIME2(0)   NOT NULL CONSTRAINT DF_Batch_started DEFAULT SYSUTCDATETIME(),
    ended_utc      DATETIME2(0)   NULL,
    error_message  NVARCHAR(2000) NULL
);
CREATE INDEX IX_Batch_source ON etl.Batch (source_code, status, batch_id) INCLUDE (file_hash);

CREATE TABLE etl.RejectRow (
    batch_id  INT            NOT NULL CONSTRAINT FK_RejectRow_Batch REFERENCES etl.Batch (batch_id),
    row_num   INT            NOT NULL,
    reason    NVARCHAR(200)  NOT NULL,
    raw_line  NVARCHAR(4000) NULL,
    CONSTRAINT PK_RejectRow PRIMARY KEY (batch_id, row_num)
);
GO
```

### Staging table template

One table per source. Housekeeping columns come first, then the file's columns in the file's order, all as text. The loader fills the file's first field into the first column after `loaded_utc`, the second into the next, and so on.

```sql
CREATE TABLE stg.gc01_result (
    batch_id    INT           NOT NULL,
    row_num     INT           NOT NULL,     -- record number: 1 = first data row
    loaded_utc  DATETIME2(0)  NOT NULL CONSTRAINT DF_gc01_result_loaded DEFAULT SYSUTCDATETIME(),
    sample_time NVARCHAR(50)  NULL,
    stream      NVARCHAR(50)  NULL,
    component   NVARCHAR(100) NULL,
    value       NVARCHAR(50)  NULL,
    unit        NVARCHAR(20)  NULL,
    status      NVARCHAR(20)  NULL,
    CONSTRAINT PK_gc01_result PRIMARY KEY CLUSTERED (batch_id, row_num)
);
GO

INSERT etl.Source (source_code, description, file_pattern, file_format, delimiter, has_header,
                   code_page, target_table, load_pattern, retention_days, owner_name)
VALUES ('gc01', N'GC-01 raw results', N'*.csv', 'delimited', N',', 1,
        65001, N'stg.gc01_result', 'append', 14, N'Analyzer technician');
GO
```

Text columns mean a bad value can never fail the load. Types, keys and checks are applied on Level 3, where a bad row goes to a reject table instead of stopping the file.

### Procedures the loader calls

```sql
-- Registers a file and says whether to load it.
-- Snapshot sources: a duplicate is a file identical to the LAST one loaded.
-- Append and change sources: a duplicate is any file loaded before.
CREATE OR ALTER PROCEDURE etl.usp_BatchStart
    @source_code VARCHAR(30), @file_name NVARCHAR(260), @file_hash CHAR(64),
    @file_bytes BIGINT, @file_time_utc DATETIME2(0)
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;
    DECLARE @pattern VARCHAR(10), @dup BIT = 0;

    SELECT @pattern = load_pattern FROM etl.Source WHERE source_code = @source_code AND is_active = 1;
    IF @pattern IS NULL THROW 50001, N'Unknown or inactive source.', 1;

    IF @pattern = 'snapshot'
        SELECT @dup = CASE WHEN last.file_hash = @file_hash THEN 1 ELSE 0 END
        FROM (SELECT TOP (1) file_hash FROM etl.Batch
              WHERE source_code = @source_code AND status = 'Loaded'
              ORDER BY batch_id DESC) AS last;
    ELSE IF EXISTS (SELECT 1 FROM etl.Batch
                    WHERE source_code = @source_code AND file_hash = @file_hash AND status = 'Loaded')
        SET @dup = 1;

    INSERT etl.Batch (source_code, file_name, file_hash, file_bytes, file_time_utc, status, ended_utc)
    VALUES (@source_code, @file_name, @file_hash, @file_bytes, @file_time_utc,
            CASE WHEN @dup = 1 THEN 'Duplicate' ELSE 'Loading' END,
            CASE WHEN @dup = 1 THEN SYSUTCDATETIME() END);

    SELECT CAST(SCOPE_IDENTITY() AS INT) AS batch_id,
           CASE WHEN @dup = 1 THEN 'Duplicate' ELSE 'Loading' END AS status;
END;
GO

-- Called inside the load transaction, so rows and status commit together.
CREATE OR ALTER PROCEDURE etl.usp_BatchEnd
    @batch_id INT, @rows_loaded INT, @rows_rejected INT
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE etl.Batch
    SET status = 'Loaded', rows_loaded = @rows_loaded, rows_rejected = @rows_rejected,
        ended_utc = SYSUTCDATETIME()
    WHERE batch_id = @batch_id AND status = 'Loading';
    IF @@ROWCOUNT <> 1 THROW 50002, N'Batch is not in Loading state.', 1;
END;
GO

CREATE OR ALTER PROCEDURE etl.usp_BatchFail
    @batch_id INT, @error NVARCHAR(2000)
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE etl.Batch
    SET status = 'Failed', error_message = LEFT(@error, 2000), ended_utc = SYSUTCDATETIME()
    WHERE batch_id = @batch_id AND status = 'Loading';
END;
GO

-- A run that died mid-load leaves its batch in Loading; its rows were rolled back with it.
CREATE OR ALTER PROCEDURE etl.usp_CleanupStale
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE etl.Batch
    SET status = 'Failed', error_message = N'Run ended before the batch finished.', ended_utc = SYSUTCDATETIME()
    WHERE status = 'Loading' AND started_utc < DATEADD(HOUR, -2, SYSUTCDATETIME());
END;
GO

-- Deletes staged rows older than each source's retention. Retention must outlast
-- the longest Level 3 outage you expect, or rows go before Level 3 has pulled them.
CREATE OR ALTER PROCEDURE etl.usp_Purge
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @sql NVARCHAR(MAX);
    SELECT @sql = STRING_AGG(CAST(
           N'DELETE t FROM ' + QUOTENAME(PARSENAME(s.target_table, 2)) + N'.' + QUOTENAME(PARSENAME(s.target_table, 1))
         + N' AS t JOIN etl.Batch AS b ON b.batch_id = t.batch_id'
         + N' WHERE b.source_code = ' + QUOTENAME(s.source_code, '''')
         + N' AND b.started_utc < DATEADD(DAY, -' + CAST(s.retention_days AS NVARCHAR(10)) + N', SYSUTCDATETIME());'
           AS NVARCHAR(MAX)), NCHAR(10))
    FROM etl.Source AS s;
    IF @sql IS NOT NULL EXEC sys.sp_executesql @sql;

    DELETE r FROM etl.RejectRow AS r JOIN etl.Batch AS b ON b.batch_id = r.batch_id
    WHERE b.started_utc < DATEADD(DAY, -30, SYSUTCDATETIME());
END;
GO

-- What Level 3 reads to know which batches are complete.
CREATE OR ALTER VIEW etl.vw_LoadedBatches
AS
SELECT batch_id, source_code, file_name, rows_loaded, rows_rejected, ended_utc
FROM etl.Batch
WHERE status = 'Loaded';
GO
```

### Security script

```sql
USE master;
CREATE LOGIN [PCN\svc_sqlload] FROM WINDOWS WITH DEFAULT_DATABASE = OT_Staging;
CREATE LOGIN l3_reader WITH PASSWORD = N'<long random passphrase>',
       CHECK_POLICY = ON, CHECK_EXPIRATION = OFF, DEFAULT_DATABASE = OT_Staging;
GO
USE OT_Staging;
CREATE USER [PCN\svc_sqlload] FOR LOGIN [PCN\svc_sqlload];
CREATE USER l3_reader FOR LOGIN l3_reader;

CREATE ROLE role_loader;
GRANT SELECT, INSERT, DELETE ON SCHEMA::stg TO role_loader;   -- bulk copy and purge
GRANT SELECT, EXECUTE ON SCHEMA::etl TO role_loader;          -- sources, batch log, procedures
GRANT INSERT ON etl.RejectRow TO role_loader;                 -- bad lines
ALTER ROLE role_loader ADD MEMBER [PCN\svc_sqlload];
ALTER ROLE db_backupoperator ADD MEMBER [PCN\svc_sqlload];    -- nightly backup task (section 13)

CREATE ROLE role_l3_pull;
GRANT SELECT ON SCHEMA::stg TO role_l3_pull;
GRANT SELECT ON etl.vw_LoadedBatches TO role_l3_pull;
ALTER ROLE role_l3_pull ADD MEMBER l3_reader;
GO

-- Only if you choose BULK INSERT (section 8):
-- USE master; GRANT ADMINISTER BULK OPERATIONS TO [PCN\svc_sqlload];
```

`CHECK_POLICY = ON` applies Windows password and lockout rules to `l3_reader`. A Level 3 job retrying with a wrong password can lock it out. If that worries you more than guessing does, set it `OFF` and rely on a long password plus the firewall scope.

## 6. Folder layout and file collection

Every file passes through the same folders on the comm server. How it gets into the first one depends on where it starts, and there are six cases.

### Folders

```text
D:\OTData\
  inbound\<source>\          files waiting to load
  work\                       a file sits here while it loads, so nothing else touches it
  archive\<source>\yyyy\MM\   loaded files, renamed <batch_id>_<original name>
  reject\<source>\            files that failed; the reason is in etl.Batch
  mirror\<source>\            copies of source folders, for sources that keep their files
  logs\                       one collector log and one loader log per day
  scripts\                    Collect-SourceFiles.ps1, Load-StagingFiles.ps1
```

Keep them all on one volume. A move within a volume is a rename: instant, and a file is never half in one place.

| Folder | `PCN\svc_sqlload` | `NT SERVICE\MSSQL$OTSTAGE` | `PCN\SQL_Admins` | Source accounts (push only) |
| --- | --- | --- | --- | --- |
| `inbound` | Modify | None | Full | Modify on their own subfolder |
| `work` | Modify | Read, only if you use BULK INSERT | Full | None |
| `archive`, `reject`, `mirror`, `logs` | Modify | None | Full | None |
| `scripts` | Read and execute | None | Full | None |

The scripts folder is read-only for the service account on purpose. If the account could edit its own script, anyone who obtained it could make it run anything.

```powershell
$root = 'D:\OTData'
'inbound','work','archive','reject','mirror','logs','scripts' |
  ForEach-Object { New-Item -ItemType Directory -Path "$root\$_" -Force | Out-Null }

# Remove inherited rights (e.g. Users), then grant explicitly
icacls $root /inheritance:r /grant:r 'BUILTIN\Administrators:(OI)(CI)F' 'NT AUTHORITY\SYSTEM:(OI)(CI)F' 'PCN\SQL_Admins:(OI)(CI)F'
foreach ($f in 'inbound','work','archive','reject','mirror','logs') {
  icacls "$root\$f" /grant 'PCN\svc_sqlload:(OI)(CI)M'
}
icacls "$root\scripts" /grant 'PCN\svc_sqlload:(OI)(CI)RX'
```

### The six collection cases

| Case | Example | How the file reaches `inbound` | Rights needed |
| --- | --- | --- | --- |
| 1. Push | The source application can write to a network path | It writes `\\COMMSRV\inbound$\gc01\name.tmp`, then renames it to `.csv` when complete | Share Change, plus NTFS Modify on that one subfolder, for the source's account |
| 2. Pull and move | A source folder you are allowed to empty | Collector runs `robocopy /MOV` | `svc_sqlload` Modify on the source folder |
| 3. Pull and copy | The source keeps its files | Collector mirrors the folder, then copies files it has not seen before | `svc_sqlload` Read on the source |
| 4. One file, overwritten | `objects.csv`, same name every day | Collector copies it with a timestamp whenever its modified time changes; `usp_BatchStart` skips unchanged content | `svc_sqlload` Read |
| 5. Growing file | A log the analyzer appends to all day | Best: ask for a new file per hour or day, then it is case 1, 2 or 3 | Read |
| 6. Already local | Cim-IO or an OPC tool writes on the comm server | Point that output straight at `inbound\<source>` | None extra |

Push is the most reliable because the source decides when a file is complete. The `.tmp`-then-rename rule matters: the loader only picks up names matching `file_pattern`, so a half-written `.tmp` is never read. For push, share the inbound folder once:

```powershell
New-SmbShare -Name 'inbound$' -Path 'D:\OTData\inbound' -FullAccess 'PCN\SQL_Admins' -ChangeAccess 'PCN\<source writer account>'
```

### Collector script

Add two columns to the source registry, so one table describes each feed end to end:

```sql
ALTER TABLE etl.Source ADD
    source_path  NVARCHAR(400) NULL,   -- \\SRV01\exports  or, for 'single', \\SRV01\exports\objects.csv
    collect_mode VARCHAR(10)   NULL CONSTRAINT CK_Source_collect
                 CHECK (collect_mode IN ('push','move','copy','single','local'));
```

`D:\OTData\scripts\Collect-SourceFiles.ps1` handles the pull cases. Push and local need no collector.

```powershell
# Collect-SourceFiles.ps1 - brings source files into D:\OTData\inbound\<source>. Runs as PCN\svc_sqlload.
$ErrorActionPreference = 'Stop'
$Root = 'D:\OTData'
$conn = 'Server=localhost\OTSTAGE;Database=OT_Staging;Integrated Security=SSPI;Application Name=OT Collector'
$log  = Join-Path $Root ('logs\collector_{0:yyyyMMdd}.log' -f (Get-Date))
function Write-Log([string]$m) { '{0:yyyy-MM-dd HH:mm:ss} {1}' -f (Get-Date), $m | Add-Content -Path $log }

$cn  = New-Object System.Data.SqlClient.SqlConnection $conn
$cmd = $cn.CreateCommand()
$cmd.CommandText = "SELECT source_code, file_pattern, source_path, collect_mode FROM etl.Source
                    WHERE is_active = 1 AND collect_mode IN ('move','copy','single')"
$cn.Open(); $sources = New-Object System.Data.DataTable; $sources.Load($cmd.ExecuteReader()); $cn.Close()

foreach ($s in $sources.Rows) {
  try {
    $inbound = Join-Path $Root "inbound\$($s.source_code)"
    New-Item -ItemType Directory -Path $inbound -Force | Out-Null
    switch ($s.collect_mode) {
      'move' {
        robocopy $s.source_path $inbound $s.file_pattern /MOV /R:2 /W:5 /NP /NJH /NJS "/LOG+:$log" | Out-Null
        if ($LASTEXITCODE -ge 8) { throw "robocopy failed with code $LASTEXITCODE" }
      }
      'copy' {
        $mirror = Join-Path $Root "mirror\$($s.source_code)"
        robocopy $s.source_path $mirror $s.file_pattern /R:2 /W:5 /NP /NJH /NJS "/LOG+:$log" | Out-Null
        if ($LASTEXITCODE -ge 8) { throw "robocopy failed with code $LASTEXITCODE" }
        $seenFile = Join-Path $Root "mirror\$($s.source_code).seen.txt"
        $seen = New-Object 'System.Collections.Generic.HashSet[string]'
        if (Test-Path $seenFile) { Get-Content $seenFile | ForEach-Object { [void]$seen.Add($_) } }
        Get-ChildItem -LiteralPath $mirror -Filter $s.file_pattern -File | ForEach-Object {
          $key = '{0}|{1}|{2:o}' -f $_.Name, $_.Length, $_.LastWriteTimeUtc
          if ($seen.Add($key)) {
            Copy-Item -LiteralPath $_.FullName -Destination (Join-Path $inbound $_.Name) -Force
            Add-Content -Path $seenFile -Value $key
          }
        }
      }
      'single' {
        $src  = Get-Item -LiteralPath $s.source_path
        $last = Join-Path $Root "mirror\$($s.source_code).last.txt"
        $stamp = '{0:o}' -f $src.LastWriteTimeUtc
        $old = if (Test-Path $last) { Get-Content $last } else { '' }
        # copy only when it changed, and only once the writer has finished (unchanged for 60 s)
        if ($stamp -ne $old -and $src.LastWriteTimeUtc -lt (Get-Date).ToUniversalTime().AddSeconds(-60)) {
          $name = '{0}_{1:yyyyMMdd_HHmmss}{2}' -f $src.BaseName, $src.LastWriteTime, $src.Extension
          Copy-Item -LiteralPath $src.FullName -Destination (Join-Path $inbound $name)
          Set-Content -Path $last -Value $stamp
        }
      }
    }
  } catch {
    Write-Log "$($s.source_code): $($_.Exception.Message)"   # one bad source never stops the others
  }
}
```

Robocopy exit codes below 8 mean success (1 = files copied, 0 = nothing new); 8 and above mean failures.

## 7. Know your file: CSV, .dat, .txt and friends

A file's extension says nothing about how to read it. A .dat can be a CSV, a fixed-width table, a printed report or binary. Spend five minutes looking inside before you write a staging table.

### Look inside

```powershell
$f = '\\SRV01\exports\GC01_20261002.dat'

# 1. The first lines as text. Gibberish here means binary.
Get-Content -LiteralPath $f -TotalCount 15

# 2. The first bytes in hex: the encoding mark and the line endings.
Format-Hex -LiteralPath $f | Select-Object -First 6

# 3. Fields per line. Every data line should give the same count (quoted commas will inflate it).
Get-Content -LiteralPath $f -TotalCount 50 | ForEach-Object { ($_ -split ',').Count } |
  Group-Object | Select-Object Name, Count
```

| In the hex | It means | Set in `etl.Source` |
| --- | --- | --- |
| `EF BB BF` at the very start | UTF-8 with a byte-order mark | `code_page = 65001` |
| `FF FE` at the start, then letters alternating with `00` | UTF-16, which Windows calls "Unicode" | `code_page = 1200` |
| No mark, plain letters | UTF-8 without a mark, or ANSI | `65001`; if accented letters come out wrong, `1252` |
| `0D 0A` at line ends | Windows line endings | Nothing; the default |
| `0A` alone at line ends | Unix line endings | Nothing for the PowerShell loader; BULK INSERT needs `ROWTERMINATOR = '0x0a'` |
| `09` between values | Tab delimited | `delimiter = CHAR(9)` |
| `00` bytes scattered with no pattern | Binary | Not loadable; see section 10 |

### File shapes

| What you see inside | Shape | `file_format` | Where |
| --- | --- | --- | --- |
| Values separated by `,` `;` `\|` or tab, one record per line | Delimited | `delimited` | Section 9 |
| Columns that line up at fixed positions, padded with spaces, no separator | Fixed width | `fixed`, with `field_widths` | Sections 9 and 10 |
| A title block, sections, totals, page breaks | Report-style text | `lines` | Section 10 |
| `name=value` on each line | Key-value | `lines` | Section 10 |
| `{ }` and `[ ]` | JSON | `json` | Section 10 |
| `<tags>` | XML | `xml` | Section 10 |
| An Excel workbook | Excel | Convert to CSV first | Section 10 |
| Unreadable text and NUL bytes | Binary | Vendor export or API | Section 10 |

.dat, .txt, .prn, .log, .asc and .out files are nearly always delimited or fixed-width text. Older analyzers often write fixed width or a printed report.

### Pin these down in the data contract

- Delimiter, quoting, and whether there is a header row.
- Encoding.
- Date format and time zone: local plant time or UTC, and what happens at daylight-saving changes.
- Decimal separator. Some analyzers write `1,25` with `;` between fields.
- How a missing or bad value is written: blank, `NaN`, `-9999`, `####`.
- What the file looks like when the analyzer is faulted or in calibration.

## 8. Ingestion methods compared

Use PowerShell with SqlBulkCopy by default. It needs nothing installed, keeps file access with the Windows account that already has it, and handles every text shape in one script. BULK INSERT is the all-T-SQL alternative when the files are clean CSV.

| Method | How it works | Good at | Watch out for | Permissions |
| --- | --- | --- | --- | --- |
| PowerShell + SqlBulkCopy (default) | The script parses the file with .NET's `TextFieldParser` and streams rows into SQL | Quoted CSV, any delimiter, fixed width, per-line checks; built into Windows PowerShell 5.1 | A script to maintain | NTFS Read for the script's account; `INSERT` on `stg` |
| `BULK INSERT` | T-SQL; SQL Server opens and reads the file itself | Very fast; pure SQL; `FORMAT = 'CSV'` handles quotes (SQL Server 2017+) | SQL Server needs file access; the path cannot be a variable; odd layouts need a format file | `ADMINISTER BULK OPERATIONS` + `INSERT`; file read as described in section 3 |
| `bcp.exe` | Command-line tool; the client reads the file | Fast, easy to script | Does not understand quoted fields | NTFS Read; `INSERT` |
| dbatools `Import-DbaCsv` | Community PowerShell module; one command per file | The shortest path for clean CSVs | Copy it to the PCN offline; it is open-source community code, so your software approval applies | NTFS Read; `INSERT` |
| SSIS | Microsoft's ETL tool; packages built in Visual Studio | Complex flows, Excel sources, redirecting bad rows | Not in Express; heavy for staging. Fits Level 3 | Runs as a SQL Agent proxy account |
| `OPENROWSET(BULK …)` | T-SQL reads a whole file as one value | Whole JSON or XML files | Same file-access rules as BULK INSERT | `ADMINISTER BULK OPERATIONS` |

### If you prefer BULK INSERT

The staging table has housekeeping columns the file does not, so BULK INSERT loads into a view shaped exactly like the file. A procedure then copies the rows across with the batch ID.

```sql
-- Load table shaped like the file, plus a line counter SQL Server fills in
CREATE TABLE stg.gc01_load (
    line_id     INT IDENTITY(1,1) NOT NULL,
    sample_time NVARCHAR(50) NULL, stream NVARCHAR(50) NULL, component NVARCHAR(100) NULL,
    value       NVARCHAR(50) NULL, unit   NVARCHAR(20) NULL, status    NVARCHAR(20)  NULL
);
GO
-- BULK INSERT targets the view, so the file's columns line up and line_id fills itself
CREATE VIEW stg.v_gc01_load AS
SELECT sample_time, stream, component, value, unit, status FROM stg.gc01_load;
GO
CREATE OR ALTER PROCEDURE etl.usp_BulkLoad_gc01 @batch_id INT, @file_path NVARCHAR(400)
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;
    IF @file_path NOT LIKE N'D:\OTData\work\%' OR @file_path LIKE N'%..%'
        THROW 50010, N'File must be in D:\OTData\work.', 1;

    -- BULK INSERT will not take a variable for the path, so the statement is built as text.
    -- Doubling single quotes keeps the path a plain string. (QUOTENAME stops at 128 characters.)
    -- ROWTERMINATOR '\n' means CRLF to BULK INSERT; use '0x0a' for files with Unix line endings.
    DECLARE @sql NVARCHAR(MAX) =
        N'BULK INSERT stg.v_gc01_load FROM N''' + REPLACE(@file_path, N'''', N'''''') + N'''
          WITH (FORMAT = ''CSV'', FIRSTROW = 2, FIELDTERMINATOR = '','', ROWTERMINATOR = ''\n'',
                CODEPAGE = ''65001'', TABLOCK);';

    BEGIN TRAN;
        DELETE FROM stg.gc01_load;          -- DELETE, not TRUNCATE: TRUNCATE needs ALTER on the table
        EXEC sys.sp_executesql @sql;
        INSERT stg.gc01_result (batch_id, row_num, sample_time, stream, component, value, unit, status)
        SELECT @batch_id, ROW_NUMBER() OVER (ORDER BY line_id),   -- record number: 1 = first data row
               sample_time, stream, component, value, unit, status
        FROM stg.gc01_load;
    COMMIT;
END;
GO
```

Because the statement is dynamic SQL, the caller needs `INSERT` on the view as well as `ADMINISTER BULK OPERATIONS`; ownership chaining does not cover it. Add `MAXERRORS` and `ERRORFILE` if you want bad rows written to a side file instead of failing the whole file. SQL Server writes that file as its service account.

## 9. The loader script

One script, `D:\OTData\scripts\Load-StagingFiles.ps1`, loads every waiting file for every active source. It reads its instructions from `etl.Source`, so adding a feed means adding a row and a table, not editing the script.

### What one run does

1. **Lock.** It holds an exclusive lock file, so a second copy started by hand exits at once.
2. **Recover.** It marks batches a crashed run left in `Loading` as failed, and moves files left in `work\` back to `inbound\`.
3. **List.** For each active source it lists files matching `file_pattern` that have not changed for 60 seconds, oldest first.
4. **Claim.** It moves each file into `work\`. A file the writer still has open cannot be moved, so it waits for the next run.
5. **Register.** It takes the file's SHA-256 hash and calls `etl.usp_BatchStart`, which returns a batch ID or says the file is a duplicate.
6. **Load.** In one transaction it parses the file, sends rows in chunks of 50,000, writes unreadable lines to `etl.RejectRow`, and calls `etl.usp_BatchEnd`.
7. **File.** It moves the file to `archive\` on success or `reject\` on failure, with the batch ID in its name.

Rerunning is always safe. Rows and the batch status commit together or not at all, an identical file is recognised by its hash, and anything a crash leaves behind is cleaned up by the next run.

### The script

```powershell
<#
  Load-StagingFiles.ps1
  Loads every waiting file for every active source into OT_Staging.
  Runs on the comm server as PCN\svc_sqlload, from Task Scheduler, after Collect-SourceFiles.ps1.
  Needs only Windows PowerShell 5.1 and .NET Framework, both built into Windows Server.
#>
[CmdletBinding()]
param(
    [string]$SqlInstance   = 'localhost\OTSTAGE',
    [string]$Database      = 'OT_Staging',
    [string]$Root          = 'D:\OTData',
    [int]   $StableSeconds = 60,      # skip files changed in the last minute: still being written
    [int]   $ChunkRows     = 50000    # rows sent to SQL per round trip
)
$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName Microsoft.VisualBasic
$ConnStr      = "Server=$SqlInstance;Database=$Database;Integrated Security=SSPI;Application Name=OT Loader"
$LogFile      = Join-Path $Root ('logs\loader_{0:yyyyMMdd}.log' -f (Get-Date))
$Housekeeping = @('batch_id', 'row_num', 'loaded_utc')

function Write-Log([string]$Message) {
    '{0:yyyy-MM-dd HH:mm:ss} {1}' -f (Get-Date), $Message | Add-Content -Path $LogFile
}

function Limit([string]$Text, [int]$Max = 4000) {
    if ($null -eq $Text) { return $null }
    if ($Text.Length -gt $Max) { return $Text.Substring(0, $Max) } else { return $Text }
}

# Runs a query and returns its first result set.
function Invoke-SqlQuery([string]$Sql, [hashtable]$Params = @{}) {
    $cn = New-Object System.Data.SqlClient.SqlConnection $ConnStr
    try {
        $cmd = $cn.CreateCommand(); $cmd.CommandText = $Sql; $cmd.CommandTimeout = 300
        foreach ($k in $Params.Keys) { [void]$cmd.Parameters.AddWithValue($k, $Params[$k]) }
        $cn.Open()
        $t = New-Object System.Data.DataTable
        $t.Load($cmd.ExecuteReader())
        return ,$t
    } finally { $cn.Dispose() }
}

# Runs a statement that returns nothing.
function Invoke-SqlExec([string]$Sql, [hashtable]$Params = @{}) {
    $cn = New-Object System.Data.SqlClient.SqlConnection $ConnStr
    try {
        $cmd = $cn.CreateCommand(); $cmd.CommandText = $Sql; $cmd.CommandTimeout = 300
        foreach ($k in $Params.Keys) { [void]$cmd.Parameters.AddWithValue($k, $Params[$k]) }
        $cn.Open(); [void]$cmd.ExecuteNonQuery()
    } finally { $cn.Dispose() }
}

# Marks the batch Loaded inside the load transaction.
function Invoke-BatchEnd($Cn, $Tx, [int]$BatchId, [int]$Loaded, [int]$Rejected) {
    $cmd = $Cn.CreateCommand(); $cmd.Transaction = $Tx
    $cmd.CommandType = [System.Data.CommandType]::StoredProcedure
    $cmd.CommandText = 'etl.usp_BatchEnd'
    [void]$cmd.Parameters.AddWithValue('@batch_id', $BatchId)
    [void]$cmd.Parameters.AddWithValue('@rows_loaded', $Loaded)
    [void]$cmd.Parameters.AddWithValue('@rows_rejected', $Rejected)
    [void]$cmd.ExecuteNonQuery()
}

# The staging table's file columns, in order (all columns except the housekeeping ones).
function Get-FileColumns($Cn, $Tx, [string]$Table) {
    $cmd = $Cn.CreateCommand(); $cmd.Transaction = $Tx
    $cmd.CommandText = 'SELECT name FROM sys.columns WHERE object_id = OBJECT_ID(@t) ORDER BY column_id'
    [void]$cmd.Parameters.AddWithValue('@t', $Table)
    $r = $cmd.ExecuteReader()
    $cols = New-Object System.Collections.Generic.List[string]
    while ($r.Read()) { if ($Housekeeping -notcontains $r.GetString(0)) { $cols.Add($r.GetString(0)) } }
    $r.Close()
    if ($cols.Count -eq 0) { throw "Table $Table not found, or it has no file columns." }
    return ,$cols
}

# Delimited or fixed-width file -> staging table. Rows, rejects and status commit together.
function Import-TextFile($Src, [string]$Path, [int]$BatchId) {
    $cn = New-Object System.Data.SqlClient.SqlConnection $ConnStr
    $cn.Open(); $tx = $cn.BeginTransaction(); $parser = $null
    try {
        $cols = Get-FileColumns $cn $tx ([string]$Src.target_table)

        $rows = New-Object System.Data.DataTable
        [void]$rows.Columns.Add('batch_id', [int]); [void]$rows.Columns.Add('row_num', [int])
        foreach ($c in $cols) { [void]$rows.Columns.Add($c, [string]) }
        $rejects = New-Object System.Data.DataTable
        [void]$rejects.Columns.Add('batch_id', [int]); [void]$rejects.Columns.Add('row_num', [int])
        [void]$rejects.Columns.Add('reason', [string]); [void]$rejects.Columns.Add('raw_line', [string])

        $bulk = New-Object System.Data.SqlClient.SqlBulkCopy($cn, [System.Data.SqlClient.SqlBulkCopyOptions]::Default, $tx)
        $bulk.DestinationTableName = [string]$Src.target_table
        $bulk.BulkCopyTimeout = 0
        foreach ($c in $rows.Columns) { [void]$bulk.ColumnMappings.Add($c.ColumnName, $c.ColumnName) }

        $parser = New-Object Microsoft.VisualBasic.FileIO.TextFieldParser($Path, [System.Text.Encoding]::GetEncoding([int]$Src.code_page))
        if ($Src.file_format -eq 'fixed') {
            $parser.TextFieldType = [Microsoft.VisualBasic.FileIO.FieldType]::FixedWidth
            $parser.SetFieldWidths([int[]](([string]$Src.field_widths) -split ','))
        } else {
            $parser.TextFieldType = [Microsoft.VisualBasic.FileIO.FieldType]::Delimited
            $parser.SetDelimiters([string]$Src.delimiter)
            $parser.HasFieldsEnclosedInQuotes = $true
        }
        $parser.TrimWhiteSpace = $true

        if ($Src.has_header) {
            $header = $parser.ReadFields()
            if ($header.Count -ne $cols.Count) {
                throw "Header has $($header.Count) fields; $($Src.target_table) expects $($cols.Count)."
            }
        }

        $rec = 0; $loaded = 0; $rejected = 0
        while (-not $parser.EndOfData) {
            $rec++
            try { $fields = $parser.ReadFields() }
            catch [Microsoft.VisualBasic.FileIO.MalformedLineException] {
                $rejected++
                [void]$rejects.Rows.Add($BatchId, $rec, 'Unreadable line (unbalanced quotes or short fixed-width line)', (Limit $parser.ErrorLine))
                continue
            }
            if ($fields.Count -ne $cols.Count) {
                $rejected++
                [void]$rejects.Rows.Add($BatchId, $rec, "Expected $($cols.Count) fields, found $($fields.Count)", (Limit ($fields -join [string]$Src.delimiter)))
                continue
            }
            $row = $rows.NewRow()
            $row['batch_id'] = $BatchId; $row['row_num'] = $rec
            for ($i = 0; $i -lt $cols.Count; $i++) {
                $row[$cols[$i]] = if ($fields[$i] -eq '') { [DBNull]::Value } else { $fields[$i] }
            }
            $rows.Rows.Add($row); $loaded++
            if ($rows.Rows.Count -ge $ChunkRows) { $bulk.WriteToServer($rows); $rows.Clear() }
        }
        if ($rows.Rows.Count -gt 0) { $bulk.WriteToServer($rows) }
        if ($loaded -eq 0 -and $rejected -gt 0) { throw "No readable rows ($rejected rejected)." }

        if ($rejects.Rows.Count -gt 0) {
            $rb = New-Object System.Data.SqlClient.SqlBulkCopy($cn, [System.Data.SqlClient.SqlBulkCopyOptions]::Default, $tx)
            $rb.DestinationTableName = 'etl.RejectRow'
            foreach ($c in $rejects.Columns) { [void]$rb.ColumnMappings.Add($c.ColumnName, $c.ColumnName) }
            $rb.WriteToServer($rejects)
        }
        Invoke-BatchEnd $cn $tx $BatchId $loaded $rejected
        $tx.Commit()
        return "$loaded rows, $rejected rejected"
    } catch {
        if ($tx.Connection) { $tx.Rollback() }
        throw
    } finally {
        if ($parser) { $parser.Close() }
        $cn.Dispose()
    }
}

# Every line as one row (report-style and key=value files; parsed later in SQL). row_num = line number.
function Import-Lines($Src, [string]$Path, [int]$BatchId) {
    $cn = New-Object System.Data.SqlClient.SqlConnection $ConnStr
    $cn.Open(); $tx = $cn.BeginTransaction()
    try {
        $rows = New-Object System.Data.DataTable
        [void]$rows.Columns.Add('batch_id', [int]); [void]$rows.Columns.Add('row_num', [int]); [void]$rows.Columns.Add('line', [string])
        $bulk = New-Object System.Data.SqlClient.SqlBulkCopy($cn, [System.Data.SqlClient.SqlBulkCopyOptions]::Default, $tx)
        $bulk.DestinationTableName = [string]$Src.target_table; $bulk.BulkCopyTimeout = 0
        foreach ($c in 'batch_id', 'row_num', 'line') { [void]$bulk.ColumnMappings.Add($c, $c) }
        $n = 0
        foreach ($line in [System.IO.File]::ReadLines($Path, [System.Text.Encoding]::GetEncoding([int]$Src.code_page))) {
            $n++
            [void]$rows.Rows.Add($BatchId, $n, (Limit $line))
            if ($rows.Rows.Count -ge $ChunkRows) { $bulk.WriteToServer($rows); $rows.Clear() }
        }
        if ($rows.Rows.Count -gt 0) { $bulk.WriteToServer($rows) }
        Invoke-BatchEnd $cn $tx $BatchId $n 0
        $tx.Commit()
        return "$n lines"
    } catch { if ($tx.Connection) { $tx.Rollback() }; throw } finally { $cn.Dispose() }
}

# Whole JSON or XML file -> the source's own procedure, which shreds it into rows (section 10).
function Import-Document($Src, [string]$Path, [int]$BatchId) {
    $text = [System.IO.File]::ReadAllText($Path, [System.Text.Encoding]::GetEncoding([int]$Src.code_page))
    # SQL Server refuses an XML encoding declaration inside Unicode text, so drop it
    if ($Src.file_format -eq 'xml') { $text = $text -replace '^\s*<\?xml[^>]*\?>', '' }
    $cn = New-Object System.Data.SqlClient.SqlConnection $ConnStr
    $cn.Open(); $tx = $cn.BeginTransaction()
    try {
        $cmd = $cn.CreateCommand(); $cmd.Transaction = $tx; $cmd.CommandTimeout = 0
        $cmd.CommandType = [System.Data.CommandType]::StoredProcedure
        $cmd.CommandText = [string]$Src.load_proc
        [void]$cmd.Parameters.AddWithValue('@batch_id', $BatchId)
        $p = $cmd.Parameters.Add('@doc', [System.Data.SqlDbType]::NVarChar, -1); $p.Value = $text
        $loaded = [int]$cmd.ExecuteScalar()     # the procedure ends by selecting its row count
        Invoke-BatchEnd $cn $tx $BatchId $loaded 0
        $tx.Commit()
        return "$loaded rows"
    } catch { if ($tx.Connection) { $tx.Rollback() }; throw } finally { $cn.Dispose() }
}

# ---- Main -------------------------------------------------------------------------------
try { $lock = [System.IO.File]::Open((Join-Path $Root 'logs\loader.lock'), 'OpenOrCreate', 'ReadWrite', 'None') }
catch { Write-Log 'Another run is still active; exiting.'; return }

try {
    Invoke-SqlExec 'EXEC etl.usp_CleanupStale;'

    # Files left in work\ by a run that died go back to inbound\ for another try.
    Get-ChildItem -LiteralPath (Join-Path $Root 'work') -File | ForEach-Object {
        $code, $name = $_.Name -split '__', 2
        Move-Item -LiteralPath $_.FullName -Destination (Join-Path $Root "inbound\$code\$name") -Force
    }

    $sources = Invoke-SqlQuery 'SELECT * FROM etl.Source WHERE is_active = 1 ORDER BY source_code;'
    foreach ($src in $sources.Rows) {
        $inbound = Join-Path $Root "inbound\$($src.source_code)"
        if (-not (Test-Path -LiteralPath $inbound)) { continue }
        $cutoff = (Get-Date).AddSeconds(-$StableSeconds)
        $files = Get-ChildItem -LiteralPath $inbound -Filter ([string]$src.file_pattern) -File |
                 Where-Object { $_.LastWriteTime -lt $cutoff } | Sort-Object LastWriteTime

        foreach ($f in $files) {
            # Claim: move into work\ (same volume, so an instant rename). Locked files wait.
            $work = Join-Path $Root ('work\{0}__{1}' -f $src.source_code, $f.Name)
            try { Move-Item -LiteralPath $f.FullName -Destination $work }
            catch { Write-Log "$($src.source_code) $($f.Name): still in use; next run."; continue }

            # Register: fingerprint the file and get a batch ID, or learn it is a duplicate.
            $hash  = (Get-FileHash -LiteralPath $work -Algorithm SHA256).Hash
            $start = Invoke-SqlQuery 'EXEC etl.usp_BatchStart @source_code = @s, @file_name = @n, @file_hash = @h, @file_bytes = @b, @file_time_utc = @t;' @{
                '@s' = [string]$src.source_code; '@n' = $f.Name; '@h' = $hash; '@b' = [long]$f.Length; '@t' = $f.LastWriteTimeUtc }
            $batchId = [int]$start.Rows[0].batch_id
            $archive = Join-Path $Root ('archive\{0}\{1:yyyy}\{1:MM}' -f $src.source_code, (Get-Date))
            New-Item -ItemType Directory -Path $archive -Force | Out-Null

            if ($start.Rows[0].status -eq 'Duplicate') {
                Move-Item -LiteralPath $work -Destination (Join-Path $archive ('{0}_dup_{1}' -f $batchId, $f.Name))
                Write-Log "$($src.source_code) $($f.Name): same as an earlier load; skipped (batch $batchId)."
                continue
            }

            try {
                $result = switch ([string]$src.file_format) {
                    'delimited' { Import-TextFile $src $work $batchId }
                    'fixed'     { Import-TextFile $src $work $batchId }
                    'lines'     { Import-Lines    $src $work $batchId }
                    'json'      { Import-Document $src $work $batchId }
                    'xml'       { Import-Document $src $work $batchId }
                }
                Move-Item -LiteralPath $work -Destination (Join-Path $archive ('{0}_{1}' -f $batchId, $f.Name))
                Write-Log "$($src.source_code) $($f.Name): batch $batchId loaded, $result."
            } catch {
                $msg = $_.Exception.Message
                Invoke-SqlExec 'EXEC etl.usp_BatchFail @batch_id = @id, @error = @e;' @{ '@id' = $batchId; '@e' = (Limit $msg 2000) }
                $rej = Join-Path $Root "reject\$($src.source_code)"
                New-Item -ItemType Directory -Path $rej -Force | Out-Null
                Move-Item -LiteralPath $work -Destination (Join-Path $rej ('{0}_{1}' -f $batchId, $f.Name))
                Write-Log "$($src.source_code) $($f.Name): batch $batchId FAILED: $msg"
            }
        }
    }
} catch {
    Write-Log "Run stopped: $($_.Exception.Message)"   # e.g. SQL Server unreachable; files stay put for next run
    throw
} finally {
    $lock.Dispose()
}
```

Save it with plain ASCII characters only: Windows PowerShell 5.1 misreads UTF-8 scripts that have no byte-order mark. Source codes must not contain a double underscore, because the script uses `__` to remember which source a file in `work\` came from.

### Test it

The service account is denied interactive logon, so `runas` will not work. Test through the scheduled task instead (section 11), then check the results:

```powershell
Start-ScheduledTask -TaskName 'OT Staging Loader'
Get-Content "D:\OTData\logs\loader_$(Get-Date -f yyyyMMdd).log" -Tail 20
```

```sql
SELECT TOP (20) * FROM etl.Batch ORDER BY batch_id DESC;
SELECT TOP (50) * FROM etl.RejectRow ORDER BY batch_id DESC, row_num;
SELECT TOP (50) * FROM stg.gc01_result ORDER BY batch_id DESC, row_num;
```

## 10. Other file shapes

Fixed width loads straight through the script. Report-style and key=value files land as raw lines and are cut up in SQL. JSON and XML go to a small procedure per source. Excel is converted to CSV first, and binary files need the vendor's tool.

### Fixed width

When every line is data, set `file_format = 'fixed'` and give the widths in characters. `-1` as the last width means "the rest of the line".

```sql
INSERT etl.Source (source_code, description, file_pattern, file_format, field_widths,
                   has_header, code_page, target_table, load_pattern)
VALUES ('gc02', N'GC-02 fixed-width results', N'GC02_*.dat', 'fixed', '10,19,12,-1',
        0, 1252, N'stg.gc02_result', 'append');
```

If title lines, page headers or totals are mixed in, load the file as lines and cut columns in SQL instead. Every lines table has the same shape:

```sql
CREATE TABLE stg.gc02_lines (
    batch_id   INT            NOT NULL,
    row_num    INT            NOT NULL,     -- line number in the file
    loaded_utc DATETIME2(0)   NOT NULL CONSTRAINT DF_gc02_lines_loaded DEFAULT SYSUTCDATETIME(),
    line       NVARCHAR(4000) NULL,
    CONSTRAINT PK_gc02_lines PRIMARY KEY (batch_id, row_num)
);
GO
-- Column positions come from the vendor's manual, or a column ruler in Notepad++
CREATE OR ALTER VIEW stg.v_gc02_parsed AS
SELECT batch_id, row_num,
       TRIM(SUBSTRING(line,  1, 10)) AS sample_id,
       TRIM(SUBSTRING(line, 11, 19)) AS sample_time,
       TRIM(SUBSTRING(line, 30, 12)) AS component,
       TRIM(SUBSTRING(line, 42, 12)) AS value
FROM stg.gc02_lines
WHERE line LIKE '[0-9]%';      -- data lines start with a digit; titles and totals do not
```

### Report-style text

A printed report puts a header line above a block of values:

```text
GC-01 ANALYSIS REPORT            2026-10-02 14:00
Sample: S-1042   Stream: 3
  Methane          92.41 mol%
  Ethane            4.12 mol%
Sample: S-1043   Stream: 1
  Methane          91.87 mol%
```

Load it as lines, then carry each `Sample:` line down to the value lines under it with a running maximum:

```sql
CREATE OR ALTER VIEW stg.v_gc01_report_parsed AS
WITH marked AS (
    SELECT batch_id, row_num, line,
           MAX(CASE WHEN line LIKE 'Sample:%' THEN row_num END)
               OVER (PARTITION BY batch_id ORDER BY row_num ROWS UNBOUNDED PRECEDING) AS header_row
    FROM stg.gc01_report_lines
)
SELECT d.batch_id, d.row_num,
       TRIM(SUBSTRING(h.line,  9, 10)) AS sample_id,    -- 'Sample: S-1042' -> 'S-1042'
       TRIM(SUBSTRING(d.line,  3, 14)) AS component,
       TRIM(SUBSTRING(d.line, 17,  8)) AS value,
       TRIM(SUBSTRING(d.line, 26, 10)) AS unit
FROM marked AS d
JOIN stg.gc01_report_lines AS h ON h.batch_id = d.batch_id AND h.row_num = d.header_row
WHERE d.line LIKE '  [A-Za-z]%';                        -- value lines are indented two spaces
```

This works, but it breaks whenever the vendor changes the report layout. Where the analyzer offers raw output, load that instead.

### Key=value

```text
Analyzer=AT-301
Timestamp=2026-10-02T14:00:00
H2S_ppm=3.8
Status=OK
```

Load as lines, split at the first `=`, then turn the keys into columns. Here one file is one record:

```sql
SELECT batch_id,
       MAX(CASE WHEN k = 'Analyzer'  THEN v END) AS analyzer,
       MAX(CASE WHEN k = 'Timestamp' THEN v END) AS sample_time,
       MAX(CASE WHEN k = 'H2S_ppm'   THEN v END) AS h2s_ppm,
       MAX(CASE WHEN k = 'Status'    THEN v END) AS status
FROM (SELECT batch_id,
             TRIM(LEFT(line, CHARINDEX('=', line) - 1))            AS k,
             TRIM(SUBSTRING(line, CHARINDEX('=', line) + 1, 4000)) AS v
      FROM stg.at301_lines
      WHERE CHARINDEX('=', line) > 1) AS kv
GROUP BY batch_id;
```

### JSON

The script reads the whole file and passes it to the source's `load_proc`. Register it with `file_format = 'json'` and `load_proc = 'etl.usp_Load_lab_json'`.

```json
{ "results": [ { "sampleId": "L-2201", "analyte": "Sulfur", "value": 12.4, "unit": "ppm", "time": "2026-10-02T09:15:00" } ] }
```

```sql
CREATE TABLE stg.lab_result (
    batch_id    INT           NOT NULL,
    row_num     INT           NOT NULL,
    loaded_utc  DATETIME2(0)  NOT NULL CONSTRAINT DF_lab_result_loaded DEFAULT SYSUTCDATETIME(),
    sample_id   NVARCHAR(50)  NULL,
    analyte     NVARCHAR(100) NULL,
    value       NVARCHAR(50)  NULL,
    unit        NVARCHAR(20)  NULL,
    sample_time NVARCHAR(50)  NULL,
    CONSTRAINT PK_lab_result PRIMARY KEY (batch_id, row_num)
);
GO
CREATE OR ALTER PROCEDURE etl.usp_Load_lab_json @batch_id INT, @doc NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;
    IF ISJSON(@doc) = 0 THROW 50020, N'File is not valid JSON.', 1;

    INSERT stg.lab_result (batch_id, row_num, sample_id, analyte, value, unit, sample_time)
    SELECT @batch_id, CAST(a.[key] AS INT) + 1, j.sample_id, j.analyte, j.value, j.unit, j.sample_time
    FROM OPENJSON(@doc, '$.results') AS a
    CROSS APPLY OPENJSON(a.value)
         WITH (sample_id   NVARCHAR(50)  '$.sampleId',
               analyte     NVARCHAR(100) '$.analyte',
               value       NVARCHAR(50)  '$.value',
               unit        NVARCHAR(20)  '$.unit',
               sample_time NVARCHAR(50)  '$.time') AS j;

    SELECT @@ROWCOUNT;     -- the script reads this as rows_loaded
END;
GO
```

`OPENJSON` needs database compatibility level 130 or higher. A new database on SQL Server 2022 is at 160.

### XML

```xml
<Results>
  <Result tag="AT-301.H2S"><Value>3.8</Value><Time>2026-10-02T14:00:00</Time></Result>
</Results>
```

```sql
CREATE OR ALTER PROCEDURE etl.usp_Load_at_xml @batch_id INT, @doc XML
AS
BEGIN
    SET NOCOUNT ON;
    INSERT stg.at_result (batch_id, row_num, tag, value, sample_time)
    SELECT @batch_id,
           ROW_NUMBER() OVER (ORDER BY (SELECT NULL)),
           r.value('@tag', 'nvarchar(100)'),
           r.value('(Value/text())[1]', 'nvarchar(50)'),
           r.value('(Time/text())[1]', 'nvarchar(50)')
    FROM @doc.nodes('/Results/Result') AS t(r);

    SELECT @@ROWCOUNT;
END;
GO
```

The script strips the `<?xml … encoding="UTF-8"?>` line before sending, because SQL Server refuses an encoding declaration inside Unicode text.

### Excel

Best is to have the source save CSV. If it can only produce workbooks, convert them before the loader runs with the ImportExcel PowerShell module, which reads .xlsx without Excel installed. Copy it to the PCN offline: on a connected machine run `Save-Module ImportExcel -Path C:\temp\mods`, then copy the folder into `C:\Program Files\WindowsPowerShell\Modules` on the comm server.

```powershell
Import-Module ImportExcel
Get-ChildItem 'D:\OTData\inbound\abb_objects_xlsx' -Filter *.xlsx | ForEach-Object {
    $csv = Join-Path 'D:\OTData\inbound\abb_objects' ($_.BaseName + '.csv')
    Import-Excel -Path $_.FullName -WorksheetName 'Objects' |
        Export-Csv -Path $csv -NoTypeInformation -Encoding UTF8
    Move-Item -LiteralPath $_.FullName -Destination 'D:\OTData\archive\abb_objects_xlsx\'
}
```

Avoid the Excel OLE DB driver and `OPENROWSET` against workbooks on a server. Its 32-bit and 64-bit versions clash, and Microsoft does not support it inside server processes.

### Binary

If Notepad shows gibberish, the file is in the vendor's own format. Use the vendor's export utility (many are command-line and can run as a step before collection), the vendor's API or OPC interface, or ask for a CSV or report output. Do not try to decode it yourself.

## 11. Scheduling and automation

Two scheduled tasks on the comm server run everything on the PCN: collect-then-load every 5 minutes, and nightly housekeeping. Both run as `svc_sqlload`. Level 3's SQL Agent does the rest.

### Run order

| When | Where | What | Scheduled by |
| --- | --- | --- | --- |
| Every 5 minutes | Comm server | `Collect-SourceFiles.ps1`, then `Load-StagingFiles.ps1` | Task Scheduler, task "OT Staging Loader" |
| Nightly, 02:10 | Comm server | `Nightly-Maintenance.ps1`: purge, backups, archive trim (section 13) | Task Scheduler, task "OT Staging Nightly" |
| Every 15 minutes | Level 3 | Pull new batches, merge, reconcile, publish (section 12) | SQL Agent |
| Nightly | Level 3 | Backups, freshness check, alert email | SQL Agent and Database Mail |

### Create the loader task

```powershell
$ps      = 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe'
$collect = New-ScheduledTaskAction -Execute $ps -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\OTData\scripts\Collect-SourceFiles.ps1"'
$load    = New-ScheduledTaskAction -Execute $ps -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\OTData\scripts\Load-StagingFiles.ps1"'
$every5  = New-ScheduledTaskTrigger -Once -At (Get-Date).Date -RepetitionInterval (New-TimeSpan -Minutes 5)
$settings = New-ScheduledTaskSettingsSet -MultipleInstances IgnoreNew -StartWhenAvailable `
              -ExecutionTimeLimit (New-TimeSpan -Hours 1)

Register-ScheduledTask -TaskName 'OT Staging Loader' -Action $collect, $load -Trigger $every5 `
  -Settings $settings -User 'PCN\svc_sqlload' -Password (Read-Host 'svc_sqlload password') -RunLevel Limited
```

What the settings do:

- The two actions run in order on every trigger: collect, then load.
- `IgnoreNew` skips a trigger while the last run is still going, so runs never overlap.
- `StartWhenAvailable` runs a missed trigger after a reboot.
- `ExecutionTimeLimit` kills a hung run after an hour. The next run cleans up after it.
- `RunLevel Limited` runs without administrator rights, which the account should not have anyway.

If the trigger stops repeating after a day on an older Windows build, open the task's trigger and set "for a duration of" to Indefinitely.

Create "OT Staging Nightly" the same way, with one action for `Nightly-Maintenance.ps1` and `New-ScheduledTaskTrigger -Daily -At 02:10`.

### With a gMSA instead of a password

```powershell
$principal = New-ScheduledTaskPrincipal -UserId 'PCN\gmsa_sqlload$' -LogonType Password -RunLevel Limited
Register-ScheduledTask -TaskName 'OT Staging Loader' -Action $collect, $load -Trigger $every5 `
  -Settings $settings -Principal $principal
```

The comm server must be allowed to retrieve the gMSA's password; your domain admin sets that when creating it.

### Script signing

`-ExecutionPolicy Bypass` applies only to that PowerShell process. If a group policy enforces `AllSigned`, the policy wins: sign the scripts with your code-signing certificate and drop `Bypass`.

### If the PCN instance is Standard

SQL Agent can run the same scripts instead of Task Scheduler. Agent needs a proxy so the job step runs as `svc_sqlload`, not as the Agent service:

```sql
USE master;
CREATE CREDENTIAL cred_sqlload WITH IDENTITY = N'PCN\svc_sqlload', SECRET = N'<password>';
GO
USE msdb;
EXEC dbo.sp_add_proxy @proxy_name = N'proxy_sqlload', @credential_name = N'cred_sqlload', @enabled = 1;
EXEC dbo.sp_grant_proxy_to_subsystem @proxy_name = N'proxy_sqlload', @subsystem_name = N'CmdExec';
```

Each job step is then type Operating system (CmdExec), run as `proxy_sqlload`, with the same `powershell.exe -NoProfile -File …` command line. Agent adds job history and email alerts, which is its advantage over Task Scheduler.

## 12. What Level 3 does with it

The Level 3 SQL Server reads staging through a linked server, copies batches newer than the last one it took, and merges them into its own typed tables. Every connection starts on Level 3.

### IP21 and SQL Server on one box

InfoPlus.21 keeps its database in memory, and SQL Server takes all the memory it is allowed. Set SQL Server's `max server memory` so IP21 and Windows keep what they need under load: check IP21's working set in Task Manager at a busy time, and leave that plus a few GB. Keep `OT_Hub` apart from any databases Aspen installed.

### Linked server to the PCN

The certificate check needs the name the certificate was issued to. Level 3 probably cannot resolve PCN names, so add a hosts-file line on the Level 3 server: `<comm server IP>  commsrv.pcn.local`.

```sql
-- On the Level 3 SQL Server, as a sysadmin
EXEC master.dbo.sp_addlinkedserver
     @server     = N'PCN_STAGE',
     @srvproduct = N'',
     @provider   = N'MSOLEDBSQL',     -- or MSOLEDBSQL19: use the name under Server Objects > Linked Servers > Providers
     @datasrc    = N'commsrv.pcn.local,1433',
     @provstr    = N'Encrypt=yes;TrustServerCertificate=no';

-- Only the SQL Agent service account (and you, for testing) may use it
EXEC master.dbo.sp_addlinkedsrvlogin @rmtsrvname = N'PCN_STAGE', @useself = N'False',
     @locallogin = N'NT SERVICE\SQLSERVERAGENT', @rmtuser = N'l3_reader', @rmtpassword = N'<password>';
EXEC master.dbo.sp_serveroption N'PCN_STAGE', N'rpc out', N'false';
EXEC master.dbo.sp_serveroption N'PCN_STAGE', N'data access', N'true';

-- Test
SELECT TOP (5) * FROM PCN_STAGE.OT_Staging.etl.vw_LoadedBatches ORDER BY batch_id DESC;
```

The Agent account name above is for a default instance; a named instance uses `NT SERVICE\SQLAgent$<instance>`. Add a second `sp_addlinkedsrvlogin` for your own admin login so you can test by hand.

### Pull with a watermark

Level 3 remembers the last batch it pulled for each source and asks only for complete batches above it. Rows are copied into a temporary table first, so the transaction that saves them is purely local.

```sql
-- In OT_Hub on Level 3
CREATE TABLE etl.Watermark (
    source_code   VARCHAR(30)  NOT NULL CONSTRAINT PK_Watermark PRIMARY KEY,
    last_batch_id INT          NOT NULL CONSTRAINT DF_Watermark_last DEFAULT 0,
    pulled_utc    DATETIME2(0) NULL
);
INSERT etl.Watermark (source_code) VALUES ('gc01');
GO
CREATE OR ALTER PROCEDURE etl.usp_Pull_gc01
AS
BEGIN
    SET NOCOUNT ON; SET XACT_ABORT ON;
    DECLARE @from INT = (SELECT last_batch_id FROM etl.Watermark WHERE source_code = 'gc01');

    -- 1. Which complete batches are new?
    SELECT batch_id INTO #new
    FROM PCN_STAGE.OT_Staging.etl.vw_LoadedBatches
    WHERE source_code = 'gc01' AND batch_id > @from;
    IF NOT EXISTS (SELECT 1 FROM #new) RETURN;
    DECLARE @to INT = (SELECT MAX(batch_id) FROM #new);

    -- 2. Copy their rows across the firewall into a temp table
    SELECT r.batch_id, r.row_num, r.sample_time, r.stream, r.component, r.value, r.unit, r.status
    INTO #rows
    FROM PCN_STAGE.OT_Staging.stg.gc01_result AS r
    WHERE r.batch_id > @from AND r.batch_id <= @to;
    DELETE FROM #rows WHERE batch_id NOT IN (SELECT batch_id FROM #new);

    -- 3. Save rows and move the watermark together
    BEGIN TRAN;
        INSERT stg.gc01_result (batch_id, row_num, sample_time, stream, component, value, unit, status)
        SELECT batch_id, row_num, sample_time, stream, component, value, unit, status FROM #rows;
        UPDATE etl.Watermark SET last_batch_id = @to, pulled_utc = SYSUTCDATETIME()
        WHERE source_code = 'gc01';
    COMMIT;
END;
GO
```

### Merge into typed tables

The merge converts text to types with `TRY_CONVERT`, which returns NULL instead of an error. Rows that fail go to Level 3's reject table, and the rest are inserted or updated on the business key.

```sql
-- Rows whose text will not convert are set aside, not allowed to stop the merge
INSERT etl.RejectRow (batch_id, row_num, reason)
SELECT batch_id, row_num, N'Bad number or date'
FROM stg.gc01_result
WHERE batch_id > @last_merged          -- the merge keeps its own watermark, like the pull
  AND (TRY_CONVERT(DECIMAL(18,6), value) IS NULL
       OR TRY_CONVERT(DATETIME2(0), sample_time, 126) IS NULL);   -- 126 = ISO 8601
```

Schedule one SQL Agent job every 15 minutes with steps Pull, Merge, Reconcile and Publish, and have it email an operator through Database Mail on failure. The deck's Fig 17 shows the typed table the merge fills.

## 13. Backups, retention and monitoring

Staging can be rebuilt from the archived files, so it needs only light backups. The batch log is what you watch, and the alerts come from Level 3, which has Database Mail.

### What is kept, and for how long

| What | Kept | Removed by |
| --- | --- | --- |
| Staged rows | `retention_days` per source, default 14. Must outlast your longest Level 3 outage | `etl.usp_Purge`, nightly |
| Reject rows | 30 days | `etl.usp_Purge` |
| Batch log (`etl.Batch`) | Indefinitely; it is small | Nothing |
| Archived files | One year, or whatever your records policy says | `Nightly-Maintenance.ps1` |
| Script logs | 90 days | `Nightly-Maintenance.ps1` |
| Backups | Seven, one per weekday | Overwritten a week later |

### Nightly script

The backup statements need `svc_sqlload` to be a backup operator in `master` and `msdb` as well:

```sql
USE master; CREATE USER [PCN\svc_sqlload] FOR LOGIN [PCN\svc_sqlload]; ALTER ROLE db_backupoperator ADD MEMBER [PCN\svc_sqlload];
USE msdb;   CREATE USER [PCN\svc_sqlload] FOR LOGIN [PCN\svc_sqlload]; ALTER ROLE db_backupoperator ADD MEMBER [PCN\svc_sqlload];
```

```powershell
# Nightly-Maintenance.ps1 - purge, back up, trim. Runs as PCN\svc_sqlload at 02:10.
$ErrorActionPreference = 'Stop'
$Root = 'D:\OTData'; $Sql = 'localhost\OTSTAGE'; $Bak = 'D:\SQLBackup'
$log  = Join-Path $Root ('logs\nightly_{0:yyyyMMdd}.log' -f (Get-Date))
$day  = (Get-Date).DayOfWeek      # one file per weekday: seven kept, each overwritten a week later

function Run-Sql([string]$Query) {
    sqlcmd -S $Sql -E -b -Q $Query *>> $log
    if ($LASTEXITCODE -ne 0) { throw "sqlcmd failed: $Query" }
}

Run-Sql 'EXEC OT_Staging.etl.usp_Purge;'
Run-Sql "BACKUP DATABASE OT_Staging TO DISK = N'$Bak\OT_Staging_$day.bak' WITH INIT, CHECKSUM;"
Run-Sql "BACKUP DATABASE master     TO DISK = N'$Bak\master_$day.bak'     WITH INIT, CHECKSUM;"
Run-Sql "BACKUP DATABASE msdb       TO DISK = N'$Bak\msdb_$day.bak'       WITH INIT, CHECKSUM;"

Get-ChildItem "$Root\archive" -Recurse -File |
    Where-Object LastWriteTime -lt (Get-Date).AddDays(-365) | Remove-Item
Get-ChildItem "$Root\logs" -File -Filter '*.log' |
    Where-Object LastWriteTime -lt (Get-Date).AddDays(-90) | Remove-Item
```

SQL Server writes the .bak files as its own service account, which the installer gave rights to `D:\SQLBackup`. Express cannot compress backups, which is why there is no `COMPRESSION` option. Have your PCN backup system copy `D:\SQLBackup` elsewhere; a backup on the same disk does not survive the disk. `DBCC CHECKDB` needs `db_owner`, so run it monthly under an admin account rather than from this task.

### Monitoring queries on the PCN

```sql
USE OT_Staging;

-- Freshness: minutes since each source last loaded
SELECT s.source_code,
       MAX(b.ended_utc)                                    AS last_loaded_utc,
       DATEDIFF(MINUTE, MAX(b.ended_utc), SYSUTCDATETIME()) AS minutes_since
FROM etl.Source AS s
LEFT JOIN etl.Batch AS b ON b.source_code = s.source_code AND b.status = 'Loaded'
WHERE s.is_active = 1
GROUP BY s.source_code
ORDER BY minutes_since DESC;

-- Failures and rejects in the last 24 hours
SELECT batch_id, source_code, file_name, status, rows_loaded, rows_rejected, error_message, started_utc
FROM etl.Batch
WHERE started_utc > DATEADD(HOUR, -24, SYSUTCDATETIME())
  AND (status = 'Failed' OR rows_rejected > 0)
ORDER BY batch_id DESC;

-- Size against the Express limit of 10 GB per database
SELECT SUM(size) * 8 / 1024 AS data_mb FROM sys.database_files WHERE type_desc = 'ROWS';
```

### Alerts from Level 3

A failed or stopped loader shows up as a source going stale, which Level 3 can see through the linked server. An hourly Agent job on Level 3:

```sql
IF EXISTS (SELECT 1
           FROM PCN_STAGE.OT_Staging.etl.vw_LoadedBatches
           GROUP BY source_code
           HAVING MAX(ended_utc) < DATEADD(HOUR, -2, SYSUTCDATETIME()))
    EXEC msdb.dbo.sp_send_dbmail
         @profile_name = N'OT Alerts',
         @recipients   = N'<your address>',
         @subject      = N'PCN staging: a source has not loaded for 2 hours',
         @body         = N'Run the freshness query in OT_Staging to see which one.';
```

Set the two-hour threshold per source to suit how often it delivers. A daily file needs a threshold of a day or more.

## 14. Build checklist

Work through these in order; each step depends only on the ones above it. Tick them off as you go.

### Accounts and approvals

- [ ] Ask the PCN domain admin for `svc_sqlload` (or a gMSA) and the `PCN\SQL_Admins` group; deny the account interactive and Remote Desktop logon
- [ ] If the comm server is an 800xA node, confirm with ABB that a SQL Server instance is allowed on it
- [ ] Identify the SQL Server already on the comm server and who owns it

### Instance (section 4)

- [ ] Carry the installer and the latest cumulative update onto the PCN
- [ ] Install named instance `OTSTAGE`: engine only, mixed mode, admin group, D: directories, collation matching Level 3
- [ ] Apply the cumulative update
- [ ] Run the instance settings script and set login auditing
- [ ] Fixed port 1433, TCP/IP on, Named Pipes off, Browser disabled
- [ ] Certificate, private-key Read for the service account, Force Encryption, restart
- [ ] Firewall rule scoped to the Level 3 address
- [ ] Disable `sa` once you have signed in through the admin group

### Database (section 5)

- [ ] Create `OT_Staging`, its schemas, control tables, procedures and view
- [ ] Run the security script and put the `l3_reader` password in your password vault
- [ ] Add the collection columns to `etl.Source` (section 6)

### Folders and scripts (sections 6, 9 and 13)

- [ ] Create the `D:\OTData` folders and set their NTFS rights
- [ ] Copy the three scripts into `D:\OTData\scripts`, saved as plain ASCII
- [ ] Grant `svc_sqlload` Read on each source share

### First source

- [ ] Look inside a sample file (section 7) and write its data contract
- [ ] Create its staging table and its `etl.Source` row
- [ ] Put one file in `inbound\<source>` by hand

### Automation (section 11)

- [ ] Create the "OT Staging Loader" and "OT Staging Nightly" tasks
- [ ] Run the loader task on demand and check `etl.Batch`, `etl.RejectRow`, the staging table and the log
- [ ] Put the same file in again and confirm it is logged as Duplicate
- [ ] Put in a deliberately broken file and confirm it lands in `reject\` with a reason

### Level 3 (section 12)

- [ ] Import the comm server's certificate into Trusted Root and add the hosts-file line
- [ ] Create the linked server and its login mappings; run the test query
- [ ] Create the watermark, pull and merge procedures in `OT_Hub` and schedule the Agent job
- [ ] Set up Database Mail and the staleness alert
- [ ] Set `max server memory` to leave IP21 its room

### Handover

- [ ] Put the scripts and SQL in version control
- [ ] Record accounts, the vault location of passwords, ports and folders in the site documentation

## 15. Troubleshooting

Most failures come from three things: which identity reads a file, the certificate, and the file's own layout. Start with the error text, then `etl.Batch.error_message` and the day's loader log.

| Error or symptom | Likely cause | Fix |
| --- | --- | --- |
| Login failed for `l3_reader`: server is configured for Windows authentication only | Mixed mode is off | Server Properties > Security > SQL Server and Windows Authentication mode; restart the service |
| Level 3 times out connecting | Firewall, a dynamic port, or TCP/IP disabled | `Test-NetConnection <comm server> -Port 1433`; the fixed port in Configuration Manager; the firewall rule's address |
| "The certificate chain was issued by an authority that is not trusted" | Level 3 does not trust the comm server's certificate | Import the .cer into Trusted Root on Level 3 (section 4) |
| Certificate name mismatch | Connecting by IP, or by a name not on the certificate | Connect by the certificate's name, through the hosts-file line |
| SQL Server will not start after Force Encryption | The service account cannot read the certificate's private key | Grant it Read under Manage Private Keys |
| "Cannot bulk load because the file could not be opened. Operating system error code 5 (Access is denied.)" | BULK INSERT: the identity reading the file lacks NTFS Read (section 3) | Grant Read to the Windows login, or to the SQL service account for a SQL login; keep files local |
| "You do not have permission to use the bulk load statement" | No `ADMINISTER BULK OPERATIONS` | Grant it, or use the PowerShell loader |
| "Column is too long in the data file for row 1, column N" | Wrong row terminator: a file with Unix line endings read as Windows | `ROWTERMINATOR = '0x0a'` |
| "Received an invalid column length from the bcp client" | A value is longer than its staging column | Widen that `NVARCHAR` column |
| "Header has N fields; table expects M" | The file layout changed, or the staging table is wrong | Compare the header with the table; update both the table and the data contract |
| Accented letters garbled in staging | Wrong `code_page` | Check the file's first bytes (section 7); try 65001 or 1252 |
| Files sit in `inbound` and nothing loads | Task not running, file still changing, or `file_pattern` does not match | Task Scheduler history, the loader log, the pattern |
| Task's last result is a logon failure | Account lacks Log on as a batch job, or its password changed | Grant the right; update the task's stored password, or use a gMSA |
| "Unable to switch the encoding" on an XML load | An XML declaration naming an encoding, sent as Unicode text | The script strips it; remove it when testing by hand |
| `OPENJSON` not recognised | Database compatibility level below 130 | `ALTER DATABASE OT_Staging SET COMPATIBILITY_LEVEL = 160;` |
| "…would exceed your licensed limit of 10240 MB per database" | Express size limit reached | Shorten `retention_days`, run the purge, check the size query; consider Standard |
