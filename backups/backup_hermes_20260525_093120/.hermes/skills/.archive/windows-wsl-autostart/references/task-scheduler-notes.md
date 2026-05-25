# Windows Task Scheduler Notes for WSL

## Critical Requirements

- `LogonTrigger` only works for *interactive* user logons (not auto-login or lock-screen resume)
- `Principal/RunLevel` must be `HighestAvailable` — otherwise GUI terminal (e.g., Windows Terminal) fails to launch
- `Principal/LogonType` must be `InteractiveToken`, not `InteractiveTokenOrPassword`

## WSL Path Gotchas

- `wsl -e bash -c "..."` is safer than `wsl -d <distro> ...` — ensures correct default user and shell
- Never rely on `~` in task actions — always use absolute paths like `/home/<user>/...`
- WSL distro name (`-d`) is optional if default distro is set; prefer omitting it for portability

## UNC Path Issue (WSL → Windows interop)

When running `cmd.exe` from inside WSL, the current directory is a UNC path (`\\wsl.localhost\...`). CMD refuses to run with:

```
UNC paths are not supported. Defaulting to Windows directory.
```

**Fix**: Use `powershell.exe` instead of `cmd.exe` when invoking Windows commands from WSL:

```bash
# BROKEN: cmd.exe inherits UNC path
cmd.exe /c "schtasks /run /tn \Hermes\Hermes Agent Auto Start"

# WORKS: PowerShell doesn't care about UNC CWD
powershell.exe -Command "Start-ScheduledTask -TaskPath '\Hermes\' -TaskName 'Hermes Agent Auto Start'"
```

## Debugging Tips

- Check task history in Task Scheduler → right-click task → "Properties" → "History" tab
- Logs appear in `~/.hermes/logs/agent.log` (if Hermes starts but doesn't show UI)
- Startup script logs to `~/.hermes/logs/startup.log`
- Test script manually first: `wsl -e bash -c "/home/liu123456/.hermes/startup.sh"`
- From WSL, trigger task test: `powershell.exe -Command "Start-ScheduledTask -TaskPath '\Hermes\' -TaskName 'Hermes Agent Auto Start'"`
