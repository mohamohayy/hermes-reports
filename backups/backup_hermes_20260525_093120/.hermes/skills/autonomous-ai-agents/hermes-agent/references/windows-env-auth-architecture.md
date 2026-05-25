# Windows Native Hermes — .env vs auth.json 密钥架构

## 问题背景

发现 config.yaml 配了 DeepSeek，但微信回复显示走的是 OpenRouter → xiaomi/mimo。
检查发现 `auth.json` 中 `api_keys` 为 `{}`（空），而 `DEEPSEEK_API_KEY` 只存在 `.env` 中。

## 密钥读取链路

Hermes 的 API key 读取顺序（Windows Native）：

1. **环境变量** — `os.environ`（进程启动时已加载）
   - Gateway 进程启动时自动读取 `~/.hermes/.env` 并注入 `os.environ`
   - 因此 gateway 能正常用 deepseek
2. **auth.json** — `~/.hermes/auth.json` 的 `api_keys` 字段
   - 未分类的 provider 会从这里读
   - 如果为空但 `.env` 有值，gateway 仍正常工作

**关键区别：**
- **Gateway 进程** → 启动时加载 `.env` → `os.environ` 有 `DEEPSEEK_API_KEY`
- **`execute_code` Python 子进程** → 继承父进程（Hermes agent）的 `os.environ`，但父进程**没有**加载 `.env` → `DEEPSEEK_API_KEY` 不存在
- **`terminal` bash 子进程** → 纯 bash 环境，同样没有 `.env` 变量

## 诊断步骤

### 1. 检查 auth.json api_keys
```python
import json
auth = json.load(open(os.path.expanduser("~/.hermes/auth.json")))
auth.get("api_keys", {})  # 空 = 密钥走 .env 而非 auth.json
```

### 2. 检查 .env 内容
```python
for line in open(os.path.expanduser("~/.hermes/.env")):
    if "API_KEY" in line or "TOKEN" in line:
        print(line.strip()[:50])
```

### 3. 检查当前 Python 进程能否看到 key
```python
os.environ.get("DEEPSEEK_API_KEY")  # 在 execute_code 中可能为 None
```

### 4. 检查 gateway 进程实际使用的模型
```python
import subprocess
subprocess.run(["hermes", "config", "show"], capture_output=True, text=True)
```
→ 确认 model.provider / model.default / model.base_url

### 5. 如果 config 对了但回复仍用其他模型
说明是**会话级模型覆盖**（agent 前序 turn 通过 tool call 修改了路由）。
修复：重启网关清除会话缓存：
```bash
hermes gateway run --replace
```

## 为什么 auth.json 可以是空的

- `.env` 是 Hermes 的**主密钥源**（gateway 进程级）
- `auth.json` 的 `api_keys` 用于 provider 发现和认证池（`credential_pool_strategies`）
- 单 provider（deepseek）场景下 `.env` 足够，`auth.json` 空是正常的
- 多个 API key 轮换时才需要往 `auth.json` 加

## execute_code 看不到 key 是否影响功能？

**通常不影响。** `execute_code` 用于 Python 脚本处理，不需要直接调用 LLM API。只有以下场景才需要：
- 在 execute_code 里手动调用 OpenAI-compatible API（curl / httpx）
- 在 execute_code 里跑 litellm / one-api 其他工具

这些场景下，需要在 execute_code 代码中手动 `load_dotenv()` 或传 key。

## 总结

| 组件 | 能看到 .env key? |
|------|-----------------|
| Gateway 进程 | ✅ 自动加载 |
| `execute_code` Python | ❌ 继承不到 |
| `terminal` bash | ❌ 继承不到 |
| 新 CLI 会话 | ✅ (gateway 下走平台) |

## Auxiliary 模块密钥继承问题

**关键发现：** `auxiliary.*` 模块的 `api_key` 字段**独立于主模型和 `.env`**。

### 密钥链路差异

```
主模型认证链路：
  config.yaml model.provider=deepseek → credential_pool
    → auth.json 或 .env DEEPSEEK_API_KEY → ✅ 成功

Auxiliary 模块认证链路：
  config.yaml auxiliary.title_generation.api_key=''
    → provider=auto 推断为 deepseek
    → 但 api_key 为空字符串 → 发送 null key → ❹❹ 401
```

### 为什么会有这个差异

- `model.*` 配置走的是 Hermes 的 **credential pool** 系统：
  `auth.json` 注册 key → credential pool 按 `credential_pool_strategies` 分配 → 对主模型透明
- `auxiliary.*` 模块是**独立工具调用**，其 `api_key` 字段是独立发送给 LLM 的，不走 credential pool
- 当 `auxiliary.*.api_key` 为 `''` 时，Hermes 不尝试从 `.env` 或 `auth.json` 补全——直接传空值
- 这是 Hermes 当前架构的限制，不是 bug

### 影响范围

`auxiliary:` 下所有模块都有这个问题：
- `title_generation` — 自动生成对话标题（报错频率最高）
- `compression` — 上下文压缩
- `vision` — 图片分析
- `session_search` — 跨会话搜索
- `approval` — 审批确认
- `curator` — 会话整理
- `mcp` — MCP 工具
- `skills_hub` — 技能仓库
- `triage_specifier` — 分诊
- `web_extract` — 网页提取

### 修复方案

1. **手动配平**：把 `.env` 中的 key 复制到每个 `auxiliary.<模块>.api_key`
2. **自动化脚本**：见下面的诊断脚本，或参考 `references/auxiliary-key-debug.md`
3. **新安装预防**：在配完主模型后立即执行全量注入

### 诊断脚本

```python
# 快速检查哪些 auxiliary 模块的 api_key 为空
import yaml
from pathlib import Path
config = yaml.safe_load(Path.home().joinpath('.hermes', 'config.yaml').read_text())
for mod, cfg in config.get('auxiliary', {}).items():
    key = cfg.get('api_key', '')
    status = '❌ EMPTY' if not key else '✅ OK'
    print(f"  {mod:20s}: api_key={status}")
```

### 批量修复脚本

```python
# 从 .env 读取 key 并注入所有空的 auxiliary api_key
import re, yaml
from pathlib import Path

env_path = Path.home() / '.hermes' / '.env'
config_path = Path.home() / '.hermes' / 'config.yaml'

# 读取 key
env_text = env_path.read_text('utf-8')
match = re.search(r'DEEPSEEK_API_KEY=(.+)', env_text)
if not match:
    print("❌ 未在 .env 找到 DEEPSEEK_API_KEY")
    exit(1)
key = match.group(1).strip()

# 读取并修改 config
config = yaml.safe_load(config_path.read_text('utf-8'))
changed = []
for mod, cfg in config.get('auxiliary', {}).items():
    if isinstance(cfg, dict) and cfg.get('api_key') == '':
        cfg['api_key'] = key
        changed.append(mod)

if changed:
    config_path.write_text(
        yaml.dump(config, default_flow_style=False, allow_unicode=True),
        'utf-8'
    )
    print(f"✅ 已修复: {', '.join(changed)}")
else:
    print("✅ 所有 auxiliary api_key 已配置")
```

对应 pitfall 在 SKILL.md 的 Troubleshooting 章节。
