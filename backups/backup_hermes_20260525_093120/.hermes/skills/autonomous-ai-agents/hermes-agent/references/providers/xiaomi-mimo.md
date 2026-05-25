# Xiaomi MiMo Provider

## Overview

Xiaomi MiMo is Xiaomi's LLM service, accessible through [AI Studio](https://aistudio.xiaomimimo.com/). Unlike most providers, MiMo does **NOT** have a native OpenAI-compatible API.

## Authentication

MiMo uses **Cookie-based authentication**, not API keys:

1. Open https://aistudio.xiaomimimo.com/ and log in with Xiaomi account
2. F12 → Network → send a message → find the `/open-apis/bot/chat` request
3. Copy the full `Cookie:` header value (includes `serviceToken=...; userId=...; xiaomichatbot_ph=...`)

## API Endpoints

There is no public REST API. All access goes through the web-based `/open-apis/bot/chat` endpoint with Cookie auth.

## Protocol Bridge: mimo-2api

Since MiMo is not OpenAI-compatible, use the [mimo-2api](https://github.com/rong6/mimo-2api) proxy:

```bash
docker run -d --name mimo-2api -p 8090:8090 \
  mimo-2api \
  -cookie 'serviceToken=xxx; userId=xxx; xiaomichatbot_ph=xxx' \
  -apikey 'your-chosen-key'
```

Then configure Hermes to use it as a local OpenAI-compatible endpoint:

```bash
hermes config set model.provider openai_compatible
hermes config set model.base_url http://localhost:8090/v1
hermes config set model.default mimo-v2-flash
```

Set `OPENAI_API_KEY=your-chosen-key` in `~/.hermes/.env` (must match `-apikey` from mimo-2api).

## Models

- `mimo-v2-flash` — fast/lightweight
- `mimo-v2` — standard
- Vision/multimodal supported via `image_url` in OpenAI format (proxy handles conversion)

## Pitfalls

- **No direct API** — requires running a proxy (mimo-2api) 24/7
- **Cookie expiration** — sessions expire, requiring re-authentication and Cookie refresh
- **sk- prefix keys** — if you have a key starting with `sk-`, it may be a mimo-2api custom key, not a Xiaomi official key. Test via: `curl -H "Authorization: Bearer sk-xxx" http://localhost:8090/v1/models`
- **China network** — Xiaomi services are China-facing; expect latency from outside China
