# Cross-Platform Migration (WSL ↔ Windows Native)

## Critical: state.db is NOT cross-compatible

Direct binary copy of `state.db` between WSL (Linux) and Windows native Hermes **silently fails** — the receiving Hermes will rebuild a fresh database, losing all session history and persistent memory.

**Symptom:** After copying state.db, `hermes sessions stats` shows 0.1MB / 2 sessions instead of the expected ~42MB.

## Correct Migration: Backup → Import

### Step 1: Backup from source

```bash
# From WSL
hermes backup -o /mnt/c/Users/liu/Desktop/hermes-backup.zip
```

Or quick snapshot (config + state.db + .env only):

```bash
hermes backup --quick -o /mnt/c/Users/liu/Desktop/hermes-backup.zip
```

### Step 2: Import on target

```powershell
# From Windows PowerShell
hermes import --force C:\Users\liu\Desktop\hermes-backup.zip
```

The `hermes import` command handles platform-specific conversion that direct file copy cannot.

## What migrates

| Data | Method | Works? |
|------|--------|--------|
| Session history | backup → import | ✅ |
| Persistent memory | backup → import | ✅ |
| config.yaml | backup → import | ✅ (but may need tweaks) |
| .env (API keys) | manual copy | ✅ (same format) |
| state.db (raw) | direct copy | ❌ silently fails |

## Post-migration verification

```powershell
hermes sessions stats     # Should show all sessions
hermes memory status      # Should show built-in active
hermes config show        # Verify provider/model
```
