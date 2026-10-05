# My Notes System — Setup, Run & Deploy Guide

A personal documentation site. You write Markdown in VS Code, D2 diagrams render automatically, and the site runs locally with live reload. When you're ready, you build it and serve it permanently from a Windows Server with IIS.

- **Stack:** Astro Starlight (the docs site framework), astro-d2 (diagrams), Node.js/npm (tooling), IIS (permanent hosting)
- **Where your notes live:** plain `.md` files in `src/content/docs/`. That folder *is* your database.

---

## 1. Why this stack

- **Markdown files are the source of truth.** No database, no lock-in, and everything works with Git.
- **Live reload while writing.** Save a file and the browser updates instantly.
- **D2 support built in.** Write a ` ```d2 ` block and it becomes an SVG diagram.
- **Docs features for free.** You get a sidebar, full-text search, dark mode, callouts, tabs, and code highlighting.
- **Builds to plain static files** (HTML/CSS/JS). Production needs no Node or database, just a web server like IIS. That makes it fast, secure, and hard to break.

---

## 2. What Node.js and npm are

### Node.js
Node.js is a **program that runs JavaScript outside a browser**. Browsers used to be the only place JavaScript could run. Node lets it run on your computer like Python or PowerShell. Tools like Astro are written in JavaScript, so they need Node to run.

**In this project, Node runs:**
- the **dev server**, which hosts your notes at `http://localhost:4321` with live reload
- the **build**, which turns your Markdown into a finished static website in the `dist/` folder

### npm (Node Package Manager)
npm comes **bundled with Node** and does two jobs:

1. **Downloads libraries ("packages")** your project depends on, such as Astro and Starlight, into a `node_modules/` folder.
2. **Runs project commands** defined in `package.json`, such as `npm run dev`.

### Key files and folders

| Thing | What it is |
|---|---|
| `package.json` | The project's manifest: its name, dependencies, and scripts (`dev`, `build`, `preview`). |
| `package-lock.json` | Records the exact versions installed, so every machine installs the same thing. Commit it. |
| `node_modules/` | The downloaded packages. It's huge and disposable. Never edit, copy, or commit it. `npm install` recreates it. |
| `dist/` | The **build output**: the finished website. This is what you deploy. |

### Commands you'll actually use

| Command | What it does |
|---|---|
| `npm install` | Reads `package.json` and downloads everything into `node_modules/`. Run it after cloning or copying the project. |
| `npm run dev` | Starts the live-reload dev server at `http://localhost:4321`. |
| `npm run build` | Builds the production site into `dist/`. |
| `npm run preview` | Serves the built `dist/` locally so you can check the production version. |
| `npx <tool>` | Runs a package's command without installing it globally, e.g. `npx astro add ...`. |

### D2 (not part of Node)
D2 is a separate program (`d2.exe`) that turns diagram text into SVG. The `astro-d2` plugin calls it during dev and build. **It only needs to be installed on the machine that builds the site**, not on the server that hosts `dist/`.

---

## 3. Install the tools (on the machine where you write and build)

This is your Windows desktop. Do the same on the server only if you want to build there too (see section 8).

| Tool | Where to get it | Why |
|---|---|---|
| **Node.js (LTS)** | https://nodejs.org → download the **LTS** Windows Installer (.msi) | Runs Astro. Includes npm. |
| **D2** | https://github.com/terrastruct/d2/releases → latest `d2-vX.Y.Z-windows-amd64.msi` | Renders diagrams. |
| **Git** | https://git-scm.com/download/win | Version history and syncing (recommended). |
| **VS Code** | https://code.visualstudio.com | Your editor. |
| **VS Code extensions** | In VS Code Extensions, search **"D2"** (by Terrastruct) and **"Astro"** | Syntax highlighting. |

On Windows 10/11 you can use winget instead:

```powershell
winget install OpenJS.NodeJS.LTS
winget install terrastruct.d2
winget install Git.Git
```

> Windows Server 2022 doesn't ship with winget, so use the .msi downloads there.

**Close and reopen your terminal**, then verify:

