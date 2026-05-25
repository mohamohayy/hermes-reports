# 微信会话模型不一致排查记录

## 问题

老板从微信发消息，回复说用的是 `xiaomi/mimo-v2.5-pro`（OpenRouter），但 `hermes config show` 显示 model.provider=deepseek, model.default=deepseek-chat。

## 排查路径

### Step 1: 确认 config.yaml

```bash
hermes config show
```
→ Model: {provider: deepseek, default: deepseek-chat, base_url: https://api.deepseek.com/v1}

### Step 2: 检查 .env 环境变量

`~/.hermes/.env` 中无模型相关覆盖，只有：
- DEEPSEEK_API_KEY
- OPENROUTER_API_KEY（存在但未被使用）
- GATEWAY_ALLOW_ALL_USERS=true
- WEIXIN_HOME_CHANNEL=o9cq809a…@im.wechat

### Step 3: 查看微信会话历史（关键）

读取 `~/.hermes/sessions/20260522_145757_5b8a9221.jsonl` 的 last 5 条消息，发现：

```
[assistant]: 当前会话用的是 OpenRouter → xiaomi/mimo-v2.5-pro，不是 config.yaml 里配的 DeepSeek。
```

证明会话内部有模型级别覆盖。

### Step 4: 查看 gateway.log 时间线

```
15:28:14 — 用户问 "现在会话用的什么模型"
15:28:23 — 回复 231 chars（回复说用的 xiaomi/mimo）
15:29:08 — 用户说 "切换到deepseek"
15:30:09 — 网关重启（因切换指令自动触发）
15:30:26 — 回复 228 chars（确认切换）
```

但配置从 session jsonl 看仍然残留了 OpenRouter 路由信息。

## 根因

### Path A: 会话内动态路由（原记录）

1. 之前某个微信会话中，agent 响应了"切换模型"的请求，通过 tool call 将**会话内模型路由**指向了 OpenRouter → xiaomi/mimo-v2.5-pro
2. 此后同一用户的所有微信消息都进入这个会话上下文，继承该路由
3. `config.yaml` 的 `model.*` 只影响 CLI 启动的新会话，不影响已有网关会话内部的动态路由
4. 网关重启（`hermes gateway run --replace`）会销毁旧会话缓存，新消息创建新会话时重新读取 config.yaml

### Path B: Credential Pool Fallback（新增，2026-05-22 遇到）

当 DeepSeek API key 被官方封禁（401 invalid）时，Hermes 网关的 credential pool 会回退使用其他可用的 provider key。具体流程：

1. `.env` 中有 `DEEPSEEK_API_KEY`（已被封）和 `OPENROUTER_API_KEY`（有效）
2. `auth.json` 中 `api_keys: {}` 为空（没有手动添加的 key）
3. Gateway 尝试用 DEEPSEEK_API_KEY → 401 → 标记为 exhausted
4. Gateway 回退到 OpenRouter（因为 OPENROUTER_API_KEY 在 .env 中可用）
5. OpenRouter 上有个小米模型路由（xiaomi/mimo-v2.5-pro）→ 回复成功
6. 结果：config.yaml 写的是 deepseek，实际回复却是 OpenRouter 的小米模型

**诊断方法：**
```bash
# 检查 credential pool 状态
python3 -c "
import json
f = open(os.path.expanduser('~/.hermes/credential_pool_state.json'))
state = json.load(f)
for entry in state.get('pool', []):
    print(f\"{entry.get('label','?'):20s} provider={entry.get('provider','?'):15s} status={entry.get('last_status','?'):15s} error={entry.get('last_error_message','')[:60]}\")
"
```

注意：DeepSeek 官方 401 表示 key 被封禁或未充值。即使 key 在 DeepSeek 官网 dashboard 中显示为 active，如果没充值也可能被 API 层拒绝。需要去 deepseek.com 重新生成 key 并充值。

**修复（Path B）：**
1. 获取有效的 DeepSeek API key（新充值或重新生成）
2. 添加到 `auth.json` 的 `api_keys` 部分
3. 重启网关：`hermes gateway run --replace`
4. 发送新消息验证

或者移除 OpenRouter key 避免回退（不推荐，会完全断掉 AI 能力）。

## 关键判断点

```python
import subprocess
subprocess.run(["hermes", "config", "set", "model.provider", "deepseek"])
subprocess.run(["hermes", "config", "set", "model.default", "deepseek-chat"])
subprocess.run(["hermes", "config", "set", "model.base_url", "https://api.deepseek.com/v1"])
# 重启网关清除会话缓存
subprocess.run(["hermes", "gateway", "run", "--replace"])
```

然后从微信发一条新消息，会创建新会话，走 config.yaml 的默认模型。

## 关键判断点

- **config.yaml 模型 ≠ 网关会话实际模型**。这是 Hermes 网关设计中容易混淆的一点
- 会话 jsonl 中的 assistant 回复是判断实际路由的权威来源
- gateway.log 不记录具体模型名（只记录 chars），必须看 jsonl
- 微信会话按 `<platform>:<user>@im.wechat` 的 chat_id 做 session key，同一个用户始终进入同一个会话
