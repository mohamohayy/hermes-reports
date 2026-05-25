# Xiaomi MiMo API — Provider Notes

## Overview

Xiaomi MiMo (小米 MiMo) is accessed via web studio at `https://aistudio.xiaomimimo.com/`. It does **NOT** offer a native OpenAI-compatible API — authentication is Cookie-based, not API-key-based.

## Key Discovery

- **No standard OpenAI endpoint.** Attempts to call `api.minimax.chat`, `api.xiaomi.com`, or any OpenRouter-style base URL with `Bearer sk-xxx` will fail (error 1004: `login fail`).
- **Auth is Cookie-based.** The web studio uses cookies (`serviceToken=xxx; userId=xxx; xiaomichatbot_ph=xxx`) for session authentication.
- **No public API documentation found** for an OpenAI-compatible interface.

## Proxy Solution: `rong6/mimo-2api`

A Go-based proxy that converts MiMo's web Cookie auth to a standard OpenAI Chat Completions API:

```
GitHub: rong6/mimo-2api
Purpose: Xiaomi MiMo Studio web API → OpenAI-compatible /v1/chat/completions
Features: multi-account load balancing, session save, debug mode
```

### Setup

```bash
# Get Cookie from browser (F12 → Network → /open-apis/bot/chat → Request Headers → Cookie)
# Run proxy
./mimo-2api -cookie 'serviceToken=xxx; userId=123; xiaomichatbot_ph=xxx' -apikey 'your-key'

# Then use as standard OpenAI endpoint:
# base_url = http://localhost:8090/v1
# api_key = your-key
# model = mimo-v2-flash
```

### Caveat

The proxy is an external community project — not officially maintained by Xiaomi. It may break when MiMo's web API changes. Use as a stopgap, not a production dependency.

## Attempted Endpoints (all failed)

| Endpoint | Error |
|----------|-------|
| `api.minimax.chat/v1/...` | HTTP 401, error 1004 |
| `api.minimaxi.com/v1/...` | HTTP 401, error 1004 |
| `api.xiaomi.com/v1/...` | DNS resolve failure |
| `mimo-api.xiaomi.com/v1/...` | DNS resolve failure |

## Conclusion

MiMo is not usable as a drop-in provider for Hermes without the `mimo-2api` proxy bridge. For the user's setup (DeepSeek primary), MiMo is not worth the proxy overhead unless specific MiMo-only features are needed (e.g., MiMo TTS, MiMo multimodal).
