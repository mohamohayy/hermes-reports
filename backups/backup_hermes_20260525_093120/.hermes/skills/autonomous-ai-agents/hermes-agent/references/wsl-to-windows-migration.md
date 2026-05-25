# WSL → Windows Native Migration Walkthrough

Exact steps from a successful May 2026 migration. Both sides: Hermes v0.14.0, DeepSeek/v4-pro provider.

## Prerequisites

- WSL Hermes fully updated (`hermes update --yes`)
- Windows has Python 3.11+ with pip
- Same weixin account token (can't run both gateways simultaneously)

## Step-by-Step

### 1. Install Windows native Hermes

```powershell
# In Windows PowerShell (not WSL):
pip install hermes-agent -i https://pypi.tuna.tsinghua.edu.cn/simple
```

Verify: `hermes --version` → v0.14.0+

### 2. Copy config (from WSL terminal)

```bash
# Model
powershell.exe -Command "hermes config set model.default deepseek-v4-pro"
powershell.exe -Command "hermes config set model.provider deepseek"
powershell.exe -Command "hermes config set model.base_url https://api.deepseek.com/v1"
powershell.exe -Command "hermes config set display.language zh"
```

### 3. Copy secrets to Windows .env

DO NOT inline `$env:USERPROFILE` from WSL — bash eats the `$`. Write a `.ps1` file instead:

```powershell
# windows_setup.ps1 on Desktop
$key = "sk-d7f00cb6ff5547758242a9960f8915a7"
"DEEPSEEK_API_KEY=$key" | Out-File -FilePath "$env:USERPROFILE\.hermes\.env" -Encoding utf8
```

Then run: `powershell.exe -ExecutionPolicy Bypass -File "C:\Users\liu\Desktop\windows_setup.ps1"`

### 4. Copy weixin platform config

```bash
powershell.exe -Command "hermes config set platforms.weixin.enabled true"
powershell.exe -Command "hermes config set platforms.weixin.token 'c33941645ab7@im.bot:...'"
powershell.exe -Command "hermes config set platforms.weixin.extra.account_id 'c33941645ab7@im.bot'"
```

Add to `C:\Users\liu\.hermes\.env`:
```
WEIXIN_HOME_CHANNEL=o9cq809aMD6ApDXFosyV-Ng-cV5A@im.wechat
GATEWAY_ALLOW_ALL_USERS=true
```

Use `execute_code` (runs on Windows) with `C:\Users\liu\.hermes\.env` path.

### 5. Migrate state.db (sessions + memory)

**DO NOT directly copy state.db** — SQLite binary format is incompatible. The Windows Hermes will silently rebuild it to an empty DB.

Correct approach:
```bash
# WSL side
hermes backup -o /mnt/c/Users/liu/Desktop/hermes-backup.zip

# Windows side
hermes import --force C:\Users\liu\Desktop\hermes-backup.zip
```

### 6. Switch gateways

1. Stop WSL gateway: `hermes gateway stop` (or `pkill -9 -f "hermes.*gateway"` if stop fails)
2. Start Windows gateway: `hermes gateway run` (from Windows PowerShell)
3. Test: send a weixin message

### 7. Uninstall WSL Hermes

```bash
# Kill all Hermes processes
pkill -9 -f hermes
# Remove config/data
rm -rf ~/.hermes
# Docker-created files may be root-owned, need sudo
sudo rm -rf /home/liu123456/.hermes/one-api  # if applicable
```

## Critical Pitfalls

### state.db cross-platform incompatibility
The most common failure: copying WSL `state.db` (41MB) to Windows → Windows rebuilds it to 192KB on next launch. All session history and persistent memory are lost. **Always use `hermes backup` + `hermes import`.**

### execute_code runs in Windows sandbox on native Hermes
On Windows native, `execute_code` runs Python in `C:\Users\liu\AppData\Local\Temp\hermes_sandbox_xxx\`. WSL paths (`/mnt/c/...`) don't exist there. Use `C:\Users\liu\...` paths instead. The `terminal` tool can still reach `/mnt/c/` via WSL bash.

### write_file fails on Windows paths
`write_file` tries to execute through bash which can't handle `C:\Users\...`. Use `execute_code` with `open("C:\\Users\\...", "w")` or terminal with `/mnt/c/` paths.

### Terminal cache-file noise
Every terminal command may emit: `/bin/bash: line 5: C:/Users/liu/.hermes/cache/terminal/hermes-snap-xxx.sh: No such file or directory`. Harmless. Add `workdir="/tmp"` to suppress.

### Chinese characters in VBS files
VBS on Chinese Windows expects GBK encoding. Writing via `execute_code` with `encoding="ascii"` fails on Chinese paths. Use `open(path, "wb").write(content.encode("gbk"))`.

### Startup folder permission denied
Writing to `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\` from WSL or execute_code may fail even with correct permissions. Write VBS to Desktop, tell user to drag into `shell:startup`.

## Verification Checklist

- [ ] `hermes --version` works on Windows
- [ ] `hermes config show` shows correct model/provider
- [ ] `hermes gateway run` starts without errors
- [ ] Weixin messages arrive on Windows native instance
- [ ] `hermes sessions list` shows sessions from WSL (if backup imported)
- [ ] WSL `~/.hermes/` deleted
