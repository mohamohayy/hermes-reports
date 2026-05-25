---
name: windows-wsl-autostart
description: Set up Windows Task Scheduler to launch Hermes Agent (or any CLI tool) in WSL automatically on user logon.
summary: Configure automatic startup of Hermes Agent (or any CLI tool) in WSL on Windows login using Task Scheduler.
trigger: "User wants Hermes (or another CLI agent) to launch automatically when Windows starts or they log in."
scope: "Windows 10/11 + WSL2. Applies to any CLI tool requiring terminal UI or background service behavior in WSL."
---

## Why This Skill Exists

WSL has no native systemd or init system that starts on Windows boot. The only reliable, user-session-aware mechanism is Windows Task Scheduler with a `LogonTrigger`. This skill codifies the minimal, secure, and portable pattern.

## Key Constraints & Pitfalls

- ❌ `wsl --shutdown` or WSL idle timeout may terminate the session before the task runs → always use `wsl -e bash -c "..."`, not `wsl -d <distro> ...`
- ❌ `~` in WSL tasks expands to root's home unless explicitly run as the target user → always use absolute paths like `/home/<user>/...`
- ⚠️ `LogonTrigger` only fires on *interactive* logon → does NOT work for auto-login or lock-screen resume
- ✅ Use `RunLevel=HighestAvailable` (or `schtasks /rl highest`) — required for GUI terminal access (e.g., Windows Terminal)
- ✅ Always wrap startup logic in a `.sh` script — avoids quoting hell and enables logging/debugging

### PowerShell version incompatibilities (real, observed failures)

- ❌ `New-ScheduledTaskTrigger -AtLogOn -Delay` — `-Delay` parameter doesn't exist on older PowerShell/Win10. Skip it; the 10s delay is not critical.
- ❌ `New-ScheduledTaskSettingsSet -ExecutionTimeLimit` — same, missing on older versions. Omit the entire `-Settings` parameter and rely on defaults.
- ❌ `.ps1` self-elevation via right-click "Run with PowerShell" is **unreliable** — the elevated child window often flashes closed before user can read output. Never use self-elevating `.ps1` as the primary delivery format.
- ✅ Use `.bat` + `schtasks` as the primary installation method. `schtasks` is a flat EXE (not a PowerShell cmdlet), works identically on Win10/Win11, and `.bat` handles elevation cleanly.

### Chinese Windows encoding (observed failure)

Chinese Windows uses **GBK (codepage 936)** as the default console code page. UTF-8 `.bat` files display garbled characters (乱码) and `chcp 65001` is unreliable across versions. Three safe approaches:
1. **Pure ASCII** — no Chinese characters. CMD reads ASCII fine regardless of codepage. Recommended for delivery.
2. **Save as ANSI/GBK from Notepad** — user can re-save with encoding dropdown to ANSI.
3. **Write from Python with `encoding='gbk'`** — reliable when scripting from WSL.

⚠️ `iconv -f UTF-8 -t GBK` from WSL can produce artifacts that break CMD parsing. Avoid.
See `references/windows-encoding.md` for full details.

### WSL to Windows interop: UNC path issue

When invoking `cmd.exe` from inside WSL, the CWD is a UNC path (`\\\\wsl.localhost\\...`). CMD errors with "UNC paths are not supported". Use `powershell.exe` instead:

```bash
# BROKEN from WSL:
cmd.exe /c "schtasks /run /tn \\Hermes\\..."

# WORKS from WSL:
powershell.exe -Command "Start-ScheduledTask -TaskPath '\\Hermes\\' -TaskName '...'"
```

### PowerShell inline commands from WSL: variable mangling (observed failure, 3 hits/session)

When passing PowerShell scripts as inline `-Command "..."` arguments from bash, the shell eats or corrupts PowerShell variables (`$_`, `$var`, `$t`, etc.). The inline string undergoes bash expansion first, then PowerShell receives the broken result. **Never pass inline PowerShell with `$_` or variable assignments from WSL bash.** Always write the script to a `.ps1` file on the Windows filesystem and invoke with `-File`:

