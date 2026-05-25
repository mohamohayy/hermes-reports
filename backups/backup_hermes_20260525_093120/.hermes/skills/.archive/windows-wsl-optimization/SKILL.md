---
name: windows-wsl-optimization
description: Diagnose and optimize Windows+WSL systems — disk, memory, startup items, cache cleanup.
summary: Comprehensive system health check and cleanup for Windows+WSL environments.
trigger: "User asks to 'optimize my computer', 'clean up my system', 'why is my PC slow', or mentions disk space / memory issues on Windows+WSL."
scope: "Windows 10/11 + WSL2. Diagnostic commands span both WSL (Linux) and Windows (PowerShell) sides."
---

## Why This Skill Exists

Users on Windows+WSL often ask to "optimize" their computer without knowing where the bottlenecks are. The agent needs to gather data from both sides (WSL and Windows), identify actual issues (not guess), and execute safe cleanups. This skill encodes the diagnostic checklist, cleanup commands, and the right script-generation pattern so every session avoids reinventing the workflow.

## Key Constraints & Pitfalls

### PowerShell inline commands from WSL bash — variable mangling

When passing PowerShell with `$variable` or `$_` via `powershell.exe -Command "..."`, bash expansion corrupts the command. **Always write `.ps1` files to the Desktop and run with `-File`.** See `references/powershell-from-wsl.md`.

### Chinese Windows encoding

Chinese Windows uses GBK (codepage 936):
- **`.bat` files**: Must be saved as GBK, not UTF-8. `write_file` writes UTF-8 — convert with `iconv -f UTF-8 -t GBK` after writing.
- **`.ps1` files**: Use **pure ASCII/English** to avoid garbled Chinese. UTF-8 with BOM sometimes works but is unreliable from WSL.
- **Python subprocess fallback for .ps1**: When the `write_file` → `terminal()` → `powershell.exe -File` chain keeps failing due to encoding (parser errors on Chinese text), use `execute_code` + `subprocess.run` instead:
  ```python
  import subprocess, os
  ps1_path = os.path.expanduser("~/.hermes/script.ps1")
  with open(ps1_path, "w", encoding="utf-8") as f:
      f.write(content_utf8)
  # Convert to GBK for PowerShell
  with open(ps1_path, "rb") as f:
      utf8_data = f.read()
  gbk_path = ps1_path.replace(".ps1", "_gbk.ps1")
  with open(gbk_path, "wb") as f:
      f.write(utf8_data.decode("utf-8").encode("gbk", errors="replace"))
  r = subprocess.run(
      ["powershell.exe", "-ExecutionPolicy", "Bypass", "-File", gbk_path],
      capture_output=True, text=True, timeout=120
  )
  print(r.stdout, r.stderr)
  ```
  This is more reliable than `terminal()` calling `powershell.exe` because `terminal()` may have its own encoding/working-directory issues.
- **`.sh` scripts** (WSL terminal): Chinese annotations are fine — WSL is UTF-8 native.

### UTF-16LE output from wsl.exe

`wsl.exe --version` and similar output UTF-16LE — garbled in WSL terminal. Use `/proc/version` or `uname -r` on WSL side instead.

### Sudo required for WSL cleanup

`journalctl --vacuum-time=7d` and `apt-get clean` require sudo. Without password, these fail silently or partially (journald can clean `/run/log/journal` but not `/var/log/journal`). Fall back to script-on-Desktop pattern.

### Windows startup item removal requires admin

`Remove-CimInstance Win32_StartupCommand` needs Administrator. The `.ps1` must be run from an elevated PowerShell session.

## Workflow

### Phase 1: Gather system data (both sides, parallel)

**WSL side:**
```bash
# Disk
df -h / /mnt/c /mnt/d

# Memory
free -h

# CPU
nproc

# Top processes
ps aux --sort=-%mem | head -11

# Cache sizes
journalctl --disk-usage
du -sh /var/cache/apt/archives/ ~/.cache/ ~/.hermes/logs/ ~/.hermes/ 2>/dev/null
```

