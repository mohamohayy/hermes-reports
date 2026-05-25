# Windows Desktop Organization

When the user asks to "clean up the desktop" or "tidy up files", use this workflow to organize the Windows Desktop efficiently.

## Classification Categories

| Category | Matches | Destination folder |
|----------|---------|-------------------|
| Shortcuts | `.lnk`, `.url` | `快捷方式/` |
| Documents | `.docx`, `.pdf`, `.pptx`, `.xlsx`, `.txt` | `文档/` |
| WeChat scripts | files with `微信`, `bot`, `wechat`, `gateway` in name | `微信工具/` (subfolder of `Hermes工具/`) |
| UFO installers | `UFO_*` | `UFO相关/` |
| Agent scripts | `.ps1`, `.py`, `.bat` (non-hermes) | `Hermes工具/微信机器人/` |
| hermes-auto-start | `hermes-auto-start.xml` | `Hermes工具/` |

## Workflow

1. **Plan before touching**: use `os.listdir(desktop)` to scan, categorize each file, print the plan
2. **Create destination folders**: `os.makedirs()` for each category
3. **Move files**: `shutil.move(src, dst)` — NEVER delete files the user didn't explicitly ask to delete
4. **Verify**: final `os.listdir()` to confirm Desktop is reduced to folders only

## Pitfalls

- `desktop.ini` must be excluded from all operations
- Existing folders (UFO/, novels/, ppt_images/) stay where they are — only consolidate loose files
- The `Hermes工具/` folder hierarchy must be preserved: `Hermes工具/微信机器人/` for scripts, `Hermes工具/截图缓存/` for screenshots
