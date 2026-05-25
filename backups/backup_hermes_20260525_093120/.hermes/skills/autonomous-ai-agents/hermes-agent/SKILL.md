---
name: hermes-agent
description: "Configure, extend, or contribute to Hermes Agent."
version: 2.2.0
author: Hermes Agent + Teknium
license: MIT
metadata:
  hermes:
    tags: [hermes, setup, configuration, multi-agent, spawning, cli, gateway, development]
    homepage: https://github.com/NousResearch/hermes-agent
    related_skills: [claude-code, codex, opencode]
---

# Hermes Agent

Hermes Agent is an open-source AI agent framework by Nous Research that runs in your terminal, messaging platforms, and IDEs. It belongs to the same category as Claude Code (Anthropic), Codex (OpenAI), and OpenClaw — autonomous coding and task-execution agents that use tool calling to interact with your system. Hermes works with any LLM provider (OpenRouter, Anthropic, OpenAI, DeepSeek, local models, and 15+ others) and runs on Linux, macOS, Windows (native, v0.14.0+), and WSL.

What makes Hermes different:

- **Self-improving through skills** — Hermes learns from experience by saving reusable procedures as skills. When it solves a complex problem, discovers a workflow, or gets corrected, it can persist that knowledge as a skill document that loads into future sessions. Skills accumulate over time, making the agent better at your specific tasks and environment.
- **Persistent memory across sessions** — remembers who you are, your preferences, environment details, and lessons learned. Pluggable memory backends (built-in, Honcho, Mem0, and more) let you choose how memory works.
- **Multi-platform gateway** — the same agent runs on Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Email, and 10+ other platforms with full tool access, not just chat.
- **Provider-agnostic** — swap models and providers mid-workflow without changing anything else. Credential pools rotate across multiple API keys automatically.
- **Profiles** — run multiple independent Hermes instances with isolated configs, sessions, skills, and memory.
- **Extensible** — plugins, MCP servers, custom tools, webhook triggers, cron scheduling, and the full Python ecosystem.

## Windows Native Installation (Beta, v0.14.0+)

Hermes Agent v0.14.0+ supports native Windows (cmd.exe / PowerShell) without WSL. Install via PyPI:

```powershell
# 在 PowerShell 或 cmd 中执行（国内用清华镜像加速）
pip install hermes-agent -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**One command, everything included.** The installer handles MinGit, Python stubs, Ctrl+C signal handling, and PTY management automatically. No cloning repos or running shell scripts.

After install, configure from WSL via `powershell.exe`:
```bash
powershell.exe -Command "hermes config set model.default deepseek-v4-pro"
powershell.exe -Command "hermes config set model.provider deepseek"
powershell.exe -Command "hermes config set model.base_url https://api.deepseek.com/v1"
powershell.exe -Command "hermes config set display.language zh"
```

Secrets go in `C:\Users\<user>\.hermes\.env` (use a `.ps1` script — inline PowerShell from WSL has encoding issues with `$env`).

**Pitfall:** Never use `$env:USERPROFILE` in inline `powershell.exe -Command` from WSL — bash eats the `$`. Always write to a `.ps1` file and run with `-File`.

**WSL vs Windows Native comparison:**

| | WSL | Windows 原生 |
|---|---|---|
| 安装 | clone 仓库 → venv | `pip install` 一行 |
| 桌面操控 | 需 `powershell.exe` 桥接 | 天然支持截图/鼠标/键盘 |
| 文件路径 | `/mnt/c/` 转换 | `C:\Users\...` 无转换 |
| 成熟度 | 稳定 | 早期 Beta |

**Cross-platform data migration:** See [references/cross-platform-migration.md](references/cross-platform-migration.md).

## Deployment

### Windows Native Install (v0.14.0+, early beta)

```bash
pip install hermes-agent -i https://pypi.tuna.tsinghua.edu.cn/simple  # China mirror
hermes config set model.provider deepseek     # or openrouter, anthropic, etc.
hermes config set model.default deepseek-v4-pro
hermes config set model.base_url https://api.deepseek.com/v1
hermes config set display.language zh
```

Add API key to `%USERPROFILE%\.hermes\.env` (NOT config.yaml). Restart after any config change.

**Pitfall:** `write_file` and `read_file` from WSL can't write to `C:\Users\...` paths. Use `execute_code` (runs on Windows) or terminal with `/mnt/c/...` paths instead.

**Pitfall (China):** Use Tsinghua mirror for pip (`-i https://pypi.tuna.tsinghua.edu.cn/simple`). GitHub raw downloads often timeout — use `ghproxy.net` mirror.

### Cross-Platform Migration (WSL → Windows Native)

**`state.db` is NOT cross-platform compatible.** A direct `cp ~/.hermes/state.db → C:\Users\...\` will be silently rebuilt (shrunk) by the Windows Hermes on next launch. Instead:

```bash
# On source (WSL):
hermes backup -o /mnt/c/Users/liu/Desktop/hermes-backup.zip

