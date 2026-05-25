# state.db Corruption — Detection & Repair

## Symptoms

In `agent.log`:
```
Session DB append_message failed: database disk image is malformed
```

The gateway still runs and processes messages, but sessions/memory may not persist. Messages may be lost.

## Common causes

- Improper shutdown (WSL killed, `taskkill`, power loss while writing)
- Cross-platform usage without proper WAL cleanup
- Disk full or filesystem errors

## Repair (Windows native)

```powershell
# 1. Stop gateway first
hermes gateway stop

# 2. Backup the broken DB just in case
copy %USERPROFILE%\.hermes\state.db %USERPROFILE%\.hermes\state.db.broken
copy %USERPROFILE%\.hermes\state.db-wal %USERPROFILE%\.hermes\state.db-wal.broken
copy %USERPROFILE%\.hermes\state.db-shm %USERPROFILE%\.hermes\state.db-shm.broken

# 3. Attempt SQLite recovery
sqlite3 %USERPROFILE%\.hermes\state.db ".recover" > %USERPROFILE%\.hermes\state_recovered.sql
sqlite3 %USERPROFILE%\.hermes\state_new.db < %USERPROFILE%\.hermes\state_recovered.sql

# 4. Replace if recovery succeeded
move /Y %USERPROFILE%\.hermes\state_new.db %USERPROFILE%\.hermes\state.db
del %USERPROFILE%\.hermes\state.db-wal %USERPROFILE%\.hermes\state.db-shm

# 5. Restart
hermes gateway run
```

If `.recover` also fails, the DB is beyond repair. Delete `state.db`, `state.db-wal`, `state.db-shm` — Hermes will recreate an empty one. Persistent memory (from `memory` tool) will be lost but re-accumulates over subsequent sessions.
