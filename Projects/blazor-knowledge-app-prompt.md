# Project Prompt: Personal Knowledge & File Management Web App (Blazor / C#)

[← Back to index](README.md)

## Overview

Build a self-hosted personal knowledge management web application entirely in C# using Blazor Server (.NET 8 or later). This app is for a single primary user (me) with the possibility of admin-controlled access later. It will host a personal knowledge base ("book" content), a log book, a file browser, and a schedule, with an eye toward long-term deployment, diagnostics, and configurability.

The architecture must prioritize:
- Ease of adding new content without recompiling (drop-in files)
- A path to SQLite-backed indexing/search without abandoning flat-file source content
- Administrator-level controls and system diagnostics suitable for eventual real deployment (not just local dev)
- All core configuration values editable without code changes

---

## Tech Stack

- **.NET 8+ / Blazor Server** (interactive server render mode) — full C# access to the file system, no WASM sandboxing limitations
- **Markdig** (NuGet) — Markdown → HTML conversion, with advanced extensions enabled (tables, footnotes, YAML frontmatter parsing, etc.)
- **KaTeX** (JS interop) — equation rendering for LaTeX-style math (`$$...$$`)
- **Prism.js or Highlight.js** (JS interop) — code block syntax highlighting within notes
- **Entity Framework Core + SQLite** — metadata index, search, schedule, log book entries, admin/config data
- **Serilog** (or built-in `Microsoft.Extensions.Logging` with a file/SQLite sink) — structured logging for diagnostics
- **ASP.NET Core Health Checks** (`Microsoft.AspNetCore.Diagnostics.HealthChecks`) — for deployment readiness

---

## Core Sections

### 1. Notebook (highest priority)
- Personal knowledge base intended to become a book manuscript over time: equations, functions, documented use cases, examples, general reference material.
- Each note is a **Markdown (`.md`) file** stored in a configurable root folder (e.g. `/Content/Notebook/`).
- Support **YAML frontmatter** per note for metadata: `title`, `tags`, `category`, `created`, `updated`.
- Route pattern: single parameterized page (`/notes/{slug}`) that reads the corresponding `.md` file, parses frontmatter, renders body via Markdig, and applies KaTeX/Prism post-render via JS interop. **New notes = new files dropped into the folder — no new pages, no recompiling, no redeploy.**
- Support **file import**: a page or button to upload an existing `.md` (or `.txt`, converted to `.md`) file directly into the Notebook folder, with frontmatter auto-generated if absent (from filename/date) and editable after import.
- Support **wiki-style internal links** (`[[note-slug]]`) that resolve to `/notes/{slug}`.
- Table of Contents page: scans the Notebook folder, reads frontmatter (title/tags/category), and lists all notes. Should not require pre-registration — reads directly off disk. (Auto-populating the *main* app menu from this is a "nice to have," not required for v1.)
- Long-term: metadata (title, tags, category, path) gets indexed into SQLite for fast search/filtering, while the `.md` file remains the source of truth for content. Design the note-loading layer so swapping "read from disk" for "read from SQLite-cached index" is a contained change (e.g. behind an `INoteRepository` interface).

### 2. Log Book
- Same technical pattern as Notebook (Markdown files, frontmatter) but treated as **sequential/dated entries** rather than reference material — think dated journal/lab notebook.
- List view sorted by date, not alphabetically/by category.
- Should support quick-entry (a simple form that creates a new dated `.md` file with a preset frontmatter template) in addition to raw file drop-in.

### 3. File Share
- A basic file browser over one or more configured root directories.
- Lists existing files/folders, allows navigation, download links.
- Low priority for v1 — simple `Directory.GetFiles` / `Directory.GetDirectories` listing with basic breadcrumb navigation is sufficient. No need for upload/delete initially unless trivial to add.

### 4. Schedule
- Basic CRUD calendar/schedule feature (create, view, edit, delete entries with date/time).
- Store entries in SQLite via EF Core.
- Simple calendar or agenda-list UI component; doesn't need to be elaborate for v1.

---

## Administrator Controls

Even for single-user use now, build in an **admin section** (`/admin`) gated behind a simple auth mechanism (ASP.NET Core Identity is fine, or a lightweight single-admin-password scheme if full Identity is overkill for v1 — note the tradeoff and pick one).

Admin section should include:
- **User/access management** (even if only one user exists today, structure it so additional users/roles can be added later without a rearchitecture)
- **Content management view**: list all Notebook/Log Book files with basic metadata, ability to see file paths, last-modified dates, and orphaned/malformed entries (e.g. missing frontmatter, parse errors)
- **Configuration editor**: a UI page to view and update core configuration values at runtime where feasible (see Configuration section below), backed by a config file or SQLite settings table — not requiring redeploys for routine changes
- **Diagnostics dashboard**:
  - Application health status (DB connectivity, file storage accessibility, disk space on content folders)
  - Recent error/log viewer (surface recent entries from the structured log, filterable by severity)
  - Basic usage stats (page views, note counts, last activity) — lightweight, not full analytics
  - .NET/runtime info (app version, environment name, uptime) useful for confirming what's actually deployed
- **Backup/export trigger**: a button to export/zip the current content folders (Notebook, Log Book, config) for manual backup, since flat files are the source of truth

---

## Diagnostics & Deployment Readiness

- Implement **ASP.NET Core Health Checks** at a `/health` endpoint (DB reachability, content directory read/write access) — standard practice for any future container/hosting platform to monitor.
- Structured logging (Serilog recommended) writing to both a rolling file and, ideally, a SQLite or lightweight log table so the admin dashboard can query recent errors without tailing log files by hand.
- Environment-aware configuration (`appsettings.json`, `appsettings.Development.json`, `appsettings.Production.json`) so behavior (log verbosity, content paths, etc.) differs cleanly between local dev and eventual real deployment.
- Design content folder paths, DB connection string, and feature toggles as **configuration values**, not hardcoded paths — see below.

---

## Core Configuration Requirements

All of the following must be defined in configuration (`appsettings.json` + optionally overridable via the admin Configuration Editor at runtime, persisted back to config or a settings table) rather than hardcoded:

- Notebook root folder path
- Log Book root folder path
- File Share root folder path(s)
- SQLite connection string / DB file path
- Log level / log retention settings
- Feature toggles (e.g. enable/disable File Share, enable/disable public registration if multi-user is added later)
- Admin credentials or auth provider settings (stored securely — do not put secrets in plaintext config; use `dotnet user-secrets` for dev and environment variables or a secrets manager for production)

Structure the app so changing any of these does **not** require a code change or full rebuild — a config reload or app restart at most.

---

## Long-Term / Explicitly Deferred (do not over-build now, but don't architect against it)

- Full-text search across Notebook/Log Book (SQLite FTS5) once note volume grows
- Auto-populating the main menu/table of contents from folder contents (structure code so this is a straightforward addition, not required in v1)
- Multi-user roles/permissions beyond a single admin
- Migrating note *content* (not just metadata) into SQLite, if flat files become unwieldy

---

## Deliverable Expectations

- Clean separation of concerns: components for markup/UI, services/repositories for file and DB access (e.g. `INoteRepository`, `IScheduleService`), so the "read from disk" vs "read from DB" swap mentioned above stays contained.
- Reasonably commented code, particularly around the Markdig/KaTeX/JS interop pipeline and the admin diagnostics/config areas, since these are the parts most likely to need future revisiting.
- A brief README covering: how to run locally, where config lives, how to add a new note (just drop a `.md` file — confirm this works end to end), and how to reach the admin panel.
