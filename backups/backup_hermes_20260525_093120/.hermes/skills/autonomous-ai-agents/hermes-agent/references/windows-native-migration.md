# Windows Native Hermes — WSL State Migration

Moving state.db (sessions + memory) from WSL Hermes to Windows native Hermes.

## Problem

Direct `state.db` copy from Linux to Windows fails:
- `cp ~/.hermes/state.db /mnt/c/Users/liu/.hermes/state.db` → `database disk image is malformed`
- `hermes backup` on WSL → `hermes import` on Windows → same malformed error
- Even `hermes sessions stats` after import reports empty database (0 sessions)

## Why

SQLite between Linux (ext4, WAL-mode, specific page size) and Windows (NTFS, DrvFs 9P protocol) is incompatible at the binary level. The `hermes import` copies the raw file but Windows Hermes can't read it.

## Working Approach: SQL Dump & Restore

### Step 1: Dump on WSL

```bash
python3 -c "
import sqlite3
src = sqlite3.connect('/home/liu123456/.hermes/state.db')
with open('/mnt/c/Users/liu/Desktop/hermes_dump.sql', 'w', encoding='utf-8') as f:
    for line in src.iterdump():
        f.write(line + '\n')
src.close()
"
```

### Step 2: Kill conflicting processes on Windows

```bash
powershell.exe -Command "Get-Process python -ErrorAction SilentlyContinue | Stop-Process -Force"
```

The state.db file may be held open by a rogue Python or Hermes process.

### Step 3: Delete existing state.db

```bash
powershell.exe -Command "Remove-Item -Path 'C:\Users\liu\.hermes\state.db' -Force"
# Verify deletion
ls -la /mnt/c/Users/liu/.hermes/state.db  # should say "No such file"
```

### Step 4: Import dump via .ps1 (ASCII-only)

Write a `.ps1` file to Desktop (NEVER inline — `$env:USERPROFILE` gets eaten by bash):

```powershell
$dumpPath = "C:\Users\liu\Desktop\hermes_dump.sql"
$dbPath = "C:\Users\liu\.hermes\state.db"

Remove-Item -Path $dbPath -Force -ErrorAction SilentlyContinue

& python -c @"
import sqlite3
db = sqlite3.connect(r'$dbPath')
with open(r'$dumpPath', 'r', encoding='utf-8') as f:
    db.executescript(f.read())
db.commit()
db.close()
print('Import done')
"@

# Verify
& python -c @"
import sqlite3
db = sqlite3.connect(r'$dbPath')
print('Integrity:', db.execute('PRAGMA integrity_check').fetchone()[0])
print('Sessions:', db.execute('SELECT COUNT(*) FROM sessions').fetchone()[0])
print('Messages:', db.execute('SELECT COUNT(*) FROM messages').fetchone()[0])
db.close()
"@
```

Run from WSL:
```bash
powershell.exe -ExecutionPolicy Bypass -File "C:\Users\liu\Desktop\restore_db.ps1"
```

### Step 5: Verify via Hermes

```bash
powershell.exe -Command "hermes sessions stats"
```

## Pitfalls

1. **`$env:USERPROFILE` eaten by bash**: When running `powershell.exe -Command` from WSL, bash eats `$` before PowerShell sees it. Always use hardcoded `C:\Users\liu` paths in .ps1 files, or escape as `\$env:USERPROFILE`.

2. **State.db file lock**: If any Hermes or Python process holds the file, `Remove-Item` silently fails. Kill processes first.

3. **Hermes auto-recreates state.db**: Running `hermes sessions stats` on an empty dir auto-creates a fresh (empty) state.db. The SQL dump will then fail with "table already exists". Always do the full delete→import in one script WITHOUT running any Hermes commands in between.

4. **Dump encoding**: Use `encoding='utf-8'` for the SQL dump. The Windows Python import must also read as UTF-8.

5. **`hermes import` from backup zip doesn't work for state.db**: The backup copies the binary file, which has the same cross-platform incompatibility. SQL dump is the only working path.