# On target (Windows):
hermes import --force C:\Users\liu\Desktop\hermes-backup.zip
```

Config settings (model, provider, display.language) and `.env` can be ported directly — they're platform-agnostic. Full walkthrough: [references/wsl-to-windows-migration.md](references/wsl-to-windows-migration.md).

## Core Operating Principles

### 1. Think Ahead, Don't Micro-Ask

The user explicitly expects the agent to plan ahead and minimize back-and-forth. When performing **any multi-step task** (migrations, setups, deployments, configuration, novel writing, file operations, debugging):

1. **Plan the full sequence upfront.** Figure out steps A→B→C before asking anything.
2. **Tell the plan once, then execute.** "Here's the plan: A → B → C. Starting now." Then do all steps without pausing for confirmation on each sub-step.
3. **Minimize "should I do X?" questions.** Prefer making reasonable default choices for low-stakes decisions. Only ask the user for decisions with genuine trade-offs.
4. **Anticipate follow-ups.** After completing task N, the user will likely want task N+1 — prepare for it. After installing, they'll want to configure. After writing chapter N, they'll want chapter N+1.

### 2. Maintain Conversation Context — Don't Regress

**Critical pitfall:** When the conversation has moved through multiple topics and the user sends a short/casual message (e.g., "你好", "在吗", "继续"), re-read the last ~5 messages before responding. Do NOT jump back to a topic from 5+ messages ago.

**Example of failure:** User discussed赚钱 plans → agent got distracted by a side task (开机自启) → user says "你好" after a brief pause → agent responds about 开机自启 progress rather than continuing the赚钱 discussion. The user's correction: "你要不要看看前几条信息是什么，记忆错乱了吗"

**Rule:** The most recent substantive topic is the current topic. A casual greeting or continuation prompt means "continue what we were just doing," not "go back to something from earlier in the conversation." If you're about to respond about topic X, verify it's the most recent topic — not something from 3+ turns ago unless the user explicitly brought it back.

### 3. Be Proactive, Not Reactive

When the user says "下次多考虑几步，不要总是让我做," they mean: factor in all the necessary steps yourself and present a complete solution, not a fragment that requires the user to fill in gaps. This applies universally — not just to migrations. Every interaction should feel like the agent is one step ahead, not one step behind.

## Multi-Profile Setup (Kanban / Multi-Agent)

Create specialized profiles for role-based task delegation:

```bash
hermes profile create ceo        # Planner / orchestrator
hermes profile create engineer   # Implementation
hermes profile create reviewer   # Code review / QA
```

Each profile inherits the default `.env` but needs its own model config and SOUL.md:

```bash
hermes -p ceo config set model.provider deepseek
hermes -p ceo config set model.default deepseek-v4-pro
hermes -p ceo config set model.base_url https://api.deepseek.com/v1
```

Edit `%USERPROFILE%\\.hermes\\profiles\\<name>\\SOUL.md` to define each profile's personality, workflow rules, and constraints. See [references/multi-profile-soul-templates.md](references/multi-profile-soul-templates.md) for ready-to-use CEO/engineer/reviewer SOUL templates. Kanban uses profile names as `assignee` — the dispatcher silently ignores unknown profiles.

### Dashboard vs Gateway API Server (important distinction)

| Feature | Dashboard | Gateway API Server |
|---------|-----------|-------------------|
| Command | `hermes dashboard --tui` | `hermes gateway run` |
| Default port | 9119 | 8642 |
| Web UI | ✅ Full web interface + Chat | ❌ OpenAI-compatible API only |
| `/v1/chat/completions` | ❌ | ✅ |
| Browser-accessible | ✅ `http://localhost:9119` | ❌ Returns 404 on root `/` |

The **Dashboard** is the web UI for humans. The **Gateway API Server** (`platforms.api_server`) is an OpenAI-compatible API endpoint for programmatic access. They serve different purposes and can run simultaneously on different ports.

To enable the API server platform (Gateway must be restarted after config change):

```bash
hermes config set platforms.api_server.enabled true
hermes config set platforms.api_server.extra.port 8648   # optional, default 8642
# Then restart gateway: Ctrl+C and re-run `hermes gateway run`
```

### Weixin (WeChat Personal) Platform

Connects Hermes to WeChat personal accounts via Tencent's iLink Bot API (`https://ilinkai.weixin.qq.com`).

**⚠️ Expectation:** This requires a one-time QR scan on the user's phone — WeChat's iLink API has no password-based auth. The token persists for days/weeks; re-scan only when the session expires. Tell the user this upfront so they're not surprised.

**Requirements:** `aiohttp` and `cryptography` Python packages.

**Pitfall:** Some Hermes venvs ship without `pip` (only `ensurepip` bootstrap). If `pip install` fails with "No module named pip", run first:
```bash
~/.hermes/hermes-agent/venv/bin/python3 -m ensurepip --upgrade
```
Then install: `~/.hermes/hermes-agent/venv/bin/python3 -m pip install aiohttp cryptography`

**Setup flow:**
1. Enable the platform: `hermes config set platforms.weixin.enabled true`
2. Run QR login to get a token:
   ```bash
   python3 ~/.hermes/weixin-login.py
   ```
   (or use `hermes gateway setup` interactive wizard → weixin section)
3. Scan the ASCII QR code with WeChat, confirm in the app
4. The token is saved automatically to `~/.hermes/weixin/accounts/` (a JSON file with token, base_url, user_id)
5. **CRITICAL**: The saved token file is NOT automatically read by the gateway. You MUST manually configure both `token` AND `extra.account_id`. The account filename IS the account_id (script saves as `~/.hermes/weixin/accounts/{account_id}.json`):
   ```bash
   # Read saved file to get token and account_id
   ACCT_FILE=$(ls ~/.hermes/weixin/accounts/*.json | head -1)
   ACCOUNT_ID=$(basename "$ACCT_FILE" .json)
   TOKEN=$(python3 -c "import json; print(json.load(open('$ACCT_FILE'))['token'])")

   # Both are required — missing account_id causes: WEIXIN_ACCOUNT_ID is required
   hermes config set platforms.weixin.enabled true
   hermes config set platforms.weixin.token "$TOKEN"
   hermes config set platforms.weixin.extra.account_id "$ACCOUNT_ID"
   ```
   Alternatively, set `WEIXIN_TOKEN=<token>` and `WEIXIN_ACCOUNT_ID=<id>` in `~/.hermes/.env`.
