# Windows System Tools & Command-Line Reference

> **How to use this file:** Press `Ctrl + F` and search for what you want to do: "wifi password", "port", "repair", "startup", "battery", "services", "hash", "uninstall", and so on. Every entry describes the task in plain words so searches hit.
>
> Written for **Windows 11** (most of it also works on Windows 10). Items marked **(Pro)** are only in Pro/Enterprise/Education editions. Items marked **⚠** can destroy data or break the system if misused. Read them twice before running.

---

## Table of Contents

1. [How to Launch Things](#1-how-to-launch-things)
2. [Management Consoles (.msc)](#2-management-consoles-msc)
3. [Built-in System Utilities (.exe)](#3-built-in-system-utilities-exe)
4. [Control Panel Applets (.cpl) & `control` Commands](#4-control-panel-applets-cpl--control-commands)
5. [Settings Pages (`ms-settings:` links)](#5-settings-pages-ms-settings-links)
6. [Special Folders (`shell:` & environment variables)](#6-special-folders-shell--environment-variables)
7. [Terminals: CMD vs PowerShell vs Windows Terminal](#7-terminals-cmd-vs-powershell-vs-windows-terminal)
8. [winget — Package Manager](#8-winget--package-manager)
9. [System Repair & Health](#9-system-repair--health)
10. [System Information](#10-system-information)
11. [Power, Battery & Shutdown](#11-power-battery--shutdown)
12. [Networking](#12-networking)
13. [Files, Folders & Disks](#13-files-folders--disks)
14. [Processes, Services & Scheduled Tasks](#14-processes-services--scheduled-tasks)
15. [Users, Permissions, Policy & Security](#15-users-permissions-policy--security)
16. [Registry from the Command Line](#16-registry-from-the-command-line)
17. [Event Logs](#17-event-logs)
18. [PowerShell Essentials](#18-powershell-essentials)
19. [Developer & Other Built-in CLI Tools (WSL, ssh, curl, tar, sudo)](#19-developer--other-built-in-cli-tools)
20. [Command-Line Tricks (CMD & PowerShell)](#20-command-line-tricks-cmd--powershell)
21. [Boot, Recovery & Safe Mode](#21-boot-recovery--safe-mode)
22. [Troubleshooting Recipes](#22-troubleshooting-recipes)
23. [Recommended Free Tools (not built in)](#23-recommended-free-tools-not-built-in)

---

## 1. How to Launch Things

| Method | How |
|---|---|
| Run dialog | `Win + R`, type the command, `Enter` |
| Run as administrator from Run dialog | type the command, then `Ctrl + Shift + Enter` |
| Start menu search | press `Win`, type the name |
| Admin terminal quickly | `Win + X`, then `A` (Terminal (Admin)) |
| Open terminal in current Explorer folder | type `cmd`, `powershell` or `wt` in Explorer's address bar |
| Open Explorer from a terminal in current folder | `start .` (CMD) or `ii .` (PowerShell) |
| All tools folder | `control admintools` (opens "Windows Tools") |
| "God Mode" folder (every Control Panel setting in one list) | create a new folder named `GodMode.{ED7BA470-8E54-465E-825C-99712043E01C}` |

Everything in sections 2–5 can be typed into `Win + R`, a terminal, or the Explorer address bar.

---

## 2. Management Consoles (.msc)

| Command | Opens | Use it to… |
|---|---|---|
| `compmgmt.msc` | Computer Management | One-stop hub: disks, services, event logs, users, shared folders |
| `devmgmt.msc` | Device Manager | Update/roll back/disable drivers, find unknown devices |
| `diskmgmt.msc` | Disk Management | Partition, format, shrink/extend volumes, change drive letters |
| `eventvwr.msc` | Event Viewer | Read system/app error logs (why did it crash?) |
| `services.msc` | Services | Start/stop services, set startup type |
| `taskschd.msc` | Task Scheduler | See/create scheduled tasks |
| `perfmon.msc` | Performance Monitor | Detailed performance counters & reports |
| `perfmon /rel` | Reliability Monitor | Timeline of crashes, failed updates, installs — great first stop for "it's been unstable" |
| `wf.msc` | Windows Defender Firewall with Advanced Security | Inbound/outbound rules |
| `certmgr.msc` | Certificates (current user) | View/remove user certificates |
| `certlm.msc` | Certificates (local machine) | Machine-wide certificates |
| `fsmgmt.msc` | Shared Folders | See shares, open sessions, open files |
| `tpm.msc` | TPM Management | Check TPM status (Windows 11 requirement) |
| `wmimgmt.msc` | WMI Control | WMI settings/repair |
| `comexp.msc` | Component Services | COM+/DCOM settings (advanced) |
| `gpedit.msc` **(Pro)** | Local Group Policy Editor | Hundreds of system policies |
| `secpol.msc` **(Pro)** | Local Security Policy | Password, audit, user-rights policies |
| `lusrmgr.msc` **(Pro)** | Local Users and Groups | Manage local accounts & groups |
| `printmanagement.msc` **(Pro)** | Print Management | Printers, drivers, queues |
| `rsop.msc` | Resultant Set of Policy | Which policies are actually applied |

---

## 3. Built-in System Utilities (.exe)

### Monitoring & diagnostics
| Command | Tool | Use it to… |
|---|---|---|
| `taskmgr` | Task Manager (`Ctrl + Shift + Esc`) | Kill apps, see CPU/RAM/disk/GPU, manage startup apps |
| `resmon` | Resource Monitor | Which process is using disk/network/a specific file |
| `msinfo32` | System Information | Full hardware/software inventory, BIOS version, Secure Boot state |
| `dxdiag` | DirectX Diagnostic | GPU, display, sound info; save report for support |
| `mdsched` | Windows Memory Diagnostic | Test RAM on next reboot |
| `winver` | About Windows | Exact Windows version & build |
| `verifier` ⚠ | Driver Verifier | Stress-test drivers (can cause boot loops; advanced only) |
| `sigverif` | File Signature Verification | Find unsigned system files |

### Configuration
| Command | Tool | Use it to… |
|---|---|---|
| `msconfig` | System Configuration | Safe boot, diagnostic startup, disable non-Microsoft services |
| `regedit` ⚠ | Registry Editor | Edit the registry (back up first: File → Export) |
| `sysdm.cpl` | System Properties | Computer name, domain, remote settings, env variables |
| `SystemPropertiesAdvanced` | System Properties → Advanced | Environment variables, performance, startup & recovery |
| `SystemPropertiesProtection` | System Protection | Turn on System Restore, create restore point |
| `SystemPropertiesPerformance` | Performance Options | Visual effects, virtual memory (page file) |
| `rstrui` | System Restore | Roll back to a restore point |
| `netplwiz` (or `control userpasswords2`) | User Accounts | Auto-login, manage accounts |
| `optionalfeatures` | Windows Features | Turn on Hyper-V, WSL, .NET 3.5, Telnet, Sandbox, etc. |
| `cttune` | ClearType Tuner | Sharper text |
| `dccw` | Display Color Calibration | Calibrate monitor colors |
| `colorcpl` | Color Management | ICC color profiles |
| `sdclt` | Backup and Restore (Windows 7) | System image backup |
| `recoverydrive` | Recovery Drive creator | Make a recovery USB |
| `iscsicpl` | iSCSI Initiator | Connect network storage |
| `shrpubw` | Create a Shared Folder Wizard | Share a folder quickly |

### Maintenance
| Command | Tool | Use it to… |
|---|---|---|
| `cleanmgr` | Disk Cleanup | Delete temp files, old updates ("Clean up system files") |
| `dfrgui` | Optimize Drives | Defrag HDDs / TRIM SSDs |
| `mrt` | Malicious Software Removal Tool | Quick malware scan |
| `wsreset` | Microsoft Store reset | Fix a broken Store |

### Everyday & accessibility apps
| Command | App |
|---|---|
| `notepad` | Notepad |
| `mspaint` | Paint |
| `calc` | Calculator |
| `snippingtool` | Snipping Tool |
| `charmap` | Character Map (special symbols) |
| `mstsc` | Remote Desktop Connection |
| `msra` | Windows Remote Assistance |
| `quickassist` | Quick Assist (help someone remotely) |
| `osk` | On-Screen Keyboard |
| `magnify` | Magnifier |
| `narrator` | Narrator |
| `fsquirt` | Bluetooth File Transfer |
| `write` | WordPad (removed in Windows 11 24H2) |

---

## 4. Control Panel Applets (.cpl) & `control` Commands

| Command | Opens |
|---|---|
| `control` | Control Panel |
| `appwiz.cpl` | Programs and Features (uninstall classic apps) |
| `ncpa.cpl` | Network Connections (adapters, IPv4 settings, DNS) |
| `sysdm.cpl` | System Properties |
| `powercfg.cpl` | Power Options (classic power plans) |
| `mmsys.cpl` | Sound (playback/recording devices, classic panel) |
| `main.cpl` | Mouse Properties |
| `inetcpl.cpl` | Internet Options (proxy, certificates, reset) |
| `firewall.cpl` | Windows Defender Firewall |
| `timedate.cpl` | Date and Time |
| `intl.cpl` | Region (date/number formats) |
| `joy.cpl` | Game Controllers |
| `wscui.cpl` | Security and Maintenance |
| `desk.cpl` | Display settings |
| `hdwwiz.cpl` | Device Manager |
| `control printers` | Devices and Printers |
| `control admintools` | Windows Tools |
| `control keyboard` | Keyboard Properties |
| `control folders` | File Explorer Options (show hidden files, extensions) |
| `control fonts` | Fonts folder |
| `control /name Microsoft.CredentialManager` | Credential Manager (saved passwords) |
| `control /name Microsoft.DefaultPrograms` | Default Programs |
| `rundll32 sysdm.cpl,EditEnvironmentVariables` | Environment Variables editor directly |

---

## 5. Settings Pages (`ms-settings:` links)

Type into `Win + R`, or run `start ms-settings:xyz` in a terminal.

| Link | Page |
|---|---|
| `ms-settings:` | Settings home |
| `ms-settings:about` | System → About (PC name, specs, rename PC) |
| `ms-settings:display` | Display, scaling, resolution, multiple monitors |
| `ms-settings:sound` | Sound |
| `ms-settings:notifications` | Notifications |
| `ms-settings:powersleep` | Power & sleep |
| `ms-settings:storagesense` | Storage (cleanup recommendations, Storage Sense) |
| `ms-settings:clipboard` | Clipboard history |
| `ms-settings:troubleshoot` | Troubleshooters |
| `ms-settings:recovery` | Recovery (reset PC, advanced startup) |
| `ms-settings:activation` | Windows activation |
| `ms-settings:developers` | For developers (sudo, dev mode, terminal defaults) |
| `ms-settings:bluetooth` | Bluetooth & devices |
| `ms-settings:printers` | Printers & scanners |
| `ms-settings:mousetouchpad` | Mouse |
| `ms-settings:network-status` | Network & internet |
| `ms-settings:network-wifi` | Wi-Fi |
| `ms-settings:network-proxy` | Proxy |
| `ms-settings:network-vpn` | VPN |
| `ms-settings:personalization` | Personalization |
| `ms-settings:taskbar` | Taskbar |
| `ms-settings:appsfeatures` | Installed apps (uninstall) |
| `ms-settings:defaultapps` | Default apps |
| `ms-settings:startupapps` | Startup apps |
| `ms-settings:optionalfeatures` | Optional features |
| `ms-settings:yourinfo` | Accounts → Your info |
| `ms-settings:signinoptions` | Sign-in options (PIN, Hello) |
| `ms-settings:dateandtime` | Date & time |
| `ms-settings:regionlanguage` | Language & region |
| `ms-settings:privacy` | Privacy & security |
| `ms-settings:windowsdefender` | Windows Security |
| `ms-settings:windowsupdate` | Windows Update |
| `ms-settings:windowsupdate-history` | Update history (uninstall updates) |

---

## 6. Special Folders (`shell:` & environment variables)

### `shell:` folders (type in `Win + R` or the Explorer address bar)
| Command | Folder |
|---|---|
| `shell:startup` | Your Startup folder (apps here run at login) |
| `shell:common startup` | Startup folder for all users |
| `shell:appsfolder` | Every installed app, including Store apps (drag to desktop for shortcuts) |
| `shell:sendto` | "Send to" right-click menu items |
| `shell:downloads` | Downloads |
| `shell:recent` | Recently opened files |
| `shell:desktop` | Desktop |
| `shell:fonts` | Fonts |
| `shell:RecycleBinFolder` | Recycle Bin |
| `shell:programfiles` | Program Files |
| `shell:system` | C:\Windows\System32 |
| `shell:start menu` | Your Start menu shortcuts |

### Environment variables (work in Run, Explorer, CMD; in PowerShell use `$env:NAME`)
| Variable | Typical path |
|---|---|
| `%USERPROFILE%` | C:\Users\You |
| `%APPDATA%` | C:\Users\You\AppData\Roaming |
| `%LOCALAPPDATA%` | C:\Users\You\AppData\Local |
| `%TEMP%` | C:\Users\You\AppData\Local\Temp (safe to clear) |
| `%PROGRAMDATA%` | C:\ProgramData |
| `%PROGRAMFILES%` | C:\Program Files |
| `%PROGRAMFILES(X86)%` | C:\Program Files (x86) |
| `%WINDIR%` / `%SYSTEMROOT%` | C:\Windows |
| `%SYSTEMDRIVE%` | C: |
| `%COMPUTERNAME%` | Your PC's name |
| `%USERNAME%` | Your user name |
| `%PATH%` | Folders searched for commands |

Show all variables: `set` (CMD) or `Get-ChildItem env:` (PowerShell).

---

## 7. Terminals: CMD vs PowerShell vs Windows Terminal

| | What it is | When to use |
|---|---|---|
| **Command Prompt** (`cmd`) | Classic shell, batch (.bat) scripts | Old commands, quick one-liners, legacy scripts |
| **Windows PowerShell 5.1** (`powershell`) | Built-in object-based shell | Admin tasks, anything with `Get-` / `Set-` cmdlets |
| **PowerShell 7** (`pwsh`) | Newer cross-platform PowerShell (install: `winget install Microsoft.PowerShell`) | Modern scripting; recommended default |
| **Windows Terminal** (`wt`) | Tabbed app that hosts CMD, PowerShell, WSL, SSH | The window you should actually use |

Every classic command in this file (`ipconfig`, `ping`, `robocopy`, etc.) works in both CMD and PowerShell. `Get-*` cmdlets only work in PowerShell.

**Getting help:** `command /?` (CMD tools) · `Get-Help Cmdlet-Name -Examples` (PowerShell) · `winget --help`.

**Run as administrator:** `Win + X` → `A`, or right-click Terminal → Run as administrator. Many repair, disk and network commands need it.

---

## 8. winget — Package Manager

Built into Windows 10/11 (via "App Installer"). Installs, updates and removes apps from the command line.

| Task | Command |
|---|---|
| Search for an app | `winget search firefox` |
| Show details of a package | `winget show Mozilla.Firefox` |
| Install an app (exact ID) | `winget install --id Mozilla.Firefox -e` |
| Install silently, auto-accept agreements | `winget install --id 7zip.7zip -e --silent --accept-package-agreements --accept-source-agreements` |
| List installed apps (including non-winget ones) | `winget list` |
| See available updates | `winget upgrade` |
| Update one app | `winget upgrade --id Mozilla.Firefox -e` |
| **Update everything** | `winget upgrade --all` |
| Update everything, including apps with unknown versions | `winget upgrade --all --include-unknown` |
| Uninstall | `winget uninstall --id Mozilla.Firefox -e` |
| Stop an app from being upgraded | `winget pin add --id Some.App` |
| Export list of installed apps (backup) | `winget export -o apps.json` |
| Reinstall everything from that list on a new PC | `winget import -i apps.json` |
| Refresh package sources | `winget source update` |
| Reset sources if broken | `winget source reset --force` (admin) |

**Starter kit example:**
```
winget install -e --id Microsoft.PowerToys
winget install -e --id Microsoft.PowerShell
winget install -e --id Git.Git
winget install -e --id Microsoft.VisualStudioCode
winget install -e --id 7zip.7zip
winget install -e --id voidtools.Everything
```

Browse packages online at **winstall.app** or **winget.run**. Alternatives: **Scoop** and **Chocolatey** (third-party package managers).

---

## 9. System Repair & Health

Run from an **admin** terminal. The standard "Windows is acting weird" sequence is **DISM → SFC → CHKDSK**.

| Task | Command |
|---|---|
| Quick check of the Windows image | `DISM /Online /Cleanup-Image /CheckHealth` |
| Deeper scan of the image | `DISM /Online /Cleanup-Image /ScanHealth` |
| **Repair the Windows image** (downloads good files from Windows Update) | `DISM /Online /Cleanup-Image /RestoreHealth` |
| Clean up old component versions (frees space) | `DISM /Online /Cleanup-Image /StartComponentCleanup` |
| **Scan & repair protected system files** | `sfc /scannow` |
| Check disk online, no reboot | `chkdsk C: /scan` |
| Fix file-system errors (schedules at reboot for C:) | `chkdsk C: /f` |
| Fix errors + scan for bad sectors (slow) | `chkdsk C: /r` |
| Check if the recovery environment is enabled | `reagentc /info` |
| Reset Microsoft Store | `wsreset` |
| Re-register all Store apps (PowerShell, admin) | `Get-AppxPackage -AllUsers \| Foreach {Add-AppxPackage -DisableDevelopmentMode -Register "$($_.InstallLocation)\AppXManifest.xml"}` |
| Test RAM | `mdsched` |
| Create a restore point (PowerShell, admin) | `Checkpoint-Computer -Description "Before changes" -RestorePointType MODIFY_SETTINGS` |
| Restore to an earlier point | `rstrui` |

SFC's log: `C:\Windows\Logs\CBS\CBS.log` · DISM's log: `C:\Windows\Logs\DISM\dism.log`.

---

## 10. System Information

| Task | Command |
|---|---|
| Full system summary (OS, install date, RAM, hotfixes, uptime) | `systeminfo` |
| Windows version/build | `winver` or `ver` |
| PC name | `hostname` |
| Current user | `whoami` |
| Current user's groups & privileges (am I admin?) | `whoami /all` |
| Installed drivers | `driverquery /v` |
| Everything about the computer (PowerShell) | `Get-ComputerInfo` |
| CPU info | `Get-CimInstance Win32_Processor` |
| RAM sticks | `Get-CimInstance Win32_PhysicalMemory \| Select Manufacturer, Capacity, Speed` |
| Disk models & health status | `Get-PhysicalDisk \| Select FriendlyName, MediaType, HealthStatus, Size` |
| BIOS version & serial number | `Get-CimInstance Win32_BIOS` |
| Motherboard | `Get-CimInstance Win32_BaseBoard` |
| GPU | `Get-CimInstance Win32_VideoController \| Select Name, DriverVersion` |
| Installed updates | `Get-HotFix` |
| Uptime (last boot) | `(Get-CimInstance Win32_OperatingSystem).LastBootUpTime` |
| Installed programs | `winget list` |
| Windows product key stored in firmware | `(Get-CimInstance SoftwareLicensingService).OA3xOriginalProductKey` |
| Activation status | `slmgr /xpr` (expiry) · `slmgr /dlv` (details) |

> **Note:** `wmic` is deprecated and removed by default in Windows 11 24H2. Use the `Get-CimInstance` equivalents above.

---

## 11. Power, Battery & Shutdown

| Task | Command |
|---|---|
| **Battery health report** (design vs. current capacity) | `powercfg /batteryreport` → opens `battery-report.html` in the current folder |
| Energy efficiency problems report (admin) | `powercfg /energy` |
| Sleep/standby history (Modern Standby laptops) | `powercfg /sleepstudy` |
| What's preventing sleep right now (admin) | `powercfg /requests` |
| What woke the PC last | `powercfg /lastwake` |
| Devices allowed to wake the PC | `powercfg /devicequery wake_armed` |
| Supported sleep states | `powercfg /a` |
| List power plans | `powercfg /list` |
| Turn hibernation off/on (frees disk space) | `powercfg /hibernate off` / `on` |
| Shut down now | `shutdown /s /t 0` |
| Restart now | `shutdown /r /t 0` |
| Shut down in 1 hour | `shutdown /s /t 3600` |
| Cancel a scheduled shutdown | `shutdown /a` |
| Sign out | `shutdown /l` |
| Hibernate | `shutdown /h` |
| Restart into Advanced Startup (recovery menu) | `shutdown /r /o /t 0` |
| Restart into BIOS/UEFI setup | `shutdown /r /fw /t 0` |
| Full shutdown (skip Fast Startup) | `shutdown /s /f /t 0` or hold `Shift` while clicking Shut down |

---

## 12. Networking

### Basics
| Task | Command |
|---|---|
| Show IP, gateway, DNS, MAC for all adapters | `ipconfig /all` |
| Release / renew IP address | `ipconfig /release` then `ipconfig /renew` |
| **Flush DNS cache** (fix "site won't load after change") | `ipconfig /flushdns` |
| Show DNS cache | `ipconfig /displaydns` |
| Test reachability | `ping google.com` |
| Ping continuously (stop with `Ctrl + C`) | `ping -t 8.8.8.8` |
| Ping a set number of times | `ping -n 20 8.8.8.8` |
| Trace the route to a host | `tracert google.com` |
| Trace + packet loss per hop | `pathping google.com` |
| DNS lookup | `nslookup example.com` |
| DNS lookup against a specific server | `nslookup example.com 1.1.1.1` |
| DNS lookup (PowerShell) | `Resolve-DnsName example.com` |
| Test if a port is open | `Test-NetConnection example.com -Port 443` |
| MAC addresses | `getmac /v` |
| Routing table | `route print` |
| ARP table (devices seen on LAN) | `arp -a` |

### Ports & connections
| Task | Command |
|---|---|
| All connections + process IDs | `netstat -ano` |
| Show which program owns each connection (admin) | `netstat -abno` |
| **What's using port 8080?** | `netstat -ano \| findstr :8080` then `tasklist /fi "PID eq 1234"` |
| Same, in PowerShell | `Get-Process -Id (Get-NetTCPConnection -LocalPort 8080).OwningProcess` |
| Listening ports only (PowerShell) | `Get-NetTCPConnection -State Listen` |

### Wi-Fi
| Task | Command |
|---|---|
| Saved Wi-Fi networks | `netsh wlan show profiles` |
| **Show a saved Wi-Fi password** | `netsh wlan show profile name="NetworkName" key=clear` → "Key Content" |
| Current Wi-Fi details (signal, channel, speed) | `netsh wlan show interfaces` |
| Wi-Fi history & disconnects report (admin) | `netsh wlan show wlanreport` |
| Forget a Wi-Fi network | `netsh wlan delete profile name="NetworkName"` |

### Reset / repair network (admin, then reboot)
| Task | Command |
|---|---|
| Reset Winsock | `netsh winsock reset` |
| Reset TCP/IP stack | `netsh int ip reset` |
| Full network reset (GUI) | Settings → Network & internet → Advanced network settings → Network reset |

### Shares, drives, remote
| Task | Command |
|---|---|
| Map a network drive | `net use Z: \\server\share /persistent:yes` |
| List mapped drives | `net use` |
| Remove a mapped drive | `net use Z: /delete` |
| List this PC's shares | `net share` |
| Remote Desktop | `mstsc /v:hostname` |
| SSH into a server | `ssh user@host` |
| Download a file | `curl -O https://example.com/file.zip` |
| Show firewall profile status | `netsh advfirewall show allprofiles` |
| Show proxy settings | `netsh winhttp show proxy` |
| Network adapters (PowerShell) | `Get-NetAdapter` |
| IP addresses (PowerShell) | `Get-NetIPAddress -AddressFamily IPv4` |

---

## 13. Files, Folders & Disks

### Navigating & viewing (CMD)
| Task | Command |
|---|---|
| List files | `dir` |
| List including hidden files | `dir /a` |
| Search subfolders for files | `dir /s /b *.pdf` |
| Change folder / drive | `cd C:\Path` · `cd ..` · `D:` |
| Change drive and folder at once | `cd /d D:\Path` |
| Show folder tree | `tree /f` |
| Print a text file | `type file.txt` |
| Page through output | `command \| more` |
| Find where a program lives | `where python` |
| Open current folder in Explorer | `start .` |

### Creating, copying, moving, deleting
| Task | Command |
|---|---|
| Make folder (creates parents too) | `mkdir C:\a\b\c` |
| Copy file | `copy source.txt dest.txt` |
| Copy folder tree | `xcopy C:\src D:\dst /e /i /h` |
| **Robust folder copy (resumes, retries, logs)** | `robocopy C:\src D:\dst /e /z /r:2 /w:5 /mt:16 /log:copy.log` |
| ⚠ Mirror a folder (deletes extras in destination) | `robocopy C:\src D:\dst /mir` |
| Move / rename | `move a.txt folder\` · `ren old.txt new.txt` |
| Delete files | `del file.txt` · `del /s /q *.tmp` |
| ⚠ Delete a folder and everything in it | `rd /s /q C:\folder` |
| Create symbolic link / junction | `mklink /d Link Target` · `mklink /j Link Target` |
| Compare two files | `fc a.txt b.txt` |
| Change attributes (unhide) | `attrib -h -s file` |

### Searching text
| Task | Command |
|---|---|
| Find text in files (CMD) | `findstr /s /i "search term" *.txt` |
| Find text in files (PowerShell "grep") | `Select-String -Path *.log -Pattern "error"` |
| Find files by name recursively (PowerShell) | `Get-ChildItem -Recurse -Filter *.docx` |
| **Find the 20 largest files on C:** | `Get-ChildItem C:\ -Recurse -File -ErrorAction SilentlyContinue \| Sort Length -Desc \| Select -First 20 FullName, @{n='GB';e={[math]::Round($_.Length/1GB,2)}}` |
| Watch a log file live | `Get-Content app.log -Tail 20 -Wait` |

### Ownership, permissions & hashes
| Task | Command |
|---|---|
| Take ownership of a folder (admin) | `takeown /f C:\folder /r /d y` |
| Give yourself full control | `icacls C:\folder /grant %USERNAME%:F /t` |
| View permissions | `icacls C:\folder` |
| **Verify a download's hash** | `certutil -hashfile file.iso SHA256` or `Get-FileHash file.iso` |
| Zip / unzip (built-in tar) | `tar -a -cf archive.zip folder` · `tar -xf archive.zip` |
| Zip / unzip (PowerShell) | `Compress-Archive folder out.zip` · `Expand-Archive out.zip dest` |
| Copy command output to clipboard | `ipconfig /all \| clip` |

### Disks & volumes
| Task | Command |
|---|---|
| List volumes (PowerShell) | `Get-Volume` |
| Free space per drive | `Get-PSDrive -PSProvider FileSystem` |
| Disk partitioning tool (admin) | `diskpart` → `list disk`, `select disk 1`, `list partition`, `list volume` |
| ⚠ Wipe a disk completely in diskpart | `select disk N` → `clean` (**double-check the disk number**) |
| ⚠ Format a drive | `format E: /fs:ntfs /q` |
| Drive label | `label E: Backup` · `vol` |
| Is TRIM enabled for SSDs? (0 = yes) | `fsutil behavior query DisableDeleteNotify` |
| Securely wipe free space | `cipher /w:C:\` (slow) |
| BitLocker status (admin) | `manage-bde -status` |
| Show BitLocker recovery key (admin) | `manage-bde -protectors -get C:` |

---

## 14. Processes, Services & Scheduled Tasks

| Task | Command |
|---|---|
| List running processes | `tasklist` |
| Find a process | `tasklist \| findstr chrome` |
| Kill by name (force) | `taskkill /im chrome.exe /f` |
| Kill by PID | `taskkill /pid 1234 /f` |
| Kill process and its children | `taskkill /im app.exe /t /f` |
| Top memory users (PowerShell) | `Get-Process \| Sort WS -Desc \| Select -First 10 Name, Id, @{n='MB';e={[int]($_.WS/1MB)}}` |
| List services | `sc query` or `Get-Service` |
| Running services only | `Get-Service \| Where Status -eq Running` |
| Start / stop a service (admin) | `net start Spooler` · `net stop Spooler` |
| Restart a service (PowerShell, admin) | `Restart-Service Spooler` |
| Change startup type (admin) | `sc config Spooler start= auto` (note the space after `=`) |
| List scheduled tasks | `schtasks /query /fo table` |
| Create a daily task | `schtasks /create /tn "Backup" /tr "C:\scripts\backup.bat" /sc daily /st 09:00` |
| Run a task now | `schtasks /run /tn "Backup"` |
| Delete a task | `schtasks /delete /tn "Backup" /f` |
| Startup programs (PowerShell) | `Get-CimInstance Win32_StartupCommand \| Select Name, Command, Location` |

---

## 15. Users, Permissions, Policy & Security

| Task | Command |
|---|---|
| List local users | `net user` |
| Info about a user | `net user username` |
| Create a local user (admin) | `net user newuser P@ssw0rd /add` |
| Add user to Administrators (admin) | `net localgroup administrators newuser /add` |
| List Administrators | `net localgroup administrators` |
| Run a program as another user | `runas /user:otheruser cmd` |
| Apply Group Policy changes now | `gpupdate /force` |
| Show applied policies | `gpresult /r` |
| Full policy report (HTML) | `gpresult /h report.html` |
| List saved Windows credentials | `cmdkey /list` |
| Kerberos tickets (domain PCs) | `klist` |
| **Defender status** (PowerShell) | `Get-MpComputerStatus` |
| Update Defender signatures | `Update-MpSignature` |
| Defender quick / full scan | `Start-MpScan -ScanType QuickScan` · `FullScan` |
| Defender offline scan (reboots) | `Start-MpWDOScan` |
| Defender threat history | `Get-MpThreatDetection` |
| Check PowerShell script policy | `Get-ExecutionPolicy -List` |
| Allow your own local scripts to run | `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` |
| Unblock a downloaded script | `Unblock-File .\script.ps1` |
| Secure Boot enabled? (admin) | `Confirm-SecureBootUEFI` |

---

## 16. Registry from the Command Line

⚠ Always export a key before changing it.

| Task | Command |
|---|---|
| Read a key | `reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"` |
| Back up a key | `reg export "HKCU\Software\MyApp" backup.reg` |
| Restore a backup | `reg import backup.reg` (or double-click the .reg) |
| Add/change a value | `reg add "HKCU\Software\MyApp" /v Setting /t REG_DWORD /d 1 /f` |
| Delete a value | `reg delete "HKCU\Software\MyApp" /v Setting /f` |
| Browse registry in PowerShell | `Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"` |

**Useful keys**
| Key | What's there |
|---|---|
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | Your startup programs |
| `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` | All-users startup programs |
| `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion` | Windows version/build/install info |
| `HKLM\SYSTEM\CurrentControlSet\Services` | Service configuration |

Tip: in `regedit`, paste a path into the address bar at the top to jump straight to it.

---

## 17. Event Logs

| Task | Command |
|---|---|
| Open Event Viewer | `eventvwr.msc` |
| Last 20 errors in the System log (PowerShell) | `Get-WinEvent -LogName System -MaxEvents 200 \| Where LevelDisplayName -eq Error \| Select -First 20 TimeCreated, Id, ProviderName, Message` |
| Unexpected shutdowns / crashes | `Get-WinEvent -FilterHashtable @{LogName='System'; Id=41,6008} -MaxEvents 10` |
| Blue screens (BSOD) | `Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-WER-SystemErrorReporting'} -MaxEvents 10` |
| Boot & shutdown times | `Get-WinEvent -FilterHashtable @{LogName='System'; Id=6005,6006} -MaxEvents 10` |
| List all logs | `wevtutil el` |
| ⚠ Clear a log | `wevtutil cl Application` |
| Crash dump files | `C:\Windows\Minidump\` |
| Visual crash timeline | `perfmon /rel` (Reliability Monitor) |

**Common Event IDs:** 41 (unexpected reboot / power loss) · 6008 (unexpected shutdown) · 1074 (who/what initiated a shutdown) · 7000–7034 (service failures) · 4624/4625 (successful/failed logon, Security log) · 1000 (application crash).

---

## 18. PowerShell Essentials

### Learning & discovery
| Task | Command |
|---|---|
| Help with examples | `Get-Help Get-Process -Examples` |
| Download full help files (admin) | `Update-Help` |
| Find commands by keyword | `Get-Command *network*` |
| What properties/methods does output have? | `Get-Process \| Get-Member` |
| PowerShell version | `$PSVersionTable` |
| Command history | `Get-History` · search history with `Ctrl + R` |

### The pipeline (the core idea)
| Task | Example |
|---|---|
| Filter | `Get-Service \| Where-Object Status -eq 'Stopped'` |
| Pick columns | `Get-Process \| Select-Object Name, CPU` |
| Sort | `Get-Process \| Sort-Object CPU -Descending` |
| First N | `... \| Select-Object -First 5` |
| Count | `(Get-ChildItem).Count` |
| Loop | `Get-ChildItem *.txt \| ForEach-Object { $_.Name }` |
| Table / list view | `... \| Format-Table -AutoSize` · `... \| Format-List *` |
| **Interactive filterable grid** | `Get-Process \| Out-GridView` |
| Export to CSV (opens in Excel) | `Get-Process \| Export-Csv procs.csv -NoTypeInformation` |
| Export to JSON | `Get-Service \| ConvertTo-Json \| Out-File svc.json` |
| Copy to clipboard | `... \| Set-Clipboard` |

### Handy cmdlets
| Task | Command |
|---|---|
| Web request / download | `Invoke-WebRequest https://example.com -OutFile page.html` |
| Call a JSON API | `Invoke-RestMethod https://api.github.com` |
| Ping (PowerShell style) | `Test-Connection google.com -Count 4` |
| Windows optional features | `Get-WindowsOptionalFeature -Online \| Where State -eq Enabled` |
| Enable a feature (admin) | `Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux` |
| Devices with problems | `Get-PnpDevice \| Where Status -ne OK` |
| Installed Store apps | `Get-AppxPackage \| Select Name, Version` |
| Remove a Store app | `Get-AppxPackage *xbox* \| Remove-AppxPackage` |
| Measure how long a command takes | `Measure-Command { some-command }` |
| Your profile script (runs at every start) | `notepad $PROFILE` |

### Aliases (Linux/CMD-style names that work in PowerShell)
`ls`/`dir` → Get-ChildItem · `cd` → Set-Location · `cat`/`type` → Get-Content · `cp` → Copy-Item · `mv` → Move-Item · `rm`/`del` → Remove-Item · `pwd` → Get-Location · `ps` → Get-Process · `kill` → Stop-Process · `cls`/`clear` → Clear-Host · `iwr` → Invoke-WebRequest · `?` → Where-Object · `%` → ForEach-Object.

---

## 19. Developer & Other Built-in CLI Tools

| Tool | Common commands |
|---|---|
| **WSL** (Linux on Windows) | `wsl --install` · `wsl --list --online` · `wsl --install -d Ubuntu` · `wsl -l -v` (installed distros & versions) · `wsl --update` · `wsl --shutdown` · `wsl -d Ubuntu` · open Linux files in Explorer: `\\wsl$` |
| **ssh / scp** (built in) | `ssh user@host` · `ssh-keygen -t ed25519` · `scp file.txt user@host:/path/` |
| **curl** (built in) | `curl https://example.com` · `curl -O URL` (save file) · `curl -I URL` (headers only) |
| **tar** (built in) | `tar -xf archive.tar.gz` · `tar -a -cf out.zip folder` |
| **sudo** (Windows 11 24H2+) | enable in `ms-settings:developers` → "Enable sudo", then `sudo command` from a normal terminal |
| **Windows Sandbox** (Pro) | enable in `optionalfeatures`; throwaway VM for testing sketchy files |
| **Hyper-V** (Pro) | enable in `optionalfeatures`; `virtmgmt.msc` to manage VMs |
| **Windows Terminal** | `wt` · new tab `Ctrl + Shift + T` · split pane `Alt + Shift + +` / `-` · settings `Ctrl + ,` |
| **clip** | send any output to clipboard: `command \| clip` |
| **certutil** | hashes, Base64: `certutil -encode in.txt out.b64` |

---

## 20. Command-Line Tricks (CMD & PowerShell)

| Trick | How |
|---|---|
| Autocomplete paths/commands | `Tab` (press again to cycle) |
| Previous commands | `↑` / `↓` |
| Command history list (CMD) | `F7` · `doskey /history` |
| Search history (PowerShell) | `Ctrl + R`, type part of a command |
| Cancel running command | `Ctrl + C` |
| Clear screen | `cls` |
| Help for any CMD tool | `command /?` |
| Save output to a file (overwrite / append) | `command > out.txt` · `command >> out.txt` |
| Save errors too | `command > out.txt 2>&1` |
| Run second command only if first succeeds | `cmd1 && cmd2` |
| Run second command only if first fails | `cmd1 \|\| cmd2` |
| Pipe output into another command | `cmd1 \| cmd2` |
| Quote paths with spaces | `cd "C:\Program Files"` |
| Drag a file into the terminal | pastes its full path |
| Copy path of a file in Explorer | select it, `Ctrl + Shift + C` |
| Paste in terminal | `Ctrl + V` or right-click |
| Run a .ps1 script | `.\script.ps1` (see execution policy in section 15) |
| Run a .bat file | `script.bat` or double-click |

---

## 21. Boot, Recovery & Safe Mode

| Task | How |
|---|---|
| **Advanced Startup menu** (Safe Mode, System Restore, Startup Repair, uninstall updates, command prompt) | hold `Shift` while clicking Restart · or `shutdown /r /o /t 0` · or Settings → System → Recovery → Advanced startup |
| Safe Mode via Advanced Startup | Troubleshoot → Advanced options → Startup Settings → Restart → press `4` (Safe Mode) or `5` (with Networking) |
| Safe Mode via msconfig | `msconfig` → Boot tab → Safe boot (**uncheck it afterwards**) |
| ⚠ Force Safe Mode at next boot (admin) | `bcdedit /set {current} safeboot minimal` · undo: `bcdedit /deletevalue {current} safeboot` |
| Show boot configuration | `bcdedit` (admin) |
| Enter BIOS/UEFI | `shutdown /r /fw /t 0` or the key below at power-on |
| Reset this PC (keep or remove files) | `ms-settings:recovery` → Reset PC |
| Repair boot from recovery command prompt | `bootrec /fixmbr` · `bootrec /fixboot` · `bootrec /rebuildbcd` |
| Rebuild EFI boot files from recovery prompt | `bcdboot C:\Windows` |
| Automatic Repair loop fallback | interrupt boot 3 times (power off during logo) to force the recovery menu |

**Common BIOS / boot-menu keys at power-on** (varies by model):

| Brand | BIOS setup | One-time boot menu |
|---|---|---|
| Dell | `F2` | `F12` |
| HP | `Esc` → `F10` | `Esc` → `F9` |
| Lenovo | `F1` or `F2` (some: Novo button) | `F12` |
| ASUS | `F2` or `Del` | `F8` or `Esc` |
| Acer | `F2` or `Del` | `F12` |
| MSI | `Del` | `F11` |
| Gigabyte | `Del` | `F12` |
| Microsoft Surface | hold **Volume Up** + press Power | hold **Volume Down** + press Power |

---

## 22. Troubleshooting Recipes

### "Internet stopped working" (admin terminal)
```
ipconfig /release
ipconfig /renew
ipconfig /flushdns
netsh winsock reset
netsh int ip reset
```
Then reboot. If still broken: `ping 8.8.8.8` works but `ping google.com` fails → DNS problem (try setting DNS to 1.1.1.1 or 8.8.8.8 in `ncpa.cpl`).

### "Windows is buggy / files corrupted" (admin)
```
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow
chkdsk C: /scan
```
Reboot, then check `perfmon /rel` for recurring errors.

### "Windows Update is stuck" (admin)
```
net stop wuauserv
net stop bits
net stop cryptsvc
ren C:\Windows\SoftwareDistribution SoftwareDistribution.old
ren C:\Windows\System32\catroot2 catroot2.old
net start cryptsvc
net start bits
net start wuauserv
```
Then retry Windows Update. Also try `ms-settings:troubleshoot` → Windows Update troubleshooter.

### "PC is slow"
1. `Ctrl + Shift + Esc` → sort by CPU / Memory / Disk.
2. Task Manager → Startup apps → disable what you don't need.
3. `resmon` → Disk tab to find what's hammering the drive.
4. `cleanmgr` → "Clean up system files".
5. `Get-MpComputerStatus` / run a Defender scan.
6. Check disk health: `Get-PhysicalDisk`.

### "PC won't stay asleep / wakes randomly" (admin)
```
powercfg /lastwake
powercfg /devicequery wake_armed
powercfg /requests
```
Disable wake on the offending device in `devmgmt.msc` → device → Power Management tab.

### "Printer won't print" (admin)
```
net stop spooler
del /q /f /s %WINDIR%\System32\spool\PRINTERS\*
net start spooler
```

### "Something is using this port / file"
- Port: `netstat -ano | findstr :PORT` → `tasklist /fi "PID eq PID"`
- File locked: `resmon` → CPU tab → Associated Handles → search the file name.

### "I forgot my Wi-Fi password"
```
netsh wlan show profile name="NetworkName" key=clear
```

### "Is my laptop battery dying?"
```
powercfg /batteryreport
```
Compare **Design Capacity** vs **Full Charge Capacity**. Below ~70–80% is significant wear.

### "Why did my PC crash / reboot?"
1. `perfmon /rel` (Reliability Monitor).
2. Event Viewer → System log → Event IDs 41, 1001, 6008.
3. Check `C:\Windows\Minidump` (open dumps with WinDbg or BlueScreenView).

### "Did this download get tampered with?"
```
certutil -hashfile file.iso SHA256
```
Compare with the hash published on the official download page.

---

## 23. Recommended Free Tools (not built in)

All installable with winget (IDs in parentheses).

| Tool | What it's for |
|---|---|
| **PowerToys** (`Microsoft.PowerToys`) | App launcher, FancyZones window layouts, text extractor, file locksmith (who's locking this file), bulk rename, keyboard remapper |
| **Sysinternals Suite** (`Microsoft.Sysinternals.Suite`) | Process Explorer (supercharged Task Manager), Autoruns (every startup location), Process Monitor (file/registry activity), TCPView (live connections) |
| **Everything** (`voidtools.Everything`) | Instant filename search across all drives |
| **WizTree** (`AntibodySoftware.WizTree`) | See what's eating disk space, fast |
| **7-Zip** (`7zip.7zip`) | Archives of every format |
| **CrystalDiskInfo** (`CrystalDewWorld.CrystalDiskInfo`) | SSD/HDD SMART health |
| **HWiNFO** (`REALiX.HWiNFO`) | Hardware details, temperatures, sensors |
| **Notepad++** (`Notepad++.Notepad++`) | Better text editor |
| **Visual Studio Code** (`Microsoft.VisualStudioCode`) | Code/config editor |
| **Git** (`Git.Git`) | Version control |
| **PowerShell 7** (`Microsoft.PowerShell`) | Modern PowerShell |
| **Rufus** (`Rufus.Rufus`) | Make bootable USB drives |
| **ShareX** (`ShareX.ShareX`) | Advanced screenshots & screen recording |

---

*Commands reflect Windows 11 (24H2) and generally apply to Windows 10. Run commands marked ⚠ only when you understand exactly what they will change.*
