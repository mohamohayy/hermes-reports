# Native Windows Support (Early Beta)

Added in v0.14.0 (v2026.5.16). Hermes runs natively on `cmd.exe` / PowerShell without WSL.

## Install

```powershell
pip install hermes-agent
hermes
```

The PowerShell installer auto-handles MinGit download and Microsoft Store Python stub detection.

## WSL vs Native Comparison

| | WSL | Windows Native (Beta) |
|---|---|---|
| Install | clone repo → manual venv | `pip install hermes-agent` |
| Dependencies | WSL自带 git/python | Auto-downloads MinGit |
| Desktop control | Needs `powershell.exe` bridge, `/mnt/c/` path translation | Direct Windows access (pyautogui, screenshot, mouse) |
| File paths | Two filesystems, constant conversion | Unified `C:\Users\...` |
| Performance | Virtualization overhead, slow cross-fs I/O | Native, faster |
| Maturity | Stable | Early beta, 40+ fixes merged |

## Status

- Early beta — works end-to-end on clean Windows boxes
- ~40 Windows-specific fixes shipped in the v0.14.0 window
- Can coexist with WSL Hermes — separate installs, profiles if needed