```powershell
node -v     # e.g. v24.x.x
npm -v
d2 --version
git --version
```

If a command isn't found, sign out and back in, or reboot, so Windows picks up the new PATH.

---

## 4. Create the project

In PowerShell:

```powershell
cd C:\
npm create astro@latest -- --template starlight notes
```

Accept the defaults: install dependencies **Yes**, and initialize git **Yes**. Then:

```powershell
cd C:\notes
npx astro add astro-d2      # answer Yes to the prompts; it installs and wires up the plugin
code .                      # open in VS Code
```

### Project layout

```
C:\notes
├─ astro.config.mjs        ← site config (title, sidebar, plugins)
├─ package.json
├─ public/                 ← files copied as-is into the site (web.config goes here)
├─ src/
│  └─ content/
│     └─ docs/             ← ★ YOUR NOTES LIVE HERE ★
│        ├─ index.mdx      ← home page
│        ├─ guides/
│        └─ reference/
└─ dist/                   ← created by `npm run build` (the deployable site)
```

---

## 5. Initial files

Replace or create these.

### `astro.config.mjs`

Keep whatever imports `astro add` created, and make it match this:

```js
// @ts-check
import { defineConfig } from 'astro/config';
import starlight from '@astrojs/starlight';
import astroD2 from 'astro-d2';

export default defineConfig({
  integrations: [
    starlight({
      title: 'My Notes',
      sidebar: [
        { label: 'Start Here', autogenerate: { directory: 'start-here' } },
        { label: 'Notes',      autogenerate: { directory: 'notes' } },
        { label: 'Reference',  autogenerate: { directory: 'reference' } },
      ],
    }),
    astroD2(),
  ],
});
```

`autogenerate` means **any file you drop into that folder appears in the sidebar automatically**.