**Windows side (write .ps1 to Desktop, run with -File):**
```powershell
# Disk
Get-PSDrive C, D | ForEach-Object { ... }

# Top memory processes
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10 | ForEach-Object { ... }

# Startup items
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command
```

### Phase 2: Analyze and report

Present findings as a table showing:
- Each resource (disk partition, memory, processes)
- Current state with metrics
- Severity indicator (OK / warning / critical)
- Recommended action if any

Thresholds:
- Disk usage > 75% → warning
- Disk usage > 90% → critical
- Memory usage within limit → OK

### Phase 3: Execute safe cleanups (no password needed first)

```bash
# These work without sudo:
journalctl --vacuum-time=7d           # partial cleanup
journalctl --disk-usage               # verify
```

### Phase 4: Generate scripts for sudo/admin tasks

For cleanups requiring elevated privileges, write scripts to `/mnt/c/Users/<winuser>/Desktop/`:

- `optimize_wsl.sh` — journald vacuum (sudo), apt-get clean (sudo), autoremove
- `manage_startup.ps1` — Remove-CimInstance for unwanted startup items (pure ASCII to avoid encoding issues)

User reviews, then runs manually. This is safer than the agent executing sudo commands directly.

### Phase 5: Verify results

```bash
# Re-check cleaned items
journalctl --disk-usage
du -sh /var/cache/apt/archives/
```

## Common Cleanup Targets

| Target | Command | Safety | Typical Savings |
|--------|---------|--------|-----------------|
| journald logs | `sudo journalctl --vacuum-time=7d` | Safe | 200-500MB |
| journald logs | `sudo journalctl --vacuum-size=100M` | Safe | Varies |
| APT cache | `sudo apt-get clean` | Safe | 50-150MB |
| Unused packages | `sudo apt-get autoremove -y` | Safe | Varies |
| WSL VHD compact | `diskpart` + `compact vdisk` | Needs WSL shutdown first | Varies |

## Windows C: Disk Cleanup Protocol

For users asking "clean up C disk" or "free up C space" specifically. This is an entirely Windows-side operation.

### Pre-cleanup: Measure

Write a .ps1 to Desktop with ASCII-only content. PowerShell from execute_code (subprocess.run) is more reliable than terminal() for this.

```powershell
# Disk overview
$disk = Get-PSDrive C | Select-Object Used,Free
"Total: {0:N2} GB" -f (($disk.Used + $disk.Free) / 1GB)
"Used: {0:N2} GB" -f ($disk.Used / 1GB)
"Free: {0:N2} GB" -f ($disk.Free / 1GB)

# User directories
$paths = @("$env:LOCALAPPDATA", "$env:TEMP", "$env:USERPROFILE\Downloads", "$env:USERPROFILE\Desktop")
foreach ($p in $paths) { ... Measure-Object ... }

# Browser caches
$chrome = "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Cache"
$edge = "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\Cache"

# Program files by folder size
Get-ChildItem "C:\Program Files" -Force -ErrorAction SilentlyContinue | Where-Object PSIsContainer | ForEach-Object { ... }

# Large AppData folders (>500MB)
Get-ChildItem "$env:LOCALAPPDATA" -Force | Where-Object PSIsContainer | ForEach-Object { ... }
```

### Safe Cleanup Sequence (all non-destructive)

Order matters — do these in sequence via separate .ps1 scripts:

**Phase A — Safe auto-clean (no sudo/admin needed):**
```
1. C:\Windows\Temp\*                    — Remove-Item -Recurse
2. C:\Windows\Prefetch\*               — Remove-Item -Recurse (safe to clear)
3. C:\Windows\SoftwareDistribution\Download\* — Remove-Item -Recurse (update cache, auto-redownloads)
4. %TEMP%\*                            — Remove-Item -Recurse -Force
5. Browser caches:
   - Chrome:  %LOCALAPPDATA%\Google\Chrome\User Data\Default\Cache\*
   - Edge:    %LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Cache\*
   - Chome Code Cache + Edge Code Cache (similar path, /Code Cache/ subdir)
6. %LOCALAPPDATA%\Microsoft\Windows\Explorer\thumbcache_*.db, iconcache_*.db
7. %APPDATA%\Microsoft\Windows\Recent\*
8. C:\Windows\Logs\CBS\*               — Remove-Item (CBS logs can be 50-200MB)
9. C:\Windows\Logs\DISM\*              — Remove-Item
```