6. Restart gateway: `hermes gateway stop && hermes gateway run`

**Pitfall — `hermes gateway restart` timeout on Windows native:** `hermes gateway restart` (and sometimes `hermes gateway stop`) may hang indefinitely on Windows native. Use `hermes gateway run --replace` instead — it atomically kills the old process and starts a fresh one. Also check `hermes gateway status` first to confirm whether a gateway is already running before deciding to start/restart.

**Pitfall:** After connecting, incoming messages may be rejected with `Unauthorized user`. The gateway denies all users by default when no allowlists are configured. Either allow all users or set a platform-specific allowlist in `~/.hermes/.env`:
```bash
# Option A: allow all (simpler, less secure)
echo 'GATEWAY_ALLOW_ALL_USERS=true' >> ~/.hermes/.env

# Option B: allow specific weixin user only
echo 'WEIXIN_ALLOWED_USERS=your_user_id@im.wechat' >> ~/.hermes/.env
```
Restart gateway after changing `.env`.

**Verification:** After restart, confirm WeChat connected successfully:
```bash
# Check gateway is running
hermes gateway status
# Confirm weixin in logs
grep "weixin connected" ~/.hermes/logs/gateway.log | tail -1
# Should show: ✓ weixin connected
```
Then send a test message from WeChat to verify round-trip delivery.

**Pitfall — Gateway management on Windows native:** The `terminal` tool (WSL bash) often cannot capture output from `powershell.exe -Command "hermes gateway ..."` — stdout is empty even when the command succeeds.

For **one-shot queries** (`hermes gateway status`, `hermes config set/show`, `hermes doctor`): use `execute_code` + `subprocess`. It runs in the Windows sandbox and captures output reliably.

For **starting the gateway** (`hermes gateway run`): do NOT use `execute_code` — `subprocess.run(['hermes', 'gateway', 'run', '--replace'], capture_output=True)` will **time out** because `gateway run` is a foreground-blocking process that never returns stdout. Instead use `terminal(background=true, workdir='/tmp')` to run:
```
powershell.exe -Command "Start-Process -WindowStyle Hidden -FilePath 'hermes' -ArgumentList 'gateway run --replace'"
```
Then wait 5-8 seconds and check with `execute_code` + `subprocess.run(['hermes', 'gateway', 'status'])`.

For **stopping the gateway**: `hermes gateway stop` may fail with "Permission denied" on non-elevated Windows. Use Task Manager (find `python.exe` on the gateway port via `netstat -ano | findstr :<port>`) or an admin PowerShell with `Stop-Process -Id <PID> -Force`.

**Pitfall:** WeChat tokens expire after days/weeks. When the gateway log shows `[Weixin] Session expired; pausing for 10 minutes`, the stored session is no longer valid. Re-run the QR login script to obtain a fresh session:

```bash
~/.hermes/hermes-agent/venv/bin/python3 ~/.hermes/weixin-login.py
```

**⚠️ Re-scanning creates a NEW account file with a NEW account_id.** Each QR scan produces a fresh account (e.g. `c33941645ab7@im.bot.json`). The old account file is NOT updated. You MUST find the newest account file and update BOTH `token` and `extra.account_id` in config.yaml:

```bash
# Find newest account file
ls -lt ~/.hermes/weixin/accounts/*.json | head -3

# Read token from newest file, update config
# (replace c33941645ab7@im.bot with actual newest filename stem)
ACCT_FILE=$(ls -t ~/.hermes/weixin/accounts/*.json | head -1)
ACCOUNT_ID=$(basename "$ACCT_FILE" .json)
TOKEN=$(python3 -c "import json; print(json.load(open('$ACCT_FILE'))['token'])")
hermes config set platforms.weixin.token "$TOKEN"
hermes config set platforms.weixin.extra.account_id "$ACCOUNT_ID"
```

Then restart gateway: `hermes gateway stop && hermes gateway run` (the running gateway will NOT pick up the new account file).

**Note:** The account file may contain a `get_updates_buf` field (session blob) rather than a literal `token` field after initial login. In that case, the file still serves as the account record — the gateway reads it directly.

**Manual config** (if QR login unavailable):
```yaml
platforms:
  weixin:
    enabled: true
    extra:
      account_id: "your_account_id"
    token: "your_bot_token"
```

**Pitfall:** The gateway MUST be restarted after enabling any platform. A running gateway does not pick up new platform config.

**Pitfall — 会话模型 vs config.yaml 模型不一致诊断：** 微信用户反馈回复用的模型和 config.yaml 配置不一致（例如 config.yaml 配了 DeepSeek，但回复却说用的 OpenRouter → xiaomi/mimo）。原因：**Hermes 网关的会话级模型路由和 config.yaml 的 model.* 设置是两个独立层**。config.yaml 的 model.* 影响 CLI 和新会话的默认模型；但已存在的微信会话可能有会话级别模型覆盖（agent 在之前的 turn 中通过 tool call 修改了模型路由）。诊断路径：
1. **查浅层**：`hermes config show` 确认 model.provider/model.default
2. **查深层**：读取最近微信会话的 jsonl 文件（`~/.hermes/sessions/20260522_*.jsonl`），看 last 5 条 assistant 回复中是否包含 "当前会话用的是" 等自我引用信息
3. **查日志**：`gateway.log` 搜索 "response ready" + 对应时间戳，配合 jsonl 确认 agent 实际路由
4. **修复**：重置 config.yaml 后需重启网关（`hermes gateway run --replace`）清除会话缓存。如果还不行，说明 jsonl 上下文持久化了模型覆盖，新消息会继承上次会话的 system prompt 路由配置
参考 [references/weixin-model-mismatch-debug.md](references/weixin-model-mismatch-debug.md) 查看完整排查记录。另外见 [references/windows-env-auth-architecture.md](references/windows-env-auth-architecture.md) 了解密钥架构，以及 [references/credential-pool-debug.md](references/credential-pool-debug.md) 了解凭据池回退诊断。

