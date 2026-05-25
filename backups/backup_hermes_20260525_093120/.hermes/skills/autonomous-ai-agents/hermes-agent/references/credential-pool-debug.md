# Credential Pool & Provider Fallback Debugging

## Architecture

Hermes Gateway has a **credential pool** that manages API keys across providers.
When the primary provider fails (401, 429, 503, connection timeout), the gateway
can fall back to an alternative provider if one is configured.

### Key Storage Locations

| Location | Purpose | Used By |
|----------|---------|---------|
| `~/.hermes/.env` | Environment variables (legacy) | CLI, some gateway startup code |
| `~/.hermes/auth.json` | Structured credential pool | Gateway credential manager |
| `~/.hermes/config.yaml` model.* | Primary model config | New sessions |

### Resolution Order (for a given provider)

1. `auth.json: api_keys[provider]` — if populated, use it
2. `auth.json: api_keys[provider][labeled_keys]` — credential pool with labels
3. `.env: <PROVIDER>_API_KEY` — environment variable fallback
4. `config.yaml: model.base_url + empty key` — may cause 401

### Fallback Chain

When the **main model** (model.provider + model.default) fails:

1. Gateway marks that key as exhausted in a runtime state
2. If `fallback_providers: []` (empty), no automatic fallback
3. If `credential_pool_strategies` has `fill_first` for that provider, it tries
   alternative keys for the same provider first
4. If the provider itself has no keys left, the gateway may pick the next
   provider from available keys in auth.json (e.g., OpenRouter key → any model)

**Important:** The fallback is NOT a list of providers in order — it's more of a
"find any working key" strategy when the primary fails.

## How to Diagnose

### 1. Check credential pool state

On Windows native (via execute_code or subprocess):
```python
import subprocess, json, os

# Read credential pool state
pool_path = os.path.expanduser("~/.hermes/credential_pool_state.json")
if os.path.exists(pool_path):
    with open(pool_path) as f:
        state = json.load(f)
    for entry in state.get("pool", []):
        print(f"label={entry.get('label','?'):20s} "
              f"provider={entry.get('provider','?'):15s} "
              f"status={entry.get('last_status','?'):15s} "
              f"error={entry.get('last_error_message','')[:60]}")
else:
    print("No credential_pool_state.json (gateway not started or pool empty)")
```

### 2. Check auth.json directly

```python
with open(os.path.expanduser("~/.hermes/auth.json")) as f:
    auth = json.load(f)
if "api_keys" in auth:
    for provider, keys in auth["api_keys"].items():
        print(f"{provider}:")
        if isinstance(keys, dict):
            for label, key in keys.items():
                print(f"  {label}: {key[:8]}...{key[-4:]}")
        else:
            print(f"  {keys[:8]}...{keys[-4:]}")
else:
    print("auth.json has no api_keys section")
```

### 3. Check .env for all API keys

```python
with open(os.path.expanduser("~/.hermes/.env")) as f:
    for line in f:
        line = line.strip()
        if "_API_KEY=" in line or "_TOKEN=" in line:
            key, val = line.split("=", 1)
            print(f"  {key}: {'SET' if val else 'EMPTY'}")
```

### 4. Check gateway logs for fallback events

```python
log_path = os.path.expanduser("~/.hermes/logs/gateway.log")
with open(log_path) as f:
    for line in f:
        if "fallback" in line.lower() or "401" in line or "exhausted" in line.lower():
            print(line.strip())
```

## Common Scenarios

### Scenario A: Model Mismatch (config says DeepSeek, reply says OpenRouter)

**Cause:** DeepSeek API key is invalid (401). Gateway falls back to OpenRouter
key from .env. On OpenRouter, there's a model route mapped to xiaomi/mimo-v2.5-pro
that the gateway uses as default OpenRouter model.

**Symptoms:**
- config.yaml shows provider=deepseek, model=deepseek-chat
- Reply says "当前会话用的是 OpenRouter → xiaomi/mimo-v2.5-pro"
- No error in gateway.log about DeepSeek (the 401 is silently handled by pool)

**Fix:**
1. Get a valid DeepSeek API key (new key, recharged)
2. Add to auth.json or update .env
3. Restart gateway: `hermes gateway run --replace`
4. Send a new message to trigger a fresh session

### Scenario B: auth.json api_keys empty, all keys in .env

This is the normal state for many installs. Gateways reads .env as fallback.
Works fine until any .env key is invalid — the gateway has no structured pool
to manage expiry or rotation.

**Fix:** Migrate keys to auth.json:
```json
{
  "api_keys": {
    "deepseek": ["sk-xxxx", "sk-yyyy"],
    "openrouter": ["sk-zzzz"]
  }
}
```

This enables `fill_first` strategy (try key 1, if exhausted try key 2).

### Scenario C: Gateway won't start after adding auth.json keys

**Check:** Ensure auth.json is valid JSON. A trailing comma is silently tolerated
by Python's json.loads but may cause issues with the gateway's parser:
```bash
python3 -m json.tool ~/.hermes/auth.json  # validate
```
