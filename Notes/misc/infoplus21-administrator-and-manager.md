# Aspen InfoPlus.21: Administrator and Manager

*A brief learning guide to how the system is organized and what runs it.*

---

## 1. The Foundation

Aspen InfoPlus.21 (IP.21) is a memory-resident real-time database. Everything in it is stored as a **record** — a tag, a data collection configuration, a history repository, a scheduled query, even the description of a running process. The entire database is loaded into RAM while the system is running.

Every record is an instance of a **definition record**, which acts as the schema. The definition says what fields the record has, what data types those fields are, and what repeat areas (table-like blocks of rows) it contains. A definition holds no data and performs no work by itself.

Nothing in the database executes on its own, either. Records are acted upon by **tasks** — operating system processes that scan the records bound to them and do the work: collect data, write history, run queries, save the database to disk.

That split is the reason there are two applications:

| Application | Purpose | Question it answers |
|---|---|---|
| **InfoPlus.21 Administrator** | Edits the contents of the database | *What is the system configured to do?* |
| **InfoPlus.21 Manager** | Controls the processes that act on the database | *Is the system currently doing it?* |

Configuration lives in Administrator. Execution lives in Manager. A perfectly configured record does nothing if its task is stopped, and a healthy running task moves no data if no records are bound to it.

---

## 2. InfoPlus.21 Administrator

Administrator presents the database as a browsable tree and lets you create, edit, and delete records. Changes are applied to the live in-memory database immediately — but they are changes to *data*, not to running processes.

### 2.1 Definition Records

This branch lists every definition (schema) in the system. Expanding a definition lists every **instance** of it — so expanding `IOLongTagGetDef` shows all of your actual Cim-IO get records. This is a browse view, not a configuration screen: you are looking at real records grouped by their type.

You rarely edit a definition itself. The only common reason is adding custom fields, and the standard practice is to copy an existing definition to a new one rather than modify a stock Aspen definition.

### 2.2 Building Blocks

The field-level and block-level pieces that definitions are assembled from — individual field descriptions with their data types and lengths, grouped into blocks that make up a definition's fixed and repeat areas.

Effectively read-only in day-to-day work. You only come here when building a custom definition from scratch.

### 2.3 Tag Records

The records you interact with most. The main historized types:

- **`IP_AnalogDef`** — continuous values. Value, input value, quality, engineering units, alarm limits, plus a repeat area holding trend/history values and the archiving settings (archiving on/off, target repository, compression significance, stepped/interpolated).
- **`IP_DiscreteDef`** — state values. Same structure, but the value resolves through a **selector record** that supplies the state text (`ON`/`OFF`, `RUNNING`/`STOPPED`, custom sets). Every discrete tag points at one; the wrong selector produces the wrong state strings.
- **`IP_TextDef`** — string values. Used for things like batch IDs, recipe names, and lot numbers.

Fixed variants (`IP_AnalogFixedDef`, etc.) are the same thing with a fixed-size history area. Definitions without the `IP_` prefix are older, non-historizing versions.

### 2.4 Root Folder

User-created folders for organizing records for browsing. Purely cosmetic — a record's folder location has no effect on how or whether it is processed.

### 2.5 Historian

History repositories, their file sets, sizes, and rollover behavior. A tag's repository setting must resolve to something defined here. When a tag holds a live value but stores no history, the cause is almost always in this area: archiving turned off, no repository assigned, or a full or unmounted file set.

### 2.6 Cim-IO

The interface layer to external control systems — for an ABB 800xA site, through an OPC server. The same records as in Definition Records, presented by connection instead of by type.