**Pitfall (Windows native — gateway stop permission denied):** On Windows-native Hermes, `hermes gateway stop` may fail with "Permission denied" when the gateway was started from a different process context (e.g. Task Scheduler, another terminal, or a different user). `taskkill /f /pid <PID>` from a non-elevated process may also be silently denied with no output. Workarounds by preference:
- Kill from Task Manager (find `python.exe` listening on the gateway port — use `netstat -ano | findstr :<port>` to get the PID first)
- Or use an elevated (admin) PowerShell: `Stop-Process -Id <PID> -Force`
- If you truly cannot kill the old process: set the config anyway and test with `send_message(action='list')`. Re-enabling a previously-configured platform (where accounts already exist under `~/.hermes/weixin/accounts/` and `.env` already has `GATEWAY_ALLOW_ALL_USERS` / `WEIXIN_HOME_CHANNEL`) sometimes takes effect without a restart — the gateway may already have the old session loaded. Not guaranteed, but worth trying before escalating.

**Pitfall (Windows — `hermes gateway install` permission denied):** On non-admin Windows accounts, `hermes gateway install` and manual `schtasks /Create` both fail with "ERROR: Access is denied" — the Windows Task Scheduler requires elevation to create tasks, even without `/rl HIGHEST`. The fallback is a VBS script in the user's Startup folder (no admin needed, same login trigger):

1. Write this VBS to `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\HermesGateway.vbs`:
   ```vbscript
   Set WshShell = CreateObject("WScript.Shell")
   WshShell.Run """C:\Users\<user>\AppData\Local\Programs\Python\Python311\Scripts\hermes.EXE"" gateway run", 0, False
   ```
   The second parameter `0` hides the console window (silent background launch). Change to `1` to show the window for debugging.
2. Verify the VBS exists under the Startup folder with `dir "%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\HermesGateway.vbs"`.
3. The user can also open the Startup folder directly via `Win+R` → `shell:startup` to inspect or delete the file.

Existing items in the Startup folder are visible under Task Manager → Startup tab as "Program" entries (not labeled "HermesGateway"). The VBS launches `hermes gateway run` in a fresh process on every login; duplicate processes are harmless — the gateway will fail to bind the port if already running, and the second instance exits cleanly.

**Pitfall — Startup folder write denied from WSL / execute_code:** Writing to `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\` from WSL (`/mnt/c/Users/...`) or from `execute_code` sandbox may fail with Permission denied even when the user has write access. The user must move the VBS file into the Startup folder manually — create it on Desktop first, then tell the user to drag it into `shell:startup` (Win+R → `shell:startup`).

**Pitfall — Chinese characters in VBS path via execute_code:** When writing VBS content via `execute_code` with `encoding="ascii"`, any Chinese characters in the batch file path (e.g., `\u542f\u52a8Hermes网关.bat`) cause `UnicodeEncodeError`. Solution: use an English-only path for the batch file (e.g., `C:\Users\liu\Desktop\start_hermes.bat`) or write the VBS via WSL terminal with `printf` + heredoc.

**Pitfall:** The QR login script (`weixin-login.py`) can NOT be run by the agent on the user's behalf. It displays a QR code that expires in ~30 seconds and the user must scan it with their phone. Running it via the agent's terminal tool will just time out. Tell the user to run it themselves.

**For WSL (git-cloned install):**
```bash
~/.hermes/hermes-agent/venv/bin/python3 ~/.hermes/weixin-login.py
```

**For Windows native (pip-installed):** The bundled `~/.hermes/weixin-login.py` has a hardcoded `sys.path.insert(0, "~/.hermes/hermes-agent")` which does NOT exist in pip installs — the gateway module lives in site-packages instead. The script will fail with `ModuleNotFoundError: No module named 'gateway'`. Fix: create a corrected copy that skips the path hack (see [references/weixin-login-fixed.py](references/weixin-login-fixed.py)). Also create a double-click `.bat` file on Desktop with `chcp 65001` for UTF-8 QR display — the user only needs to double-click and scan with their phone:
```batch
@echo off
chcp 65001 >nul
python "%USERPROFILE%\.hermes\weixin-login-fixed.py"
pause
```

The script auto-refreshes expired QR codes up to 3 times within the 300s timeout.

**Pitfall — Stale gateway.lock causes "token already in use (PID XXXX)" after crash:** When the gateway process crashes or is killed uncleanly, `~/.hermes/gateway.lock` and `~/.hermes/auth.lock` persist with the old PID. On subsequent gateway starts, Hermes reads the lock file, finds the PID, and refuses to connect WeChat: `ERROR gateway.platforms.base: [Weixin] Weixin bot token already in use (PID XXXX)`. This happens even when the old process is dead — and worse, Windows may have recycled that PID to an unrelated process (e.g., `NgcIso`, `svchost`), making the error look nonsensical. The stale PID is NOT stored in `state.db` or sync files — it comes purely from the lock file on disk.

**Diagnosis:**
```bash
# Check what's in the lock file
python3 -c "import json; print(json.load(open('$HOME/.hermes/gateway.lock')))"
# Verify if that PID is actually a Hermes/gateway process
tasklist /fi "PID eq XXXX"
```

**Fix:** Delete both lock files and restart:
```bash
rm ~/.hermes/gateway.lock ~/.hermes/auth.lock
# Or on Windows:
del %USERPROFILE%\.hermes\gateway.lock %USERPROFILE%\.hermes\auth.lock
```
Then restart: `hermes gateway run`. If the error persists even after clearing locks, the WeChat token itself may be locked server-side — re-scan QR for a fresh token.

See [references/gateway-stale-lock.md](references/gateway-stale-lock.md) for the full diagnosis transcript and step-by-step recovery walkthrough.

### Opening from WSL in Windows Chrome

```bash
# Dashboard (web UI)
powershell.exe -Command "Start-Process 'chrome' -ArgumentList 'http://localhost:9119'"

