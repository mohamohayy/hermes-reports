# China Network Workarounds for Development

Consolidated list of network-related workarounds needed when developing from mainland China.
All of these are encountered repeatedly — don't rediscover them each session.

## GitHub Downloads (Releases, raw files)

GitHub releases download slowly (~100KB/s) or time out from China. Use ghproxy as a caching proxy:

```bash
# Original
curl -L -o file.exe "https://github.com/USER/REPO/releases/download/TAG/ASSET"

# Accelerated
curl -L -o file.exe "https://ghproxy.net/https://github.com/USER/REPO/releases/download/TAG/ASSET"
```

Typical improvement: 100KB/s → 2-5MB/s. For files < 50MB, direct download may be acceptable.

**ghproxy reliability note:** `ghproxy.net` sometimes returns HTTP/2 PROTOCOL_ERROR for `git clone` operations. For releases, it usually works; for repo cloning, see the section below.

## Git Clone / Repo Download

Cloning repos from China is significantly harder than downloading releases. Multiple mirrors have been tested:

| Mirror | URL Pattern | Result |
|--------|-------------|--------|
| ghproxy.net | `https://ghproxy.net/https://github.com/USER/REPO.git` | Unreliable for clone (HTTP/2 error). OK for release assets. |
| codeload.github.com | `https://codeload.github.com/USER/REPO/zip/refs/heads/main` | Unreliable (timeout after partial download ~6MB in 300s). |
| hub.gitmirror.com | `https://hub.gitmirror.com/https://github.com/USER/REPO/...` | DNS resolution timeout. |
| gitclone.com | `https://gitclone.com/github.com/USER/REPO.git` | 403 Forbidden. |

**Recommended strategy (in priority order):**

1. **Browser download (most reliable):** Give the user the raw GitHub URL — e.g. `https://github.com/USER/REPO/archive/refs/heads/main.zip` — and let them download via Windows browser, which often has better connectivity than WSL CLI. Put the downloaded file on Desktop, then process from WSL via `/mnt/c/Users/<user>/Desktop/`.

2. **ghproxy for release assets** (`Setup.exe`, `.tar.gz` releases): First choice for CLI download. Works reliably for release artifacts.

3. **Direct `codeload.github.com`** with long timeout (300s+): May work for smaller repos (< 20MB). Use `--depth=1` for git clone to minimize data transfer.

4. **`git clone --depth=1`** directly: Worth trying — sometimes succeeds where full clones fail. The shallow clone reduces transfer size dramatically.

When all CLI methods fail, fall back to browser download immediately rather than cycling through mirrors. The user's browser often has proxy/VPN configured that WSL doesn't.

## pip (PyPI mirror)

Default PyPI times out. Always add Tsinghua mirror:

```bash
pip install <package> -i https://pypi.tuna.tsinghua.edu.cn/simple
```

## npm / pnpm

npm/pnpm may be slow but usually work. If needed:

```bash
npm config set registry https://registry.npmmirror.com
pnpm config set registry https://registry.npmmirror.com
```

## api.anthropic.com

Directly blocked. Requires proxy/VPN. Claude Code CLI needs proxy configured separately (see claude-code skill for details).

## api.openai.com

Usually blocked. Use api-key-online or similar proxy services, or configure HTTP_PROXY.
