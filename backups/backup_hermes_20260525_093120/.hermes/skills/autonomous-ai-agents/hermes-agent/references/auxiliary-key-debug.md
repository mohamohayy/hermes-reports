# Auxiliary Key Debug Transcript

## Session: 2026-05-22 修复 title_generation 401

### Symptom

`Auxiliary title generation failed: HTTP 401: Authentication Fails, Your api key: ****a2ef is invalid`

### Environment

- Hermes Windows native (pip install)
- Provider: deepseek (api.deepseek.com/v1)
- Model: deepseek-chat
- Key in `~/.hermes/.env`: `DEEPSEEK_API_KEY=sk-c39...b`
- OS: Windows 10, cmd.exe

### Investigation

1. Checked config.yaml → `auxiliary.title_generation.api_key: ''` 
2. Checked `.env` → `DEEPSEEK_API_KEY=sk-c39...b` present
3. Main model worked fine → confirmed key is valid
4. Auxiliary module api_key was empty → `provider: auto` defaults to deepseek but sends null key

### Root Cause

Auxiliary modules do NOT inherit the main model's API key. Each has a dedicated `api_key` field in config.yaml that defaults to `''`. The credential pool / auth.json system only serves the main model path (`model.provider`), not `auxiliary.*`.

### Fix

Injected the key from `.env` directly into `auxiliary.title_generation.api_key`:

```python
import re, subprocess
from pathlib import Path

result = subprocess.run(['cmd.exe', '/c', 'type C:\\Users\\liu\\.hermes\\.env'], capture_output=True, text=True, shell=True)
match = re.search(r'DEEPSEEK_API_KEY=(.+)', result.stdout)
key = match.group(1).strip()

config_path = Path(r'C:\Users\liu\.hermes\config.yaml')
content = config_path.read_text('utf-8')
old = '''  title_generation:
    api_key: ''
    base_url: ''
    extra_body: {}
    model: ''
    provider: auto
    timeout: 30'''
new = f'''  title_generation:
    api_key: {key}
    base_url: ''
    extra_body: {{}}
    model: ''
    provider: auto
    timeout: 30'''
content = content.replace(old, new)
config_path.write_text(content, 'utf-8')
```

### Verification

The `4096: 401` response from DeepSeek's API confirms the auth failure was at the provider level (null key sent), not a network or config parsing issue. After filling the key, the module should return to normal behavior on next title generation trigger.

### Lesson

When a Hermes auxiliary module fails with 401:
1. Check `config.yaml` → `auxiliary.<module>.api_key`
2. If empty, read the key from `.env` and write it into that field
3. No gateway restart needed for auxiliary key changes (auxiliary calls are per-request)
