# Windows Native Hermes Installation (v0.14.0+)

从 v0.14.0 起，Hermes 支持原生 Windows（cmd.exe / PowerShell），无需 WSL。

## 安装

```powershell
# 前置：Windows 上需要 Python 3.11+
python --version
pip --version

# 一步安装（国内用清华镜像加速）
pip install hermes-agent -i https://pypi.tuna.tsinghua.edu.cn/simple
```

验证：
```powershell
hermes --version
# Hermes Agent v0.14.0+ — 显示 Windows 路径则成功
```

## 配置

Windows 的 Hermes home 在 `%USERPROFILE%\.hermes\`（即 `C:\Users\<username>\.hermes\`）。

### 从 WSL 中配置（跨平台操作）

如果人在 WSL 里，要通过 `powershell.exe` 操控 Windows 版：

```bash
# 设置模型和 provider
powershell.exe -Command "hermes config set model.default deepseek-v4-pro"
powershell.exe -Command "hermes config set model.provider deepseek"
powershell.exe -Command "hermes config set model.base_url https://api.deepseek.com/v1"
powershell.exe -Command "hermes config set display.language zh"
```

## 跨平台记忆迁移

**❌ 不要直接复制 state.db！** WSL 和 Windows 的 SQLite 格式不兼容，复制后会被重建。

**✅ 正确方式：backup → import**

```bash
# 1. WSL 导出
hermes backup --quick -o /mnt/c/Users/liu/Desktop/hermes-backup.zip

# 2. Windows 导入
powershell.exe -Command "hermes import --force C:\Users\liu\Desktop\hermes-backup.zip"
```

## 从 WSL 操控 Windows 的 Pitfalls

### $env 变量被 bash 吃掉
```bash
# ❌ bash 会吃掉 $env:USERPROFILE
powershell.exe -Command "echo $env:USERPROFILE"

# ✅ 写 .ps1 文件再执行（避免变量转义问题）
powershell.exe -ExecutionPolicy Bypass -File "C:\path\to\script.ps1"
```

### 编码乱码
从 WSL 调 powershell.exe -Command，中文输出会乱码（GBK）。使用 `-File` 方式调用 .ps1 脚本更可靠。
