---
name: novel-writing
description: "Write and publish short-form web novels — chapter writing, style consistency, and platform publishing (七猫/番茄/起点)."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [novel, writing, publishing, qimao, creative]
    category: creative
---

# Novel Writing & Publishing

Write short-form web novels with consistent style and publish to Chinese platforms (七猫, 番茄小说, 起点).

## When to Use

- User asks to write a novel, short story, or web fiction chapter
- User wants to publish to 七猫/番茄/起点
- User mentions "写小说", "短篇", "续写", "发布", "投稿"
- User wants brainstorming, outlines, or chapter planning

## Chapter Conventions

**Save path:** `桌面\短篇小说\第X章_章节标题.md`

**Chapter naming:** Use Chinese numerals (第一章, 第二章, 第三章...). Title after underscore should be a 4-8 character hook. Example: `第一章_冷宫打卡第一天.md`

**Chapter length:** ~1500-2500 characters (中文). Web novel readers on mobile prefer short, punchy chapters with cliffhangers.

## Style Guide (Default: Ancient Politics + Modern Humor)

Default voice (user's preference demonstrated in 《冷宫公务员》):

| Element | Rule |
|---------|------|
| **Genre blend** | Ancient setting (宫廷/朝堂) + modern sensibility (吐槽/方法论) |
| **Protagonist** | Female lead, competent but not OP, uses brains over brawn |
| **Humor** | Light, dry, observational — never slapstick |
| **Pacing** | Fast — hook in first 3 sentences, cliffhanger at chapter end |
| **Dialogue** | Short, charged, subtext-heavy. No exposition dumps. |
| **Internal monologue** | Modern voice ("考公杀穿", "政治学硕士") contrasting with ancient setting |
| **Chapter structure** | Problem → Attempt → Complication → Partial resolution → New tension |

**Override:** If the user specifies a different style (e.g., 纯古言, 玄幻, 现代言情), follow their lead. This guide is the default, not a constraint.

## Quality Checklist (per chapter)

- [ ] Hook in first 3 sentences
- [ ] At least one quotable line
- [ ] Protagonist takes action (not passive)
- [ ] Ending creates tension (cliffhanger or question)
- [ ] No info-dump paragraphs
- [ ] Consistent character voices
- [ ] Cross-chapter continuity (timeline, setting, subplots)

## Chapter File Management

**Write chapters via `write_file` to `C:\Users\liu\Desktop\短篇小说\`** (Windows path — execute_code runs in Windows sandbox). Use `execute_code` for file writes, not the `write_file` tool (which may fail with Windows paths).

```python
# Example: write chapter via execute_code
path = r"C:\Users\liu\Desktop\短篇小说\第一章_标题.md"
with open(path, "w", encoding="utf-8") as f:
    f.write(content)
```

Create the `短篇小说` directory if it doesn't exist (use `os.makedirs`).

**After writing each chapter:** Report word count and core plot point. Ask the user: "继续写还是先发?" Don't auto-write the next chapter without checking in.

## Publishing to 七猫 (Qimao)

**Platform:** [七猫免费小说](https://www.qimao.com) — 女频 strongest (古言/现言), 男频 also growing.

### Preparation

| Item | Rule |
|------|------|
| **书名** | 2-5 characters, hooky. Test: would you click it on a crowded shelf? |
| **分类** | 女频 → 古代言情 → 宫斗宅斗 (default for ancient politics) |
| **标签** | 3-6 tags: 穿越, 权谋, 甜宠, 爽文, 女强, 系统 (pick relevant ones) |
| **简介** | ~200 chars. Format: hook line + premise + unique selling point + closing punch |
| **笔名** | User provides. Ask if not known. |

### Registration Flow (User Does This)

1. Download 七猫免费小说 App → 我的 → 成为作者
2. Register with phone number → verification code → real-name auth
3. Set pen name (笔名)
4. Open browser: `https://author.qimao.com` → login

### Chapter Publishing (User Does This)

After each chapter is written:
1. Tell the user to go to 作者后台 → 作品管理 → 新建章节
2. User copies from `桌面\短篇小说\第X章_xxx.md` and pastes into the editor
3. User clicks 发布

**The agent cannot publish directly** — 七猫 has no public API. All publishing is manual by the user.

## Pitfalls

1. **Don't overwrite existing chapters.** Check if file exists before writing. If it does, confirm with user.
2. **Don't forget continuity.** Before writing chapter N, quickly review chapters N-2, N-1 for subplots, timeline, and character states.
3. **Chinese encoding.** Write files with `encoding="utf-8"`. VBS/batch files need GBK but `.md` files are UTF-8.
4. **Don't get stuck in a loop.** After 3 chapters of the same scene, advance the plot. Web novel readers have zero patience for filler.
5. **Platform≠platform.** 七猫 is NOT the same as 番茄小说 or 起点. Different audiences, different algorithms. Don't mix up publishing instructions.
6. **Don't publish without user confirmation.** Always ask before telling the user to publish.
7. **Chapter duplication in file.** When generating long chapters programmatically, make sure the content doesn't inadvertently include a copy of an earlier chapter appended at the end. Check the final file for duplicate separators (`---`).