# Or to launch Dashboard if not running:
hermes dashboard --tui &
sleep 3
powershell.exe -Command "Start-Process 'chrome' -ArgumentList 'http://localhost:9119'"
```

`cmd.exe /c start chrome` is unreliable from WSL due to UNC path issues. Always use `powershell.exe` for WSL→Windows interop.

**Encoding pitfall:** PowerShell on Chinese Windows uses GBK (CP936) as the default console encoding. Inline `powershell.exe -Command "..."` that contains non-ASCII characters will get garbled, and variables may be corrupted by the shell. Two workarounds:
- Write the script to a `.ps1` file (via `write_file`) with ASCII-only content, then run with `powershell.exe -ExecutionPolicy Bypass -File <path>`
- Or add `[Console]::OutputEncoding = [Text.Encoding]::UTF8` at the top — but this only fixes output, not input parsing
The `.ps1` file approach is more robust and should be the default for complex commands.

### Windows + WSL — State Migration

When migrating from WSL Hermes to Windows native Hermes, the state.db (sessions + memory) cannot be directly copied — SQLite binary format is incompatible across platforms. Use the SQL dump → restore approach documented in `references/windows-native-migration.md`.

### Windows + WSL Auto-Start (Recommended for Windows Users)

To launch `hermes --tui` automatically at Windows login:

1. Create `~/.hermes/startup.sh` (executable) that activates your Python environment and runs `hermes --tui`;
2. Generate a Windows Task Scheduler `.xml` definition pointing to `wsl -e bash -c '/path/to/startup.sh'`;
3. Import the `.xml` into Task Scheduler under `\Hermes\`.

This ensures Hermes starts in a clean WSL session with full terminal access. See `templates/wsl-startup.sh` and `templates/hermes-auto-start.xml` for ready-to-use, tested templates.

### Windows Native Installation (v0.14.0+)

Hermes now runs natively on Windows (`cmd.exe` / PowerShell) without WSL — no virtualisation, no path translation, direct access to Windows desktop tools (screenshots, pyautogui, window management).

**Install (one command):**

```powershell
pip install hermes-agent -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**Configure (same as WSL):**

```powershell
hermes config set model.default deepseek-v4-pro
hermes config set model.provider deepseek
hermes config set model.base_url https://api.deepseek.com/v1
hermes config set display.language zh
```

Config lives at `%USERPROFILE%\.hermes\config.yaml`, secrets in `%USERPROFILE%\.hermes\.env`.

**Launch:** Open PowerShell, type `hermes`.

**Pitfall — WSL→Windows config:** Inline `powershell.exe -Command "hermes config set ..."` from WSL sometimes times out. Run each `config set` command separately with 30s timeout, or write a `.ps1` file and run via `-File`.

**Pitfall — `$env:USERPROFILE` from WSL:** When running PowerShell inline from WSL (`powershell.exe -Command "..."`), bash eats `$`-prefixed variables. Always write to a `.ps1` file on desktop and run with `powershell.exe -ExecutionPolicy Bypass -File <path>`.

