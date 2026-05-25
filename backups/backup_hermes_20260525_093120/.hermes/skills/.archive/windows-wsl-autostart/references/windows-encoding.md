# Chinese Windows Encoding Pitfalls

## Default Code Page

Chinese Windows (Simplified) uses **GBK (codepage 936)** as the default console code page.
CMD.EXE reads `.bat` files as ANSI/GBK, **not UTF-8**.

## Symptoms

| Symptom | Root Cause |
|---------|-----------|
| Chinese chars show as garbled (乱码) in CMD window | `.bat` saved as UTF-8 without BOM |
| "不是可运行的程序或批处理文件" | File corrupted by encoding conversion (e.g. `iconv` artifacts) |
| `chcp 65001` doesn't fix garbled output | `chcp 65001` support is incomplete on older Win10 builds |

## Safe Approaches (ranked)

### 1. Pure ASCII (recommended for delivery)
No Chinese characters. CMD reads ASCII correctly regardless of codepage.
This is the approach used in `templates/install-autostart.bat`.

### 2. Save as ANSI/GBK from Windows Notepad
User opens the file in Notepad → "Save As" → encoding dropdown → "ANSI".
This is the most reliable manual fix.

### 3. Write from Python with `encoding='gbk'`
```python
with open('/mnt/c/Users/.../file.bat', 'w', encoding='gbk') as f:
    f.write(content)
```
Works when scripting from WSL.

## Dangerous Approaches

| Approach | Why it fails |
|----------|-------------|
| `iconv -f UTF-8 -t GBK` from WSL | Can produce BOM or line-ending artifacts that break CMD parsing |
| `chcp 65001` in `.bat` | Unreliable across Win10 builds; may fix display but not parsing |
| UTF-8 with BOM | CMD ignores BOM, treats as raw ANSI → garbled |

## Verification

After writing a `.bat` file from WSL:
```bash
file /mnt/c/Users/liu/Desktop/file.bat
# Should show: "ASCII text" or "ISO-8859 text"
# Should NOT show: "UTF-8 Unicode text"
```