```bash
# BROKEN — $_ gets replaced by bash, $t becomes empty, variables vanish:
powershell.exe -Command "Get-Process | ForEach-Object { $_.Name }"

# WORKS — write to file first, then run:
powershell.exe -ExecutionPolicy Bypass -File "C:\\Users\\liu\\Desktop\\script.ps1"
```

See `references/wsl-powershell-interop.md` for full error transcripts and encoding details.

### UTF-16LE output from wsl.exe

`wsl.exe --version` and `wsl.exe --list` use UTF-16LE encoding. Piped into a WSL terminal, every other byte is a NUL, producing garbled output. For version checks, prefer inspecting `/proc/version` or `uname -r` on the WSL side instead.

## Two Startup Patterns

### Pattern A: TUI Only (original)

Dashboard/TUI launches, but **no messaging platforms** (WeChat, Telegram, etc.) are connected — the gateway isn't running.

### Pattern B: Gateway + Dashboard (recommended for messaging)

Gateway starts **first** in background, waits for health check, then dashboard launches. This ensures messaging platforms connect automatically on boot.

**Why order matters:** The gateway must be running and have platform connections established before the user starts chatting. If the dashboard launches without the gateway, incoming WeChat/Telegram messages get queued by the platform but never delivered until `hermes gateway run` is manually executed. Starting gateway first eliminates this gap.

## Workflow

### Phase 1: Create startup script (WSL side)

1. **Determine paths**: Find Hermes venv path (`which hermes` → `readlink -f $(which hermes)` to trace) and WSL username (`whoami`).
2. **Choose pattern**:
   - **Pattern A (TUI only)**: Copy `templates/startup.sh` → `~/.hermes/startup.sh`
   - **Pattern B (Gateway + Dashboard)**: Copy `templates/startup-gateway.sh` → `~/.hermes/startup.sh`
3. Edit `VENV_PATH` to match, verify `GATEWAY_PORT` (default 8648). Make executable: `chmod +x ~/.hermes/startup.sh`.
4. **Test manually**: `bash ~/.hermes/startup.sh` — should launch dashboard with gateway ready. Check `~/.hermes/logs/startup.log` and `~/.hermes/logs/gateway-autostart.log` if it doesn't.

**Pattern B startup.sh logic:**
```bash
# Step 1: Start gateway in background (no-op if already running)
nohup hermes gateway run > "$HOME/.hermes/logs/gateway-autostart.log" 2>&1 &

# Step 2: Wait for gateway health check (up to 10s)
for i in $(seq 1 10); do
    if curl -s http://127.0.0.1:8648/health >/dev/null 2>&1; then
        echo "Gateway ready (port 8648)"
        break
    fi
    sleep 1
done

# Step 3: Launch dashboard (replaces shell process)
exec hermes dashboard --tui
```

**Pitfall:** `set -euo pipefail` is active — the `curl` health check inside an `if` condition is exempt from `set -e` failures, so the loop completes gracefully even if the gateway takes longer than 10s to start. Dashboard still launches.

### Phase 2: Register Windows scheduled task (primary: `.bat` + `schtasks`)

4. **Generate `.bat` installer**: Copy `templates/install-autostart.bat` to the Windows Desktop (`/mnt/c/Users/<winuser>/Desktop/`). Edit `STARTUP_SCRIPT` path if different from `/home/liu123456/.hermes/startup.sh`.
5. **Run**: Double-click the `.bat` file on Windows Desktop. It auto-elevates, registers the task via `schtasks`, and stays open showing result.
6. **Verify**: Task Scheduler GUI → `\Hermes\Hermes Agent Auto Start` → right-click → Run. A WSL terminal window should open showing Hermes TUI.

