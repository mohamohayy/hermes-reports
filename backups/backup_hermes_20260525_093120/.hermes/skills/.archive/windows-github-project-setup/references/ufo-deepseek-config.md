# UFO Project - DeepSeek Configuration

## Context

Microsoft UFO is a Windows-native AI agent framework. Its modular config lives in
`config/ufo/agents.yaml` (copied from `agents.yaml.template`). This reference
documents configuring it for DeepSeek API, which is OpenAI-compatible.

## agents.yaml Template (DeepSeek)

Replace ALL agent blocks (HOST_AGENT, APP_AGENT, BACKUP_AGENT, EVALUATION_AGENT):

```yaml
HOST_AGENT:
  VISUAL_MODE: True
  REASONING_MODEL: False
  API_TYPE: "openai"           # DeepSeek is OpenAI-compatible
  API_BASE: "https://api.deepseek.com/v1/chat/completions"
  API_KEY: "sk-xxxxxxxx"
  API_VERSION: ""               # Not needed for DeepSeek
  API_MODEL: "deepseek-chat"
```

Repeat for APP_AGENT, BACKUP_AGENT, EVALUATION_AGENT with same values.
All 4 agents can share a single API key.

## Key Points

- `API_TYPE` stays "openai" — DeepSeek API is fully OpenAI-compatible
- `API_BASE` must include the full `/v1/chat/completions` path (UFO appends nothing)
- `API_VERSION` must be empty string or absent — DeepSeek doesn't use it
- `deepseek-chat` maps to DeepSeek-V3; for reasoning tasks use `deepseek-reasoner`
- The old-style `ufo/config/config.yaml` is legacy; prefer `config/ufo/agents.yaml`

## Verifying Config

```python
import yaml
with open("config/ufo/agents.yaml") as f:
    config = yaml.safe_load(f)
for name in ["HOST_AGENT", "APP_AGENT", "BACKUP_AGENT", "EVALUATION_AGENT"]:
    a = config[name]
    assert a["API_KEY"].startswith("sk-"), f"{name}: missing API key"
    assert a["API_BASE"] != "", f"{name}: empty API_BASE"
```

## Running UFO

```cmd
cd C:\Users\liu\Desktop\UFO
python -m ufo
```
