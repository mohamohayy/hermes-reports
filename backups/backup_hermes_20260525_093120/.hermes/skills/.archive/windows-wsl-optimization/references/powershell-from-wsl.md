# PowerShell from WSL — Patterns & Pitfalls

## The Golden Rule

**Never use inline PowerShell from WSL bash.** Write `.ps1` files to disk, then run with `-File`. Inline commands break in at least three ways simultaneously (bash expansion, encoding, UNC paths).

## Correct Pattern

```bash
# 1. Write .ps1 to Windows Desktop (safe, user-reviewable)
cat > /mnt/c/Users/liu/Desktop/my_script.ps1 << 'PSEOF'
# Use single-quoted heredoc delimiter to prevent bash expansion of $()
$processes = Get-Process | Where-Object { $_.ProcessName -like '*target*' }
Write-Host "Found: $($processes.Count) processes"
PSEOF

# 2. Run with Bypass + File
powershell.exe -ExecutionPolicy Bypass -File "C:\Users\liu\Desktop\my_script.ps1"
```

## Why Inline Breaks

### 1. Bash Variable Expansion
`$_`, `$var`, `$t`, `$(...)` get eaten by bash before PowerShell sees them:
```bash
# ❌ $_ expands in bash
powershell.exe -Command "Get-Process | Where-Object { $_.ProcessName -like '*x*' }"

# ❌ $(Get-Date) runs in bash, not PowerShell
powershell.exe -Command "Write-Host $(Get-Date)"
```

### 2. Encoding (Chinese Windows)
Chinese Windows uses GBK (codepage 936). UTF-8 `.ps1` with Chinese characters displays garbled output. Write **ASCII-only** `.ps1` files.

### 3. UNC Path Issues (cmd.exe)
`cmd.exe /c` from WSL runs with UNC working directory (`\\wsl.localhost\...`). Use `powershell.exe -Command` or `-File` instead of `cmd.exe`.

## Common Tasks

### Open a file with default Windows app
```bash
# ✅ Use Invoke-Item (uses file association)
powershell.exe -Command "Invoke-Item 'C:\Users\liu\Desktop\file.pptx'"

# ❌ Start-Process with guessed app name (often fails)
powershell.exe -Command "Start-Process 'wps' -ArgumentList '...'"
```
`Invoke-Item` is the equivalent of double-clicking — always preferred.

### Kill Windows processes
```bash
cat > /mnt/c/Users/liu/Desktop/kill_process.ps1 << 'PSEOF'
$target = Get-Process | Where-Object { $_.ProcessName -like '*Pattern*' }
if ($target) {
    $target | ForEach-Object {
        Write-Host "Closing: $($_.ProcessName) (PID: $($_.Id))"
        Stop-Process -Id $_.Id -Force -ErrorAction SilentlyContinue
    }
    Write-Host "Done."
} else {
    Write-Host "Process not found."
}
PSEOF
powershell.exe -ExecutionPolicy Bypass -File "C:\Users\liu\Desktop\kill_process.ps1"
```

### List processes by name
```powershell
Get-Process | Where-Object { $_.ProcessName -like '*WeChat*' } | Select-Object ProcessName, Id
```

## WeChat/Weixin Specifics
WeChat on Windows runs as multiple processes:
- `Weixin.exe` — main application (user-facing)
- `WeChatAppEx.exe` — multiple subprocess instances (mini-programs, webviews)

Kill all to fully close.

## WSL wsl.exe Output Encoding
`wsl.exe --version` outputs UTF-16LE — garbled in WSL terminals. Use `/proc/version` or `uname -r` on the WSL side instead.
