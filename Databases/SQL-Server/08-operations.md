# 08 — Operations

[← Back to index](README.md) · [← Project Layout](07-project-layout.md) · [Next: Integration →](09-integration.md)

Keeping it alive. This is the chapter that matters at 2am.

---

## Contents

- [RPO and RTO — start here](#rpo-and-rto--start-here)
- [Recovery models](#recovery-models)
- [Backup strategy](#backup-strategy)
- [Restore testing](#restore-testing)
- [Instance configuration](#instance-configuration)
- [Maintenance](#maintenance)
- [Monitoring](#monitoring)
- [Common disasters](#common-disasters)
- [The minimum viable ops setup](#the-minimum-viable-ops-setup)

---

## RPO and RTO — start here

Two numbers that determine every other decision in this chapter.

**RPO — Recovery Point Objective.** How much data can you afford to lose, measured in time? If the answer is "15 minutes," your log backups run every 15 minutes.

**RTO — Recovery Time Objective.** How long can you afford to be down? This determines whether you need an Availability Group or whether restoring from backup is acceptable.

> **These are business decisions, not technical ones.** As the owner, you set them — nobody else will do it for you, and if you don't, you'll be making the decision under pressure during an outage with no time to think.

### Writing them down

| System | RPO | RTO | Implies |
|---|---|---|---|
| Master/reporting DB | 24h | 4h | Nightly full, `SIMPLE` recovery, restore from backup |
| Transactional app DB | 15 min | 1h | `FULL` recovery, log backups every 15 min |
| Raw landing zone | Re-extractable | 8h | `SIMPLE`, weekly full — it can be reloaded from source |

**Different databases legitimately get different answers.** If raw data can be re-pulled from the source system, its RPO is effectively "whatever it takes to re-extract" and you can be much more relaxed with it than with data that exists nowhere else.

That question — *"if I lost this entirely, could I get it back from somewhere?"* — is the one to ask about every database you own.

---

## Recovery models

| Model | Log behavior | Point-in-time restore | Use for |
|---|---|---|---|
| `SIMPLE` | Auto-truncates at checkpoint | ❌ | Dev, staging, anything rebuildable |
| `FULL` | Retained until log backup | ✅ | Production with real RPO needs |
| `BULK_LOGGED` | Minimally logs bulk operations | Partial | Temporarily, during huge loads |

```sql
SELECT name, recovery_model_desc, log_reuse_wait_desc
FROM   sys.databases WHERE database_id > 4;
```

### The most common way a small shop's SQL Server falls over

Someone sets `FULL` recovery (or inherits it from the `model` database, where it's the default) and **never configures log backups**.

The log grows. And grows. It cannot truncate because SQL Server is holding every transaction for a point-in-time restore that will never be taken. Eventually the disk fills and **the database stops accepting writes entirely.**

The `log_reuse_wait_desc` column tells you exactly why the log can't truncate:

| Value | Meaning |
|---|---|
| `NOTHING` | Healthy |
| `LOG_BACKUP` | **You're in FULL and haven't backed up the log** — this is the classic |
| `ACTIVE_TRANSACTION` | A long-running transaction is holding it open |
| `REPLICATION` / `AVAILABILITY_REPLICA` | Replication or AG hasn't caught up |

> **If you're in `FULL` recovery, you MUST back up the log on a schedule.** There is no exception. If you don't need point-in-time restore, switch to `SIMPLE` and the problem disappears.

### Never shrink the log as a routine practice

Shrinking a log file that will just grow back causes fragmentation and VLF sprawl, and it's a symptom of not fixing the actual problem. A one-time shrink after fixing a runaway growth event is fine. A scheduled shrink job is not.

---

## Backup strategy

### The three types

| Type | Contains | Restore needs |
|---|---|---|
| **Full** | Everything | Just itself |
| **Differential** | Changes since the last *full* | Last full + this diff |
| **Log** | Transactions since the last *log* backup | Full + diff + **every** log in sequence |

### A standard chain

```
Sunday 02:00   Full
Mon–Sat 02:00  Differential
Every 15 min   Log
```

To restore to Thursday 14:37: Sunday's full → Wednesday's diff → every log backup from then to 14:37.

**The log chain is a chain.** Miss one file and you can't restore past that point. Never delete log backups until the next full is verified.

### Commands

```sql
BACKUP DATABASE MyDataDb
TO DISK = 'D:\Backups\MyDataDb_Full.bak'
WITH COMPRESSION, CHECKSUM, INIT, STATS = 10;

BACKUP DATABASE MyDataDb
TO DISK = 'D:\Backups\MyDataDb_Diff.bak'
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM, INIT;

BACKUP LOG MyDataDb
TO DISK = 'D:\Backups\MyDataDb_Log.trn'
WITH COMPRESSION, CHECKSUM, INIT;
```

**`WITH CHECKSUM`** verifies page checksums while backing up — it catches corruption *at backup time* rather than during a restore you're depending on.

**`WITH COMPRESSION`** is typically 3–5× smaller and often *faster* (less I/O outweighs the CPU). Turn it on by default.

### Where backups go

**Not on the same disk as the database.** A disk failure that takes the database should not also take the backups.

The **3-2-1 rule**: 3 copies, 2 different media types, 1 offsite. For a small shop: local disk, a NAS or file server, and cloud/offsite. Ransomware makes the offsite copy non-optional — an attacker who reaches your server reaches your local backups too.

### Verify

```sql
RESTORE VERIFYONLY FROM DISK = 'D:\Backups\MyDataDb_Full.bak' WITH CHECKSUM;

-- backup history: when did each database last get backed up?
SELECT d.name,
       LastFull = MAX(CASE WHEN b.type = 'D' THEN b.backup_finish_date END),
       LastDiff = MAX(CASE WHEN b.type = 'I' THEN b.backup_finish_date END),
       LastLog  = MAX(CASE WHEN b.type = 'L' THEN b.backup_finish_date END)
FROM   sys.databases AS d
LEFT JOIN msdb.dbo.backupset AS b ON b.database_name = d.name
WHERE  d.database_id > 4
GROUP  BY d.name;
```

Run that last query today. If any production database shows NULL or a stale date, that's the most important thing on your plate.

---

## Restore testing

> **A backup you have never restored is not a backup.**

The number of organizations that discover their backups are unusable *during an outage* is not small. Common causes: the job was failing silently for months; the file was corrupt; nobody knew the encryption certificate wasn't backed up; the restore takes 9 hours and RTO was 2.

### Do this

1. **Monthly**, restore the production full backup to a different server or a differently-named database
2. Run `DBCC CHECKDB` on the restored copy
3. Query something and confirm it looks right
4. **Time it.** That number is your real RTO — not the one you hoped for.

```sql
RESTORE DATABASE MyDataDb_RestoreTest
FROM DISK = 'D:\Backups\MyDataDb_Full.bak'
WITH MOVE 'MyDataDb'     TO 'E:\Test\MyDataDb_Test.mdf',
     MOVE 'MyDataDb_log' TO 'E:\Test\MyDataDb_Test.ldf',
     REPLACE, STATS = 10;

DBCC CHECKDB('MyDataDb_RestoreTest') WITH NO_INFOMSGS;
DROP DATABASE MyDataDb_RestoreTest;
```

**Bonus:** restoring to a second server gives you a free reporting/testing copy, and it exercises the restore path regularly so it's familiar rather than terrifying.

### Also back up

- **The `master` and `msdb` databases** — msdb holds all your Agent jobs, schedules, and history. Losing it means rebuilding every job from memory.
- **Encryption certificates and keys** — if you use TDE and lose the certificate, **your backups are permanently unreadable.** Store them somewhere other than the server they protect.
- **SSIS packages / SSISDB**
- **Instance-level config**: logins (with SIDs), linked servers, Agent operators, alerts

---

## Instance configuration

### Set these on day one

```sql
-- Max server memory. Default is "everything", which starves the OS.
-- Rule of thumb: total RAM minus 4GB, minus 1GB per 8GB above 16GB.
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'max server memory (MB)', 24576; RECONFIGURE;

-- MAXDOP: for OLTP-ish workloads, number of cores in one NUMA node, max 8.
EXEC sp_configure 'max degree of parallelism', 4; RECONFIGURE;

-- Cost threshold for parallelism: default 5 is from 1997 hardware.
-- 50 is a much better starting point.
EXEC sp_configure 'cost threshold for parallelism', 50; RECONFIGURE;

-- Optimize for ad hoc workloads: stops single-use plans bloating the cache.
EXEC sp_configure 'optimize for ad hoc workloads', 1; RECONFIGURE;

-- Backup compression on by default
EXEC sp_configure 'backup compression default', 1; RECONFIGURE;
```

### Database-level

```sql
ALTER DATABASE MyDataDb SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
ALTER DATABASE MyDataDb SET QUERY_STORE = ON
    (OPERATION_MODE = READ_WRITE, QUERY_CAPTURE_MODE = AUTO);
ALTER DATABASE MyDataDb SET AUTO_CLOSE OFF;
ALTER DATABASE MyDataDb SET AUTO_SHRINK OFF;     -- never on. ever.
ALTER DATABASE MyDataDb SET PAGE_VERIFY CHECKSUM;
```

**`AUTO_SHRINK` should never be on.** It fragments indexes catastrophically, causes the file to grow again immediately, and the cycle repeats forever.

### File growth

Set explicit sizes in **MB, not percent**. Percentage growth on a large file means huge growth events that block writes while they happen.

Pre-size data and log files to their expected size plus headroom. Growing during production is pure latency.

**Enable Instant File Initialization** (grant the service account "Perform Volume Maintenance Tasks" in Local Security Policy). Data file growth and restores become dramatically faster. It's a checkbox in modern SQL Server installers.

### tempdb

- 4–8 data files, equally sized, same autogrowth
- Fastest storage available
- Pre-sized so it doesn't grow in production
- Sized to accommodate the version store if RCSI is on

---

## Maintenance

### What actually matters, in order

**1. Backups.** Everything else is optional by comparison.

**2. `DBCC CHECKDB` — weekly.**

```sql
DBCC CHECKDB('MyDataDb') WITH NO_INFOMSGS, ALL_ERRORMSGS;
```

Detects corruption. **Corruption discovered six months late, after every backup containing good data has aged out, is unrecoverable.** This is the check that saves you from unrecoverable data loss, and people skip it because it's invisible when everything is fine.

Run it against a restored copy if it's too heavy for production — that also tests your restores. Two birds.

**3. Statistics updates.** Usually more valuable than index rebuilds and far cheaper.

**4. Index maintenance.** Least important of the four. Modern SSD storage makes fragmentation matter much less than the internet suggests. Don't build a nightly rebuild job before you have a measured problem.

### Use Ola Hallengren's scripts

Free, battle-tested, the de facto standard: **ola.hallengren.com**

`DatabaseBackup`, `IndexOptimize`, `DatabaseIntegrityCheck` — they handle every edge case you'd otherwise discover the hard way, and they're better than anything you'd write. Install them and schedule them; this is genuinely the correct answer for backup and maintenance jobs.

### Cleanup jobs to have

```sql
-- Agent job history grows forever
EXEC msdb.dbo.sp_purge_jobhistory @oldest_date = DATEADD(day, -30, GETDATE());

-- backup history in msdb grows forever and slows the GUI badly
EXEC msdb.dbo.sp_delete_backuphistory @oldest_date = DATEADD(month, -3, GETDATE());

-- your own logs
DELETE FROM dbo.RefreshLog WHERE StartedUtc < DATEADD(month, -6, SYSUTCDATETIME());
```

---

## Monitoring

### Alert on these

| Condition | Why |
|---|---|
| Backup job failed | Obvious |
| **No successful backup in N hours** | Catches "the job succeeded but did nothing" |
| `DBCC CHECKDB` found errors | Corruption — act immediately |
| Disk space < 15% | The most common preventable outage |
| Log file > X% full | The FULL-recovery trap |
| Refresh job failed | Your data is now stale |
| **Refresh row count deviates from normal** | Catches silent partial loads |
| Blocking > 60 seconds | Something is stuck |
| Deadlock rate spike | An access pattern regressed |

> The two in bold are the ones people miss. **Alerting on "the job failed" is not enough — you must also alert on "the job hasn't succeeded recently."** A job that was accidentally disabled, or an Agent service that's stopped, produces no failures at all. It produces silence, and silence looks exactly like success.

### Configure Agent alerts for severity 19–25 and errors 823, 824, 825

```sql
EXEC msdb.dbo.sp_add_alert
     @name = N'Severity 024 - Fatal Error: Hardware Error',
     @message_id = 0, @severity = 24, @enabled = 1,
     @include_event_description_in = 1;
```

**Error 825** is "read-retry succeeded" — SQL Server had to retry an I/O and it worked the second time. It's a warning that your storage is failing, and it is *silent* unless you alert on it. It typically shows up weeks before a real corruption event.

Set up Database Mail and an operator so alerts actually reach you.

### Freshness, exposed

```sql
CREATE OR ALTER VIEW mart.vw_DataFreshness
AS
SELECT ProcName,
       DataYear,
       LastSuccess  = MAX(FinishedUtc),
       HoursSince   = DATEDIFF(hour, MAX(FinishedUtc), SYSUTCDATETIME())
FROM   dbo.RefreshLog
WHERE  Status = 'Success'
GROUP  BY ProcName, DataYear;
```

Put "Data as of <timestamp>" on every dashboard. It prevents an enormous amount of confusion, and it turns a silent staleness problem into something a user reports on day one.

---

## Common disasters

### Transaction log filled the disk

**Immediate:** back up the log (this truncates it). If you're out of space entirely, add a second log file on another drive temporarily.
**Root cause:** `FULL` recovery with no log backups, or a long-running open transaction.
**Fix properly:** schedule log backups, or switch to `SIMPLE` if you don't need point-in-time restore.

### Someone ran UPDATE without WHERE

**If in `FULL` recovery:** restore to a copy at a point in time just before, then copy the correct rows back. This is the scenario point-in-time restore exists for.
**If in `SIMPLE`:** restore the last full backup to a copy and recover what you can.
**Prevention:** RCSI doesn't help here; what helps is nobody having write access to prod for routine work, and this being why.

### Database marked SUSPECT

Usually corruption or an inaccessible log file. **Don't panic-run `REPAIR_ALLOW_DATA_LOSS`** — the name is accurate and it's a last resort.

Order of operations: check the error log for the actual cause → restore from backup if you have one → only then consider repair.

### Backup job "succeeded" but there are no backups

Check the destination path, and check that the job's *steps* succeeded rather than just the job. Alert on backup *age*, not just job failure. → [monitoring](#monitoring)

### Everything is slow, suddenly

Check, in this order: blocking chain → a plan regression (Query Store makes this obvious) → disk space → a maintenance job still running → statistics that just auto-updated → someone deployed something.

### Agent jobs all vanished

`msdb` was lost or restored from an old backup. Back up `msdb`. Also keep job definitions scripted in your repo — `Script Job As > CREATE To` on every job, committed to git, means you can rebuild in minutes rather than from memory.

---

## The minimum viable ops setup

If you do nothing else, do these ten things:

1. **Nightly full backups** to a different disk, with `CHECKSUM` and `COMPRESSION`
2. **Log backups every 15 minutes** if the database is in `FULL` recovery
3. **`SIMPLE` recovery** on anything that doesn't need point-in-time restore
4. **Weekly `DBCC CHECKDB`**
5. **Monthly restore test**, timed
6. **Ola Hallengren's scripts** for backup and maintenance
7. **Set `max server memory`** — do not leave it at the default
8. **Turn on RCSI and Query Store**
9. **Alert on: backup age, disk space, job failure, and severity 19–25 + errors 823/824/825**
10. **Write down your RPO and RTO**, and verify your setup actually meets them

Steps 1, 4, and 5 together prevent nearly every unrecoverable disaster. Step 10 is what makes the other nine defensible.

---

[← Back to index](README.md) · [Next: Integration →](09-integration.md)
