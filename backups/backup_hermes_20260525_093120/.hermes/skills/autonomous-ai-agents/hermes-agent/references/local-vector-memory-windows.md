# Local Vector Memory on Windows (Qdrant + FastEmbed)

When Hermes's built-in memory (2,200 char limit) isn't enough, or when you need semantic search across many memories but cannot use Mem0 (torch DLL errors, missing API keys for LLM extraction, or network restrictions), this reference describes a fully-local alternative.

## Architecture

```
User request → vector search (semantic) → relevant memories injected as context
                        ↕
              Qdrant (local file mode)
                        ↕
              FastEmbed (ONNX, no GPU)
                        ↕
         BAAI/bge-small-zh-v1.5 (512-dim, Chinese-optimized)
```

- **Zero API keys** — everything runs locally
- **Zero GPU** — ONNX runtime, CPU only
- **Zero network** — model cached locally after first download
- **~150MB disk** — model + Qdrant store
- **Chinese-optimized** — uses `bge-small-zh-v1.5`

## Implementation

### Core Bridge Script

A self-contained `hermes_memory_bridge.py` (~100 lines) exposes CLI commands:

```bash
# Add a memory
python hermes_memory_bridge.py add "用户喜欢喝咖啡，每天早上一杯美式"

# Search semantically
python hermes_memory_bridge.py search "喝什么"
# Returns: [{"content": "用户喜欢喝咖啡...", "score": 0.67, "id": "abc123"}]

# List all
python hermes_memory_bridge.py list

# Count
python hermes_memory_bridge.py count
```

### Python API

```python
from hermes_memory_bridge import add, search

add("任何你想记住的内容")
results = search("相关查询", n=5)
for content, score, mid in results:
    print(f"[{score:.4f}] {content}")
```

### Storage Location

`~/.hermes/memory_store/` — Qdrant local file database, a single collection `hermes_memory`.

## Dependencies

```bash
pip install qdrant-client fastembed sentence-transformers -i https://pypi.tuna.tsinghua.edu.cn/simple
```

Set `HF_ENDPOINT=https://hf-mirror.com` (for users in China) before first use to download the embedding model from the mirror.

## Pitfalls & Gotchas

### Qdrant v1.18+ API Change
- `client.search()` was removed in Qdrant v1.18+. Use `client.query_points(collection_name=..., query=vector)` instead.
- The response is a `QueryResponse` object with a `.points` list (each has `.score`, `.payload`).

### FastEmbed Model Cache
The first run downloads the model from HuggingFace. With the China mirror (`HF_ENDPOINT`), the download succeeds but at ~100MB for `bge-small-zh-v1.5`. Subsequent runs use cached files under `~/.cache/fastembed/` or `C:\Users\<user>\.cache\fastembed\`.

### Storage Location Migration

The bridge script stores data at `~/.hermes/memory_store` by default. To change location (e.g., to a larger drive `F:`), modify `DB_PATH` in the bridge script. After changing, copy existing data before first use:
```python
import shutil
DB_PATH = r"F:\\hermes_memory_store"
shutil.copytree(r"C:\\Users\\xxx\\.hermes\\memory_store", r"F:\\hermes_memory_store")
```

In the desktop user's actual deployment, the memory store was migrated from `~/.hermes/memory_store` (C:) to `F:\\hermes_memory_store` (F: drive, the largest disk). After migration, verify by running `python hermes_memory_bridge.py count` and `python hermes_memory_bridge.py search "咖啡"` to confirm old memories exist and new memories persist.

**Recommended on multi-drive Windows systems**: Store on the largest drive to avoid filling C:/. The Qdrant store grows with memory count — ~50KB for 8 memories, expect ~1MB per 100 memories.

### Duplicate Detection
Qdrant does NOT deduplicate by content. When adding the same text multiple times, you get duplicate vectors. The `list_all()` function in the bridge script deduplicates by content at query time, but search results still return duplicates. To prevent this, check for existing content before adding, or use a content-hash dedup at the application layer.

### Qdrant Local Mode on Windows
Qdrant local mode (no server) uses `portalocker` for file locking. On Windows, this may emit harmless `ModuleNotFoundError: import of msvcrt halted` on shutdown — this is a clean-up race in Python's atexit handlers and does NOT affect data integrity. The file store remains consistent.

On Windows native Hermes, `execute_code` sandbox processes may trigger `ImportError: sys.meta_path is None, Python is likely shutting down` when Qdrant's destructor runs during interpreter shutdown. This is cosmetic — data is already flushed to disk before this error fires.

### Writing the Bridge Script on Windows Native
When using `write_file` (from `hermes_tools`) to save `hermes_memory_bridge.py` to `C:\Users\liu\Desktop\...`, the WSL-backed `terminal` may fail with path errors. Use `execute_code` + native `open()` instead:
```python
path = r"C:\Users\liu\Desktop\Hermes工具\hermes_memory_bridge.py"
with open(path, 'w', encoding='utf-8') as f:
    f.write(script_content)
```
This runs in the Windows Python sandbox and bypasses the WSL path translation layer entirely.

### Memory Format for Agent Context
For best results when injecting memories into an agent's context, prefix content with user identifiers:
```
用户喜欢喝咖啡，每天早上一杯美式
```
Rather than raw facts. This helps the search context match natural-language queries better.

### Why Not Mem0?
Mem0 v2.0 on Windows had these blockers:
1. **LLM dependency**: Mem0 requires a configured LLM for entity extraction — it refuses to initialize without `llm` config. Even with a dummy API key, every `add()`/`search()` call times out trying to reach the LLM.
2. **Torch PermissionError**: `sentence-transformers` → `transformers` → `torch` → `torch_python.dll` gets `WinError 5 Access Denied` when loaded from the execute_code sandbox.
3. **FastEmbed submodule**: Mem0's built-in fastembed integration tries to download the model at import time, which fails behind firewalls (unless HF_ENDPOINT is set).
4. **Config format quirks**: OpenAI-compatible embedding needs `openai_base_url` (not `base_url`) in the embedder config dict.
5. **V2 API changes**: `search()` rejects `user_id` as top-level kwarg — requires `filters={"user_id": "..."}`. Same for `get_all()`.
