---
name: wechat-bot
description: 微信自动回复机器人 — 基于 pyautogui + Tesseract OCR + Hermes Gateway 的方案。启动、停止、配置监控名单、故障排查。
trigger: 启动|连接|运行.*微信.*机器人|打开.*微信.*自动|wechat.*bot|微信.*AI.*回复
---
# 微信自动回复机器人

基于 pyautogui + Tesseract OCR + Hermes Gateway 的微信自动回复方案。

## 架构

```
微信窗口(必须可见) → 全屏截图(每10秒) → Tesseract OCR(中文+英文)
→ 识别聊天对象 → 匹配监控名单 → Hermes API判断 → 自动回复
```

## 启动前检查

1. **微信已登录且窗口可见** — 不可最小化（Electron 销毁渲染窗口，OCR 读不到）
2. **Hermes Gateway 运行中** — 默认 `http://127.0.0.1:8648`
3. **依赖就绪**：Python 3.11+, Tesseract OCR + 中文包, pyautogui, pytesseract, pyperclip

检查命令（execute_code，因为 terminal 在 WSL 关闭时会挂）：

```python
import subprocess, os, urllib.request, json

# Tesseract
print(os.path.exists(r"C:\Program Files\Tesseract-OCR\tesseract.exe"))

# Gateway
urllib.request.urlopen("http://127.0.0.1:8648/v1/models", timeout=5)

# Python deps
for mod in ['pyautogui','pytesseract','pyperclip']:
    __import__(mod)
```

## 启动

```python
import subprocess
subprocess.Popen(
    [r"C:\Users\liu\AppData\Local\Programs\Python\Python311\python.exe",
     r"C:\Users\liu\Desktop\Hermes工具\微信机器人\wechat_bot.py"],
    creationflags=subprocess.CREATE_NO_WINDOW | 0x00000008
)
```

或手动双击：`C:\Users\liu\Desktop\Hermes工具\微信机器人\启动机器人.bat`

## 配置

核心文件：`wechat_bot.py`，第 13 行 `WATCH_LIST` 控制监控哪些联系人。

```python
WATCH_LIST = ["文件传输助手", "a小小号"]  # 第13行
INTERVAL = 10                              # 第12行，扫描间隔(秒)
HERMES_URL = "http://127.0.0.1:8648/v1/chat/completions"  # 第11行
```

## 工作原理

- `ocr_screen()` → 全屏截图 + Tesseract OCR（chi_sim+eng）
- `find_wechat_region()` → 通过左侧聊天列表特征（时间戳、联系人名）判断是否在微信窗口
- `find_current_chat()` → 识别当前聊天对象
- `check_unread_badge()` → 检测未读数字红点
- `switch_to_chat()` → Ctrl+F 搜索 + Enter 切换聊天
- `type_and_send()` → 剪贴板粘贴 + PowerShell SendKeys 发送（绕过 UIPI）
- `ask_hermes()` → 发给 Hermes Gateway 判断是否回复

## 停止

```powershell
taskkill /f /im python.exe   # 注意：会关掉所有 Python 进程
```

## 已废弃方案

WeChatFerry (wcf_bot.py) — 微信 3.9.x 强制更新后封杀，不可用。
PaddleOCR / easyocr — 均崩溃，不可用。

## 文件位置

```
C:\Users\liu\Desktop\Hermes工具\微信机器人\
├── wechat_bot.py      ← 主程序（OCR方案，当前使用）
├── wcf_bot.py         ← WeChatFerry方案（已废弃）
├── 启动机器人.bat      ← 双击启动
├── switch_wechat.ps1  ← 切换聊天辅助
└── clean_screenshots.ps1 ← 清理截图缓存
```