- **Device record (`IoDeviceRecordDef`)** — one per logical connection. Holds the Cim-IO server node name and the logical device/service name matching the Cim-IO side's configuration, and names the tasks that service it (main/get, unsolicited, async/put). **The task binding lives here**, not on the individual transfer records.
- **Get record (`IOGetDef` / `IOLongTagGetDef`)** — points at a device record; carries processing on/off, scan frequency, base time, and status fields. Its repeat area is the actual tag map: one row per tag, containing the device-side tag name, the destination IP.21 record, the destination field (normally `IP_INPUT_VALUE`), and per-row status/quality/timestamp columns. The *LongTag* variant exists because standard get records cannot hold tag names as long as full OPC item paths.
- **Put record (`IOPutDef` / `IOLongTagPutDef`)** — the mirror image: source IP.21 record and field, destination device tag. This is the write path back out to the control system.
- **Unsolicited records** — for data the device pushes on change rather than data you poll for.

Each get record has a practical row limit, so large tag counts are split across multiple get records, typically grouped by scan rate.

### 2.7 Queries and Calculations

- **`QueryDef` (SQLplus)** — SQL text stored inside a record and executed by a SQLplus task. Triggering is on demand, on a schedule, or event-driven. SQLplus can also reach external ODBC data sources, which is the built-in bridge to an outside SQL Server without writing a separate application.
- **Calculation and aggregate records** — formulas and time-based rollups (hourly averages, daily totals) that write their results into their own tags. Often the better home for derived logic than an external query, because the result lands in a real historized tag.

### 2.8 KPI and OEE

Higher-level reporting layers built on top of tag data. Not required for data collection, historization, or automation, and safely ignored unless the site is using those modules.

---

## 3. InfoPlus.21 Manager

Manager is the runtime control panel. It is where IP.21 itself is started and stopped, and where the tasks that service the database are launched, stopped, restarted, and monitored.

### 3.1 External Task Records

Each task is described by an **`ExternalTaskDef`** record naming the executable, its arguments, restart behavior, and priority. The record name is the task name. The record is created and edited in Administrator; the process it describes is controlled in Manager.

### 3.2 What the Tasks Do

Names vary by site, but the roles are consistent:

- **Data collection tasks** — the Cim-IO main, unsolicited, and async processes that service device records and move values in and out.
- **History tasks** — write trend values from tag records into the history repositories.
- **Snapshot save** — writes the in-memory database out to disk.
- **SQLplus tasks** — execute query records.
- **Calculation tasks** — execute formula and aggregate records.

Check your own `ExternalTaskDef` list for the exact set on your system.

### 3.3 Why You Come Here

Manager is the troubleshooting surface. It shows whether a task is running, has died, is failing to connect, or is logging errors. A configuration problem and a stopped task look identical from the tag's point of view — the value simply stops updating — and Manager is what tells the two apart.

### 3.4 The Snapshot

Because the database is memory-resident, record edits exist only in RAM until a save writes the snapshot to disk. An unclean shutdown loses everything changed since the last save. After any significant configuration work, save.

---

## 4. How the Two Fit Together

The full execution chain, from a value in the control system to a historized tag:

```
Control system tag
      ↓
Cim-IO get record  (which tag, where it lands, how often)
      ↓
Device record      (which connection, which tasks service it)
      ↓
External task      (the process that does the polling)
      ↓
IP_AnalogDef tag   (IP_INPUT_VALUE → IP_VALUE)
      ↓
History repository (if archiving is enabled)
```

**Adding a tag, end to end:**

1. Create the tag record from the appropriate definition (Administrator).
2. Set its archiving fields and target repository (Administrator).
3. Add a row to the correct get record mapping the device tag to the new record's input field (Administrator).
4. Cycle processing on that get record — editing a repeat area alone does not take effect until processing is toggled off and back on.
5. Confirm the servicing task is running and error-free (Manager).
6. Save the snapshot.

**The two failure modes to keep straight:**

- Configured but not running — the records are right, the task is stopped or erroring. Diagnosed in Manager.
- Running but not configured — the task is healthy, but nothing is bound to it, processing is off, or the mapping is wrong. Diagnosed in Administrator.

For bulk work, the Administrator record editor is fine for a handful of rows; for hundreds of tags, use record import/export files or SQLplus `INSERT`/`UPDATE` statements against the definitions, which are faster and reproducible.

Exact field names vary slightly across IP.21 versions, so verify against the field list in your own record editor before scripting against them.
