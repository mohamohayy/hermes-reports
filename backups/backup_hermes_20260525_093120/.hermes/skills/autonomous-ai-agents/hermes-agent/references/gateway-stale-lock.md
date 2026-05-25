# Gateway Stale Lock — Diagnosis & Recovery

## Symptom

```
ERROR gateway.platforms.base: [Weixin] Weixin bot token already in use (PID 6496). Stop the other gateway first.
WARNING gateway.run: ✗ weixin failed to connect
ERROR gateway.run: Gateway exiting cleanly
```

Gateway exits immediately after startup. Port 8642 never becomes available.

## Root Cause

`~/.hermes/gateway.lock` and `~/.hermes/auth.lock` persist after gateway crashes. Lock file content:

```json
{"pid": 6496, "kind": "hermes-gateway", "argv": ["...hermes", "gateway", "run"], "start_time": null}
```

The PID in the lock file belongs to a dead process. On Windows, PIDs get recycled — the same PID may now belong to `NgcIso.exe` (Windows Hello) or another unrelated system process. Hermes reads the lock file and blocks startup.

## Tier 1: Clear locks and restart

1. **Delete both lock files:**
   ```powershell
   del %USERPROFILE%\.hermes\gateway.lock %USERPROFILE%\.hermes\auth.lock
   ```

2. **Kill any lingering gateway processes:**
   ```powershell
   Get-Process python* | Where-Object {
     (Get-CimInstance Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine -like "*gateway*"
   } | Stop-Process -Force
   ```

3. **Restart gateway:**
   ```
   hermes gateway run
   ```

4. **Verify:** Check if port 8642 is listening OR a new sync file appears at
   `~/.hermes/weixin/accounts/{account_id}.sync.json`.

## Tier 2: Token locked server-side (lock clear not enough)

If Tier 1 fails with the same "already in use" error, the WeChat iLink API server-side
still considers the old session active. Solution: re-scan QR for a fresh token.

1. **Create a fixed login script** (original `weixin-login.py` has broken path on pip installs):
   ```python
   # Save as ~/.hermes/weixin-login-fixed.py
   from gateway.platforms.weixin import qr_login
   from hermes_constants import get_hermes_home
   import asyncio, sys

   async def main():
       result = await qr_login(str(get_hermes_home()), timeout_seconds=300)
       if result:
           print(f"account_id: {result['account_id']}")
           print(f"token: {result['token']}")
       else:
           sys.exit(1)

   asyncio.run(main())
   ```

2. **Create a double-click .bat** on Desktop (`Hermes工具\微信扫码登录.bat`, GBK encoding):
   ```batch
   @echo off
   chcp 65001 >nul
   python "%USERPROFILE%\.hermes\weixin-login-fixed.py"
   pause
   ```

3. **User scans QR** → gets new `account_id` and `token`.

4. **Update config with new values:**
   ```powershell
   hermes config set platforms.weixin.token "ff8f3ec36e44@im.bot:0600007d..."
   hermes config set platforms.weixin.extra.account_id "ff8f3ec36e44@im.bot"
   ```

5. **Delete locks and restart:**
   ```powershell
   del %USERPROFILE%\.hermes\gateway.lock %USERPROFILE%\.hermes\auth.lock
   hermes gateway run
   ```

6. **Verify success:** New `.sync.json` appears at
   `~/.hermes/weixin/accounts/{new_account_id}.sync.json`. The `get_updates_buf`
   field being non-empty confirms session is active.

## Debugging Commands Cheat Sheet

```powershell
# Check gateway port
Get-NetTCPConnection -State Listen -LocalPort 8642

# Check what's using the PID from lock file
tasklist /fi "PID eq 6496"

# Check lock file contents
python -c "import json; print(json.load(open('$env:USERPROFILE/.hermes/gateway.lock')))"

# List all Python processes with their command lines
Get-Process python* | ForEach-Object {
  $cmd = (Get-CimInstance Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine
  Write-Host "PID $($_.Id): $cmd"
}

# Check latest weixin account
ls $env:USERPROFILE\.hermes\weixin\accounts\*.json | Sort-Object LastWriteTime -Descending | Select -First 5
```

## Cross-Reference

- SKILL.md pitfall: "Stale gateway.lock causes token already in use"
- SKILL.md pitfall: weixin-login.py path issue on pip-installed Windows
- Related: WeChat token expiry (session expired after days/weeks)
- Related: gateway stop permission denied