**Phase B — Windows Disk Cleanup tool:**
```powershell
# Run cleanmgr with saved settings (sagerun:1 must be pre-configured, or just fire it)
Start-Process -FilePath "cleanmgr.exe" -ArgumentList "/sagerun:1" -NoNewWindow -Wait
```
If sagerun isn't configured, skip this step or run cleanmgr interactively.

**Phase C — Component Store (DISM, needs admin):**
```powershell
dism /online /Cleanup-Image /StartComponentCleanup /ResetBase
```
Only do this if user explicitly asks or if C: is critically full (>90%). It's slow (several minutes).

### What NOT to delete

- `C:\Windows\assembly\` — .NET Native Images, system-critical
- `C:\Windows\WinSxS\` — Windows component store. Use DISM, not manual deletion
- `C:\ProgramData\*` without checking — many apps store data here
- `C:\Users\<user>\AppData\Local\Microsoft\Windows\INetCache\` without checking — IE/Edge legacy cache, sometimes actively in use
- `C:\Windows\Installer\` — MSI cache, removing breaks app uninstall

### Post-cleanup Verification

Re-run the disk overview .ps1 to report before/after delta.

### Reporting Format

Present results as a clean table with action, space freed, and current disk overview. Keep it brief — user wants numbers, not narrative.

### User Told You to "Just Do It"

When the user says "你来着来", "你来处理", "你看着办", or "搞", **stop asking for permission on individual items**. You already showed the plan in your initial analysis. Proceed to execute each phase immediately. Only ask if something is one-way irreversible (formatting, deleting user data you haven't identified). The user trusts the analysis you already presented — don't make them confirm each step.

### Also Clean User Downloads & Desktop Installers

After system temp cleanup, check and remove:
- `%USERPROFILE%\Downloads\*.exe`, `*.msi`, `*.zip`, `*.rar`, `*.7z`, `*.iso` — especially old installer archives
- `%USERPROFILE%\Desktop\*.exe` — standalone installer files left on desktop
- Large folders on Desktop that look like accidental extractions (e.g. "新建文件夹", "3")
- WeChat installer files (`WeChatWin_*.exe`) — user likely already installed the app
- `%LOCALAPPDATA%\pip\cache\*` — pip cache, often 500MB+
- `%APPDATA%\npm-cache\*` — npm cache if present

**Check first, then delete.** Use `Get-ChildItem` to list items with sizes before removing. Delete installers upfront (they're safe to remove); for unknown folders, list them and ask only if >50MB with unclear contents.

```
Before: 191.95 GB used / 123.61 GB free
After:  190.30 GB used / 125.26 GB free
Freed:  ~1.65 GB

Breakdown:
  Sys Temp+Pfetch+UpdateCache:  ~200 MB
  CBS/DISM logs:                43 MB
  Chrome cache+code:            234 MB
  Edge cache+code:              447 MB
  Other (thumbnails, recent):   minor
  Windows Disk Cleanup:         further reclaim
```

If freed space is under 2 GB and C: is still >60% used, note the largest folders the user can manually investigate (WSL vdisk, pip cache, installed apps, Downloads/Desktop contents).

## Cleanup After Session

After scripts are executed, offer to delete them from Desktop (`optimize_wsl.sh`, `manage_startup.ps1`, diagnostic `.ps1`, `clean_step*.ps1`).

## See Also

- `windows-wsl-autostart` — for startup configuration and WSL/Windows interop quirks
- `hermes-agent` — for gateway platform setup (WeChat, Telegram, etc.)
- `references/session-2026-05-15.md` — full diagnostic run from a real session with findings and pitfalls
- `references/china-network-workarounds.md` — GitHub/pip/npm acceleration from mainland China