Create these folders (you can delete the template's `guides/` folder):

```
src/content/docs/start-here/
src/content/docs/notes/
src/content/docs/reference/
```

### `src/content/docs/index.mdx` (home page)

```mdx
---
title: My Notes
description: Personal knowledge base.
template: splash
hero:
  tagline: Everything I know, written down, with diagrams.
  actions:
    - text: Start here
      link: /start-here/how-this-works/
      icon: right-arrow
---
```

### `src/content/docs/start-here/how-this-works.md`

````md
---
title: How this works
description: How to add, edit and remove notes.
sidebar:
  order: 1
---

Every `.md` file under `src/content/docs/` becomes a page.
The folder path becomes the URL: `notes/networking/dns.md` → `/notes/networking/dns/`.

## Workflow

```d2
direction: right
write: Write .md in VS Code
dev: npm run dev\n(live preview)
build: npm run build
deploy: Copy dist/ to server
live: IIS serves the site

write -> dev: save
dev -> build: happy with it
build -> deploy
deploy -> live
```

## Adding a note
1. Create `src/content/docs/notes/<topic>.md`.
2. Add front matter with at least a `title`.
3. Save. It appears in the sidebar.

## Removing a note
Delete the file. Rebuild and deploy to remove it from production.
````

### `src/content/docs/notes/example-note.md`

````md
---
title: Example note
description: Shows the formatting you can use.
---

Regular **Markdown** works: lists, tables, links, code.

:::tip
Callouts: use `note`, `tip`, `caution`, or `danger`.
:::

## A diagram

```d2
user: User {shape: person}
server: Windows Server 2022 {
  iis: IIS
  site: dist/ (static files)
  iis -> site
}
user -> server.iis: HTTP
```

## Code

```powershell
Get-Service W3SVC
```
````

### `src/content/docs/reference/commands.md`

````md
---
title: Commands cheat sheet
---

| Task | Command |
|---|---|
| Live preview | `npm run dev` |
| Build site | `npm run build` |
| Check the build | `npm run preview` |
| Reinstall deps | `npm install` |
| Update packages | `npm update` |
````

### `public/web.config` (makes IIS serve the site correctly)

IIS refuses to serve file types it doesn't recognize. Starlight's search engine (Pagefind) uses unusual extensions, so this file registers them. It sits in `public/`, which means every build copies it into `dist/` automatically.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <staticContent>
      <remove fileExtension=".pagefind" />
      <mimeMap fileExtension=".pagefind" mimeType="application/octet-stream" />
      <remove fileExtension=".pf_meta" />
      <mimeMap fileExtension=".pf_meta" mimeType="application/octet-stream" />
      <remove fileExtension=".pf_index" />
      <mimeMap fileExtension=".pf_index" mimeType="application/octet-stream" />
      <remove fileExtension=".pf_fragment" />
      <mimeMap fileExtension=".pf_fragment" mimeType="application/octet-stream" />
      <remove fileExtension=".wasm" />
      <mimeMap fileExtension=".wasm" mimeType="application/wasm" />
      <remove fileExtension=".webmanifest" />
      <mimeMap fileExtension=".webmanifest" mimeType="application/manifest+json" />
    </staticContent>
    <httpErrors errorMode="Custom" existingResponse="Replace">
      <remove statusCode="404" />
      <error statusCode="404" path="/404.html" responseMode="ExecuteURL" />
    </httpErrors>
  </system.webServer>
</configuration>
```

### `start-dev.bat` (project root; used for dev mode on boot)

```bat
@echo off
cd /d C:\notes
set PATH=%PATH%;C:\Program Files\nodejs;C:\Program Files\D2
npm run dev -- --host >> C:\notes\dev-server.log 2>&1
```

### `deploy.ps1` (project root; build and copy to the server)

```powershell
param(
  # Where the site lives on the server. Use a local path if hosting on this machine.
  [string]$Target = "\\YOUR-SERVER\c$\inetpub\notes"
)
$ErrorActionPreference = "Stop"
Set-Location $PSScriptRoot

npm run build
if ($LASTEXITCODE -ne 0) { throw "Build failed - nothing deployed." }

# /MIR = make the target an exact mirror of dist (removes deleted pages too)
robocopy .\dist $Target /MIR /NFL /NDL /NJH /NP
if ($LASTEXITCODE -ge 8) { throw "Copy failed (robocopy code $LASTEXITCODE)." }
Write-Host "Deployed to $Target" -ForegroundColor Green
```

---

## 6. Run it (dev mode, while writing)

```powershell
cd C:\notes
npm run dev
```

Open **http://localhost:4321**. Edit any `.md` and the browser refreshes on save. Press `Ctrl+C` to stop.

- **Debugging:** errors show in the terminal and as an overlay in the browser. Most issues are a missing `title:` in front matter or a D2 syntax error.
- **Search** only works on the built site (`npm run build` then `npm run preview`), not in dev mode.
- To reach it from other devices on your network, run `npm run dev -- --host` and open `http://<your-pc-ip>:4321`.

---

## 7. Run it permanently (starts on boot)

There are two ways. **Option A is the recommended production setup.**

### Option A — IIS serving the built site (recommended)

IIS is Windows' built-in web server. It runs as a service, starts at boot, and needs no Node. It works the same on Windows Server 2022 or your desktop.

**1. Install IIS** (PowerShell as Administrator):

```powershell
# Windows Server 2022:
Install-WindowsFeature Web-Server -IncludeManagementTools

# Windows 10/11 desktop (if hosting there for now):
Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebServerRole, IIS-WebServer, IIS-StaticContent, IIS-DefaultDocument, IIS-HttpErrors, IIS-ManagementConsole
```

**2. Create the site** on port 8080, which avoids clashing with IIS's "Default Web Site" on port 80:

```powershell
Import-Module WebAdministration
New-Item C:\inetpub\notes -ItemType Directory -Force
New-Website -Name "Notes" -Port 8080 -PhysicalPath "C:\inetpub\notes"
New-NetFirewallRule -DisplayName "Notes site (8080)" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
```

If you'd rather use port 80, stop the default site and use port 80 in the commands above:

```powershell
Stop-Website "Default Web Site"
```

**3. Deploy** from your desktop, in the project folder:

```powershell
.\deploy.ps1 -Target "\\YOUR-SERVER\c$\inetpub\notes"   # to the server
.\deploy.ps1 -Target "C:\inetpub\notes"                 # if IIS is on this same machine
```

Copying to `\\server\c$` requires an admin account on the server. You can also RDP in and copy the `dist\` contents to `C:\inetpub\notes` by hand.

**4. Open it:** go to `http://YOUR-SERVER:8080` (or `http://localhost:8080`).

IIS starts automatically on every boot. There's nothing else to configure.

### Option B — Dev server on boot (always-live editing)

Use this if you want the machine to always run the live-reload version, for example your desktop acting as a wiki. It needs Node and D2 installed on that machine.

PowerShell as Administrator:

```powershell
schtasks /create /tn "NotesDevServer" /tr "C:\notes\start-dev.bat" /sc onstart /ru SYSTEM /rl HIGHEST
New-NetFirewallRule -DisplayName "Notes dev (4321)" -Direction Inbound -Protocol TCP -LocalPort 4321 -Action Allow
```

- Start it now without rebooting: `schtasks /run /tn "NotesDevServer"`
- Stop it: `schtasks /end /tn "NotesDevServer"`
- Remove it: `schtasks /delete /tn "NotesDevServer" /f`
- Logs: `C:\notes\dev-server.log`

> Dev mode is slower and not hardened. Treat it as a convenience for trusted networks, not public hosting.

---

## 8. Moving to production

| Approach | Server needs | How |
|---|---|---|
| **Build on desktop, copy `dist/`** (simplest) | IIS only | Run `.\deploy.ps1` after editing. |
| **Build on the server** | IIS, Node, D2, Git | Push to a Git remote from your desktop. On the server: `git pull`, `npm install`, `npm run build`, then `robocopy dist C:\inetpub\notes /MIR`. |

To copy the **whole project** to another machine, copy everything **except** `node_modules/` and `dist/` (or use `git clone`), then run `npm install` there.

Later, you can have GitHub Actions or Cloudflare Pages build and host it automatically on every `git push`. The files stay exactly the same.

---

## 9. Day-to-day: adding, editing, removing notes

**Add a note:** create a `.md` file under `src/content/docs/` (e.g. `notes/networking/dns.md`):

```md
---
title: DNS basics
description: Optional one-liner used in search and previews.
sidebar:
  order: 3          # optional: position in the sidebar
---

Content here.
```

- **Add a new section:** create a folder under `notes/`. With `autogenerate`, it appears automatically. For a new top-level section, add another line to `sidebar` in `astro.config.mjs`.
- **Edit a note:** open it, change it, save. The dev server updates live.
- **Remove a note:** delete the file. Its sidebar entry disappears.
- **Rename or move a note:** move the file. The URL changes to match the new path.
- **Publish changes:** run `.\deploy.ps1`. `/MIR` removes deleted pages from the server too.
- **Save history:** `git add . ; git commit -m "notes: dns"` (and `git push` if you have a remote).

### Diagram tips
- Use ` ```d2 ` for a diagram block. The D2 tour at https://d2lang.com/tour/intro covers the syntax.
- Try different looks with attributes on the opening line, e.g. ` ```d2 sketch ` or ` ```d2 layout=elk `.

---

## 10. Troubleshooting

| Problem | Fix |
|---|---|
| `node`, `npm`, or `d2` not recognized | Reopen the terminal, or sign out and back in. Check that Node's and D2's install folders are on PATH. |
| Diagrams don't render / "d2 not found" | D2 isn't installed or isn't on PATH on the machine running `dev` or `build`. |
| Page missing from sidebar | The file isn't under an autogenerated folder, or it's missing `title:` front matter. |
| IIS shows 404 on search or a blank page | `web.config` is missing from `C:\inetpub\notes`. Make sure it's in `public/` and rebuild. |
| IIS site won't start | Port conflict. Pick another port, or stop "Default Web Site". |
| Dependencies broken | Delete `node_modules` and `package-lock.json`, then run `npm install`. |
| Execution policy blocks `deploy.ps1` | Run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`. |