**Pitfall — `execute_code` runs in Windows sandbox on native Hermes:** On Windows native installs, `execute_code` Python scripts execute in a Windows sandbox (e.g., `C:\Users\liu\AppData\Local\Temp\hermes_sandbox_xxx\`), NOT in WSL. WSL paths like `/mnt/c/Users/liu/` are inaccessible from `execute_code` — use Windows native paths (`C:\Users\liu\...`). The `terminal` tool can still reach `/mnt/c/` paths via WSL bash. This is the reverse of WSL Hermes where `execute_code` and `terminal` both share the WSL environment.

**Pitfall — `terminal` cache-file errors on Windows native:** Terminal commands may emit `/bin/bash: line 5: C:/Users/liu/.hermes/cache/terminal/hermes-snap-xxx.sh: No such file or directory` on every invocation. This is harmless noise from the terminal backend trying to apply cached directory/snapshot state that doesn't survive across sessions. The shell still executes the actual command correctly. Workaround: add `workdir="/tmp"` or `workdir="/home/liu123456"` to bypass the stale cache lookup.

**Pitfall — WSL uninstall leaves root-owned Docker files:** `rm -rf ~/.hermes` may fail on files created by Docker containers (e.g., one-api logs) that are owned by `root:root`. Use `sudo rm -rf` or the user must manually `sudo rm` those files from WSL terminal.

## Token Optimization

Hermes has **built-in context compression**ws (`cmd.exe` / PowerShell) without WSL — no virtualisation, no path translation, direct access to Windows desktop tools (screenshots, pyautogui, window management).

**Install (one command):**

```powershell
pip install hermes-agent -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**Configure (same as WSL):**

```powershell
hermes config set model.default deepseek-v4-pro
hermes config set model.provider deepseek
hermes config set model.base_url https://api.deepseek.com/v1
hermes config set display.language zh
```

Config lives at `%USERPROFILE%\.hermes\config.yaml`, secrets in `%USERPROFILE%\.hermes\.env`.

**Launch:** Open PowerShell, type `hermes`.

**Pitfall — WSL→Windows config:** Inline `powershell.exe -Command "hermes config set ..."` from WSL sometimes times out. Run each `config set` command separately with 30s timeout, or write a `.ps1` file and run via `-File`.

**Pitfall — `$env:USERPROFILE` from WSL:** When running PowerShell inline from WSL (`powershell.exe -Command "..."`), bash eats `$`-prefixed variables. Always write to a `.ps1` file on desktop and run with `powershell.exe -ExecutionPolicy Bypass -File <path>`.

**Pitfall — `execute_code` runs in Windows sandbox on native Hermes:** On Windows native installs, `execute_code` Python scripts execute in a Windows sandbox (e.g., `C:\Users\liu\AppData\Local\Temp\hermes_sandbox_xxx\`), NOT in WSL. WSL paths like `/mnt/c/Users/liu/` are inaccessible from `execute_code` — use Windows native paths (`C:\Users\liu\...`). The `terminal` tool can still reach `/mnt/c/` paths via WSL bash. This is the reverse of WSL Hermes where `execute_code` and `terminal` both share the WSL environment.

**Pitfall — `terminal` cache-file errors on Windows native:** Terminal commands may emit `/bin/bash: line 5: C:/Users/liu/.hermes/cache/terminal/hermes-snap-xxx.sh: No such file or directory` on every invocation. This is harmless noise from the terminal backend trying to apply cached directory/snapshot state that doesn't survive across sessions. The shell still executes the actual command correctly. Workaround: add `workdir="/tmp"` or `workdir="/home/liu123456"` to bypass the stale cache lookup.

**Pitfall — WSL uninstall leaves root-owned Docker files:** `rm -rf ~/.hermes` may fail on files created by Docker containers (e.g., one-api logs) that are owned by `root:root`. Use `sudo rm -rf` or the user must manually `sudo rm` those files from WSL terminal.

## Token Optimization

Hermes has **built-in context compression**

For additional savings, use the lightweight `scripts/token_saver.py` which provides:
- **Semantic caching** — identical/similar queries return cached responses (100% savings on repeats)
- **Prompt compression** — keyword-preserving head+tail extraction (40-60% reduction)
- Zero dependencies (stdlib only), 7-day cache expiry

```bash
# Check cache stats
python3 ~/.hermes/skills/autonomous-ai-agents/hermes-agent/scripts/token_saver.py
```

**Heavier alternatives** (require torch/transformers, often blocked by China GFW):
- **GPTCache** (zilliztech/gptcache, ⭐8k) — full semantic cache with vector DB backends
- **LLMLingua** (microsoft/LLMLingua, ⭐6.2k) — ML-based prompt compression (2-10x reduction)
- **LiteLLM** (BerriAI/litellm, ⭐47.7k) — multi-provider proxy with built-in caching and cost tracking

These are worth installing if the lightweight version isn't sufficient, but expect 2GB+ of dependencies.

## Windows Native Migration (WSL → Windows)

When migrating from a WSL Hermes instance to a native Windows install:

1. `pip install hermes-agent -i https://pypi.tuna.tsinghua.edu.cn/simple` (China mirror)
2. Copy model config: same `model.default`, `model.provider`, `model.base_url`
3. Copy API keys to `C:\Users\<user>\.hermes\.env`
4. Copy weixin/gateway config: `platforms.weixin.*`, `WEIXIN_HOME_CHANNEL`, `GATEWAY_ALLOW_ALL_USERS`
5. **Do NOT directly copy state.db** — SQLite format is cross-platform incompatible. Use `hermes backup` (WSL) → `hermes import` (Windows) instead. Direct copy causes silent rebuild to empty DB.
6. Persistent memory survives through `.env`/`config.yaml` — the `memory` tool stores entries in state.db, so after migration persistent memories are lost until re-imported.

**Pitfall:** Running both gateways simultaneously with the same weixin token causes conflicts. Stop WSL gateway (`hermes gateway stop`) before starting Windows gateway (`hermes gateway run`).

See full step-by-step: [references/wsl-to-windows-migration.md](references/wsl-to-windows-migration.md)

## Troubleshooting

### Config Loss / Missing config.yaml

**Symptoms**: Hermes uses wrong provider (e.g., openrouter instead of deepseek), API calls fail with 401/403, or `hermes config show` shows empty Model field.

**Root cause**: `config.yaml` is missing from `~/.hermes/`. Hermes falls back to defaults — no model, no provider, no API keys.

**Detection**:
```bash
hermes doctor   # Look for "config.yaml not found (using defaults)"
```

**Recovery** — rebuild from scratch:
```bash
hermes config set model.default <model>
hermes config set model.provider <provider>
hermes config set model.base_url <base_url>
hermes config set display.language zh
# Then re-apply all platform configs:
hermes config set platforms.weixin.enabled true
hermes config set platforms.weixin.token "<token>"
hermes config set platforms.weixin.extra.account_id "<account_id>"
# Re-create .env with essential vars:
# GATEWAY_ALLOW_ALL_USERS=true
# WEIXIN_HOME_CHANNEL=<user_id>@im.wechat
```

Restart gateway after recovery.

**Pitfall:** `execute_code` sandbox may not see the real `~/.hermes/config.yaml` due to path mapping. Use `hermes doctor` or `hermes config show` (run via subprocess) instead of direct file reads to diagnose.

### Pitfall — Proactive Key Discovery (Don't Ask, Look First)

When diagnosing a 401/auth failure, the user expects you to **find the key yourself** rather than asking "where is your API key?". Common places to search, in order:

1. `~/.hermes/.env` — standard secrets file (DEEPSEEK_API_KEY, OPENAI_API_KEY, etc.)
2. `~/.hermes/auth.json` — credential pool (structured key storage with providers and labels)
3. `~/.hermes/state-snapshots/` — backup copies from upgrades
4. Windows-specific: `set DEEPSEEK_API_KEY` / environment variables
5. Shell files: `~/.bashrc`, `~/.zshrc`, `~/.profile`

Use `execute_code` (runs on Windows sandbox) to search these paths automatically. Only ask the user if none of these locations yield a valid key.

### Pitfall — Auxiliary Module API Keys (401 on title_generation, vision, compression, etc.)

**Symptom:** The main model works fine (200 OK), but auxiliary modules fail with `HTTP 401: Authentication Fails, Your api key: ****a2ef is invalid`. Error is `Auxiliary title generation failed` or similar.

**Root cause:** Hermes config.yaml has an `auxiliary:` section with per-module key fields (`api_key`) that default to `''`. Unlike the main model which reads credentials from `auth.json` / credential pool / `.env`, **auxiliary modules have their own independent `api_key` field** and DO NOT inherit from the model-level key unless `auxiliary.<module>.provider` is explicitly set AND the module's credential resolution logic finds a suitable key.

When `auxiliary.title_generation.api_key: ''` and `provider: auto` (the default), the auxiliary module sends a null key to the provider, getting 401 — even though the main model uses the same provider just fine with the key from `.env`.

**Fix:** Read the API key from `~/.hermes/.env` or `auth.json` and write it directly into the auxiliary module's `api_key` field in `config.yaml`:

```yaml
auxiliary:
  title_generation:
    api_key: sk-xxxxx           # ← must match the environment key
    base_url: ''
    provider: auto              # inherits main provider by default
    timeout: 30
```

Example fix (Windows native, via execute_code):
```python
import re, subprocess, yaml
from pathlib import Path

# Read key from .env
result = subprocess.run(['cmd.exe', '/c', 'type C:\\Users\\<user>\\.hermes\\.env'], capture_output=True, text=True, shell=True)
match = re.search(r'DEEPSEEK_API_KEY=(.+)', result.stdout)
key = match.group(1).strip()

# Read config, patch the auxiliary section
config_path = Path.home() / '.hermes' / 'config.yaml'
content = config_path.read_text('utf-8')
old = "title_generation:\n    api_key: ''"
new = f"title_generation:\n    api_key: {key}"
content = content.replace(old, new)
config_path.write_text(content, 'utf-8')
```

This same issue can affect any auxiliary module: `compression`, `vision`, `session_search`, `approval`, `curator`, `mcp`, `skills_hub`, `triage_specifier`, `web_extract`. When one fails with 401, check its `api_key` first.

See [references/windows-env-auth-architecture.md](references/windows-env-auth-architecture.md) for the full key resolution architecture and [references/auxiliary-key-debug.md](references/auxiliary-key-debug.md) for a worked debug transcript.

**Quick check script:** `scripts/check-auxiliary-keys.py` scans all auxiliary modules and reports which have empty/missing api_key. Run it directly to audit the auxiliary section without grepping config.yaml manually.

**Prevention on new installs:** After configuring the main model key in `.env`, run this to fill all auxiliary `api_key` fields at once:

```bash
# Read key from .env and inject into all empty auxiliary api_key fields
KEY=$(grep DEEPSEEK_API_KEY ~/.hermes/.env | cut -d= -f2)
python3 -c "
import yaml
p = Path.home() / '.hermes' / 'config.yaml'
c = yaml.safe_load(p.read_text())
for mod in c.get('auxiliary', {}):
    if c['auxiliary'][mod].get('api_key') == '':
        c['auxiliary'][mod]['api_key'] = '$KEY'
p.write_text(yaml.dump(c, default_flow_style=False, allow_unicode=True), 'utf-8')
"
```

### Pitfall — Session Summaries Can Be Wrong (Don't Trust LLM-Generated History)

**Symptom:** `session_search` returns summaries that claim "there are 9 more modules with empty api_key" or "multiple issues still unresolved" — but direct inspection shows otherwise.

**Root cause:** Session search summaries are generated by a downstream summarizer LLM. They can hallucinate details, conflate separate conversations, or mis-state the current state of the system. The summary is NOT a verifiable audit trail — it's a compressed narrative that may prioritize "most dramatic" over "most accurate".

**Rule of thumb:** When a session summary contradicts something you can directly inspect (a file on disk, a running process, a config value), **trust the direct inspection**. The summary is a hint about what might have been discussed, not a fact about the current system state.

**Example:** Session 20260522_145757_5b8a was summarized as "9 other auxiliary modules have empty api_key". Direct inspection of config.yaml showed all 9 were already populated — only `title_generation` was empty. The summarizer conflated the initial discovery (before the fix) with the final state (after the fix).

### Pitfall — session_search Pretend-pass Output

When `session_search` returns no results or suspiciously generic summaries, this typically means your query was too specific or the session content wasn't well-indexed. Try broader queries with OR-based keywords (e.g., "auxiliary OR api_key OR 401") rather than multi-word AND queries.

### Desktop File Organization (Windows Native)

When cleaning up the Windows Desktop:

1. **Plan first** — scan with `os.listdir(desktop)` and categorize into logical groups before moving anything
2. **Common categories**: `快捷方式` (`.lnk`/`.url`), `文档` (`.docx`/`.pdf`/`.pptx`), `微信工具` (WeChat scripts), `UFO相关` (UFO installers), and the project's own `Hermes工具` folder
3. **Always create folders first** with `os.makedirs()`, then `shutil.move()` each file
4. **Leave `.lnk` files alone** — they're user's app shortcuts, not agent scripts. Move them into a `快捷方式/` folder instead of deleting
5. **Move, don't delete** — when in doubt, move to a categorized folder. The user can delete later
6. **Verify final state** — list what remains on Desktop after the move
7. **The `Hermes工具/` folder is the canonical location** for all agent-generated scripts. Move stray `.ps1`/`.py`/`.bat` files into subdirectories within it (e.g., `微信机器人/` for WeChat-related scripts)

### Background Task Pattern (User Preference: "好了通知我")

When the user says "好了通知我", "好了叫我", or similar during a long operation:

1. **Use `terminal(background=true, notify_on_complete=true)`** — NOT synchronous `execute_code` with `subprocess.Popen` polling
2. The background+notify pattern lets the user continue the conversation while the task runs
3. Only use synchronous waiting when the user explicitly asks "等它完" or the next step depends on the task's output
4. Pitfall: `execute_code` with `subprocess.run(timeout=300)` for long pip installs blocks the conversation. Use `terminal(background=true)` + return control to the user immediately

## Maintenance

### Proactive Auto-Update

Check for and apply Hermes Agent updates proactively — do not wait for the user to ask. The user expects the agent to keep itself current.

```bash
# Check if updates are available
hermes update --check

# If behind, apply with --yes to skip interactive prompts
hermes update --yes
```

**Post-update:** `hermes update` stops the dashboard if the backend no longer matches the updated frontend. Restart it:

```bash
hermes dashboard --port 9119 &
```

The `--yes` flag skips all interactive prompts (config migration, stash restore). If new config options appear, run `hermes config migrate` separately.

### State DB Health

If `agent.log` shows `database disk image is malformed`, the state.db is corrupted. Sessions/memory may be lost even though the gateway appears to work. See [references/state-db-repair.md](references/state-db-repair.md) for recovery steps.

### Cross-Platform State Migration

When moving between WSL and native Windows (or any two platforms), the state.db uses SQLite WAL mode — **always copy `state.db` + `state.db-wal` + `state.db-shm` together**, then run `wal_checkpoint(TRUNCATE)` + `journal_mode=DELETE` + `VACUUM` on the target.

Do NOT use:
- Direct `cp state.db` alone (discards WAL → malformed)
- `hermes backup` → `hermes import` (backup skips .db-wal/.db-shm)
- `sqlite3 .dump` / Python `iterdump()` (FTS virtual tables break)

The bridge script is also mirrored at `scripts/hermes_memory_bridge.py` within this skill directory for version tracking. See `references/local-vector-memory-windows.md` for full setup, pitfalls, and usage patterns.

## Configuration

Configuration lives in `~/.hermes/config.yaml`. Use `hermes config show/set` to read/write values safely. (There is no `get` subcommand — use `show` to display, `set` to change.)

```bash
# Core settings
hermes config set provider openrouter
hermes config set model llama-3.1-405b

# Display / UI
hermes config set display.tool_progress_command true
hermes config set display.language zh           # UI language: zh / en / ja ...

# Gateway platforms (api_server for web access)
hermes config set platforms.api_server.enabled true
hermes config set platforms.api_server.extra.port 8648
```

Secrets (API keys, tokens) go in `~/.hermes/.env` — never in `config.yaml`.

## Provider Quirks

Non-standard providers that need special setup:

- **Xiaomi MiMo** — no native OpenAI-compatible API; requires Cookie-based auth + `mimo-2api` proxy. See [references/xiaomi-mimo.md](references/xiaomi-mimo.md).

## Skills

Skills are self-contained markdown documents (`SKILL.md`) stored in `~/.hermes/skills/`. They define reusable workflows — e.g., debugging a failing tool, setting up a local LLM, or deploying to a cloud platform.

Use `hermes skills list` to see installed skills, and `hermes skill view <name>` to load one.

## Development

See [AGENTS.md](https://github.com/NousResearch/hermes-agent/blob/main/AGENTS.md) for full architecture docs, file dependencies, CLI/TUI/gateway internals, and testing guidance.

Platform docs: https://hermes-agent.nousresearch.com/docs/user-guide/messaging/

### Sessions

```
hermes sessions list        List recent sessions
hermes sessions browse      Interactive picker
hermes sessions export OUT  Export to JSONL
hermes sessions rename ID T Rename a session
hermes sessions delete ID   Delete a session
hermes sessions prune       Clean up old sessions (--older-than N days)
hermes sessions stats       Session store statistics
```

> ⚠️ **Cross-platform migration:** `state.db` is NOT binary-compatible between WSL and Windows native. Use `hermes backup` + `hermes import`, not direct file copy. See [references/cross-platform-migration.md](references/cross-platform-migration.md).

## Contributing

Contributions welcome! All PRs must pass CI and include tests.

### Git Commit Conventions

Use conventional commits:

Types: `fix:`, `feat:`, `refactor:`, `docs:`, `chore:`

### Key Rules

- **Never break prompt caching** — don't change context, tools, or system prompt mid-conversation
- **Message role alternation** — never two assistant or two user messages in a row
- Use `get_hermes_home()` from `hermes_constants` for all paths (profile-safe)
- Config values go in `config.yaml`, secrets go in `.env`
- New tools need a `check_fn` so they only appear when requirements are met