### Phase 3: Alternative (XML import, fallback only)

If `schtasks` is unavailable (rare), save `templates/task-scheduler.xml` to Desktop and import via PowerShell **run as Administrator** (not self-elevating):

```powershell
$xml = Get-Content "$env:USERPROFILE\Desktop\hermes-autostart.xml" -Raw
Register-ScheduledTask -Xml $xml -TaskName "Hermes Agent Auto Start" -TaskPath '\Hermes\' -Force
```

Note: single quotes around `'\Hermes\'` to avoid escaping issues with backslash in double-quoted strings.

## Removal

To remove the auto-start task:

1. **Write `remove-autostart.bat`** — copy `templates/remove-autostart.bat` to the Windows Desktop (`/mnt/c/Users/<winuser>/Desktop/`). Pure ASCII, self-elevating, uses `schtasks /delete`.
2. **Run** — double-click the `.bat` file. It auto-elevates, deletes the task, and stays open showing the result.

### Pitfall: PowerShell `Unregister-ScheduledTask` from WSL

Avoid calling `Unregister-ScheduledTask` directly from WSL — backslash path quoting (`-TaskPath '\Hermes\'`) interacts badly with bash escaping and often fails with `ParameterBindingException`. Use `schtasks /delete` via a `.bat` script instead.

## Verification

- ✅ Task status = "Ready" and "Last Run Time" updates after manual trigger
- ✅ WSL terminal window opens showing Hermes TUI
- ✅ Startup log at `~/.hermes/logs/startup.log` captures timestamps and any errors
- ❌ If `.bat` flashes and closes: right-click → "Run as Administrator" instead of double-click
- ❌ If blank terminal opens: check `startup.sh` permissions (`chmod +x`) and `VENV_PATH`
- ❌ If task shows "Running" but no window: check `schtasks` used `/rl highest`

## Chaining Multiple Services in startup.sh

The startup script can launch multiple services in sequence. Pattern (order matters):

```bash
# 1. Recover Docker containers (Docker auto-starts via systemd)
docker start one-api 2>/dev/null || true

# 2. Start Hermes Gateway (runs in background)
nohup hermes gateway run > "$HOME/.hermes/logs/gateway-autostart.log" 2>&1 &
# Wait for health check
for i in $(seq 1 10); do
    curl -s http://127.0.0.1:8648/health >/dev/null 2>&1 && break
    sleep 1
done

# 3. Launch Dashboard (runs in foreground — must be LAST)
exec hermes dashboard --tui
```

**Key points:**
- `exec` must be the LAST command (replaces the shell process)
- Background services use `nohup ... &` before the `exec`
- Docker containers use `docker start` (not `docker run`) for existing containers
- Always `|| true` on recovery commands — don't fail if service already running
- Check Docker systemd status first: `sudo systemctl is-enabled docker`

## Support Files

- `templates/startup.sh` — WSL-side startup script (Pattern A: TUI only)
- `templates/startup-gateway.sh` — WSL-side startup script (Pattern B: Gateway + Dashboard, recommended for messaging)
- `templates/install-autostart.bat` — Windows one-click installer using `schtasks` (pure ASCII)
- `templates/task-scheduler.xml` — Fallback XML task definition for `Register-ScheduledTask`
- `references/task-scheduler-notes.md` — WSL path gotchas, UNC path issue, debugging tips
- `references/windows-encoding.md` — Chinese Windows GBK/UTF-8 encoding pitfalls

## Post-Install: Dashboard UI Language

If using `hermes dashboard --tui`, set UI language to Chinese (or any locale):

```bash
hermes config set display.language zh    # Chinese
hermes config set display.language en    # English
```

Restart dashboard after changing: `hermes dashboard --stop && hermes dashboard --tui`

## See Also

- `hermes-agent`: for configuring `hermes --tui` behavior and defaults
- `wsl-config`: for WSL distro initialization and default user setup
