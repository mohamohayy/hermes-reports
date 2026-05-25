# WSL → PowerShell Interop Pitfalls

## Error 1: Variable Mangling in Inline Commands

**Cause**: Bash expands `$_`, `$var`, `$t` before PowerShell sees them.

**Symptom** (from real session):
```
所在位置 行:1 字符: 164
+ ... | Select-Object -First 10 Name, @{N='MemMB';E={[math]::Round(/home/li ...
+                                                                  ~
方法调用中缺少“)”。
```
Note how `$_.WorkingSet64` became `/home/liu123456/.hermes/hermes-agent.WorkingSet64` — bash replaced `$_` with the last argument of the previous command, then concatenated the rest.

**Fix**: Always write `.ps1` to disk and invoke with `-File`:
```bash
# Write script
cat > /mnt/c/Users/liu/Desktop/check.ps1 << 'PSEOF'
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10 | ForEach-Object {
    $mem = [math]::Round($_.WorkingSet64 / 1MB, 1)
    Write-Host "$($_.Name) - ${mem}MB"
}
PSEOF

# Run
powershell.exe -ExecutionPolicy Bypass -File "C:\Users\liu\Desktop\check.ps1"
```

Note the `<< 'PSEOF'` (single-quoted delimiter) — prevents bash from expanding `$()` inside the heredoc.

## Error 2: Chinese Output Garbled Despite UTF8 Setting

Even with `[Console]::OutputEncoding = [Text.Encoding]::UTF8`, Chinese characters in PowerShell output captured from WSL may display as garbled (e.g., `纾佺洏` instead of `磁盘`). 

**Root cause**: WSL's PTY and the PowerShell console host negotiate encoding independently of `OutputEncoding`. The actual bytes on stdout may be GBK (codepage 936) regardless.

**Workaround**: Accept that Chinese PowerShell output from WSL will be garbled. Either:
- Use English-only column headers in the PowerShell script
- Or parse numeric fields only and ignore text labels

## Error 3: `wsl.exe --version` UTF-16LE Output

```
W S L  版本:  2 . 6 . 3 . 0
```
Every other byte is NUL. `wsl.exe` uses UTF-16LE regardless of console codepage.

**Fix**: Use WSL-side equivalents:
```bash
uname -r              # kernel version
cat /proc/version     # full version string
wslpath -w /          # quick check that WSL interop works
```
