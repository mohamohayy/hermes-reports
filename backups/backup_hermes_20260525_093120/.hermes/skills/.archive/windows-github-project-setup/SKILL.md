---
name: windows-github-project-setup
description: "Install Windows Python projects from GitHub when in China — browser download, winget Python, bat installer with Tsinghua mirror."
version: 1.0.0
---

# Windows GitHub Project Setup (China)

Workflow for installing Windows-native Python projects from GitHub when CLI access is blocked/slow in China.

## When to Use

- Installing any Windows Python project from GitHub
- GitHub clone/curl times out from WSL or Windows CLI
- Project needs Windows-native packages (pywin32, pywinauto, uiautomation, etc.)
- User is on Chinese Windows (GBK encoding)

## Workflow

### Step 1: Browser Download

**DO NOT** attempt CLI git clone or curl from WSL. Every mirror tested in China fails consistently:
- `git clone` directly → timeout after 120s
- `ghproxy.net` → HTTP/2 PROTOCOL_ERROR (err 1)
- `gitclone.com` → 403 Forbidden
- `hub.gitmirror.com` → resolving timed out
- `codeload.github.com` (zip) → partial progress (3-6MB) then curl: (28) timeout after 300s

The ONLY reliable method: user downloads via Windows browser and saves to Desktop.

1. Give user the direct download link: `https://github.com/<owner>/<repo>/archive/refs/heads/main.zip`
2. User downloads via Windows browser (always faster)
3. User saves to Desktop: `C:\Users\liu\Desktop\`

### Step 2: Python Setup

Check if Windows Python exists. If not, install via winget:

```bash
# Check via WSL
powershell.exe -ExecutionPolicy Bypass -File check_python.ps1

# Install if missing (may exceed 120s terminal timeout — Python installs anyway)
powershell.exe -Command "winget install Python.Python.3.11 --accept-package-agreements --accept-source-agreements"
```
Winget install of Python frequently exceeds the default 120s terminal timeout
but completes successfully in the background. After the command appears to
time out, wait ~30 seconds and re-run `check_python.ps1` to confirm.

Python installs to: `%LOCALAPPDATA%\\Programs\\Python\\Python311\\python.exe`

The `python` command may NOT be in PATH immediately after winget install (new terminal needed). Always use full path in scripts.

### Step 3: Create Installer Bat

Create a `.bat` installer on Desktop. Key rules:

1. **Hardcode Python path** — don't rely on PATH:
   ```bat
   set PYTHON=%LOCALAPPDATA%\Programs\Python\Python311\python.exe
   ```
2. **Use Tsinghua mirror** for all pip installs:
   ```bat
   %PYTHON% -m pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
   ```
3. **GBK encode** the bat file (Chinese Windows CMD = GBK):
   ```bash
   iconv -f UTF-8 -t GBK /tmp/script.bat > /tmp/script_gbk.bat
   mv /tmp/script_gbk.bat "/mnt/c/Users/liu/Desktop/script.bat"
   ```
4. Handle GitHub zip naming: zip extracts to `<repo>-main/`, rename to `<repo>/`

### Step 4: User Executes

User double-clicks the `.bat` on Desktop. Script:
- Extracts zip via PowerShell `Expand-Archive`
- Installs pip dependencies (Tsinghua mirror)
- Prepares config files

## Bat Template

See `templates/install_project.bat` for a reusable project installer template.

## Pitfalls

- **Never use cmd.exe /c start** from WSL — UNC path issues. Use `powershell.exe -Command` or `-File`.
- **Python PATH not refreshed** after winget install — hardcode the full path in scripts.
- **Winget install times out after 120s** but Python installs successfully in the background. Wait 30s and re-run check_python.ps1 to confirm.
- **pywin32/pywinauto/uiautomation** are Windows-only — must install under Windows Python, not WSL Python.
- **PowerShell .ps1 with non-ASCII chars** garbles on Chinese Windows — use ASCII-only in .ps1 files.
- **write_file writes UTF-8** — bat files MUST be iconv-converted to GBK before Windows use.

## Check Python Script

See `scripts/check_python.ps1` — finds Python in common install locations regardless of PATH.

## References

- `references/ufo-deepseek-config.md` — UFO agents.yaml DeepSeek configuration example
