---
name: news-briefing
description: 生成每日新闻简报并发布到 GitHub Pages
trigger_keywords:
  - 生成新闻简报
  - 今日新闻
  - 新闻简报
  - 每日新闻
  - news briefing
  - 帮我整理新闻
platform: CoPaw
---

# 📰 News Briefing

生成每日新闻简报并自动发布到 GitHub Pages。

**设计原则**：Agent 使用自身 LLM 能力处理翻译、聚合、分类，无需外部 API。

---

## 工作流程

```
定时任务触发 (每天 06:00)
    ↓
Step 1: Agent 调用采集脚本
    ↓
Step 2: Agent 翻译 + 聚合 + 分类
    ↓
Step 3: Agent 调用生成脚本
    ↓
Step 4: ⚠️ 反思检查（必须执行）
    ↓
Step 5: Agent 发布到 GitHub
```

---

## Step 1: 采集 Top10 新闻

```bash
python3 scripts/collect_top10.py
```

**输入**: `media_status.json`（媒体可访问性列表）
**输出**: `data/news_raw.json`（每媒体 Top10 条新闻）

**采集内容**：
- 新闻标题（具体文章标题，非首页）
- 新闻详情页 URL（完整链接）
- 摘要

---

## Step 2: Agent 处理（核心 - 必须执行翻译）

### ⚠️ 翻译是必选项，不是可选项！

Agent 读取 `data/news_raw.json`，使用自身 LLM 能力完成以下工作：

### 2.1 翻译标题（必须执行）
**要求**：
- 将英文标题翻译为**地道的中文**
- 长度控制在 **10-20 字**
- 保留原意，去除冗余
- 使用中文标点
- 常见词翻译：
  - war → 战争
  - killed/died → 死亡/遇难
  - attack/strike → 袭击
  - deal/agreement → 协议
  - deadline → 最后期限
  - warns/threatens → 警告/威胁
  - crisis/conflict → 危机/冲突

**❌ 错误示例**：
- "Us-以色列 战争 on 伊朗"（中英混杂）
- "Trump threatens Iran in expletive-laden post"（未翻译）
- "伊朗 战争: 特朗普 sees 'good chance'"（中英混杂）

**✅ 正确示例**：
- "特朗普警告伊朗：霍尔木兹海峡最后期限前必须开放"
- "美军从伊朗成功营救失踪飞行员"
- "教皇发表复活节致辞，呼吁全球和平"

### 2.2 聚合同事件
判断哪些新闻是同一事件，合并多来源报道

### 2.3 分类整理
- `international`: 国际政治、战争、外交
- `economy`: 经济、金融、商业
- `tech`: 科技、AI、科学
- `other`: 其他

### 2.4 标记热点
选出最重要的 3-5 条作为热点新闻

### 2.5 ⚠️ 重要：必须保留新闻 URL！
每条新闻的 `sources` 数组必须包含：
- `name`: 媒体名称
- `url`: 新闻详情页完整 URL（如 `https://bbc.com/articles/xxx`）
- `title_en`: 英文标题（原始标题）

**输出**: `data/news_categorized.json`

```json
{
  "date": "2026-03-29",
  "hot_news": [
    {
      "title_cn": "胡塞武装对以色列发动导弹袭击",
      "sources": [
        {"name": "BBC", "url": "https://...", "title_en": "..."},
        {"name": "France 24", "url": "https://...", "title_en": "..."}
      ]
    }
  ],
  "international": [...],
  "economy": [...],
  "tech": [...],
  "other": [...],
  "source_stats": {"BBC": 10, "Al Jazeera": 5}
}
```

---

## Step 3: 生成简报

```bash
python3 scripts/generate_briefing.py
```

**输入**: `data/news_categorized.json`
**输出**: `memory/news-briefing-YYYY-MM-DD.md`

---

## ⚠️ Step 4: 反思检查（Push 前必须执行）

**在发布到 GitHub 之前，必须执行以下自检**：

### 检查清单

```bash
# 1. 检查文件是否存在
ls -la memory/news-briefing-YYYY-MM-DD.md

# 2. 检查 front matter
head -5 memory/news-briefing-YYYY-MM-DD.md
# 必须包含: layout: post
# ❌ 如果是 layout: default → 必须修复
```

### 🔄 翻译检查（最重要）

```bash
# 3. 检查是否有英文标题残留（不应该有）
grep -E '^\[.*\]\(https?://' memory/news-briefing-YYYY-MM-DD.md | head -10
# ❌ 如果有 → 格式错误，标题应该是纯中文

# 4. 检查是否中英混杂
grep -E '[a-zA-Z]{3,}.*\[\[🔗链接\]\]' memory/news-briefing-YYYY-MM-DD.md | head -10
# ❌ 如果有 → 标题未完全翻译

# 5. 检查英文关键词残留
grep -E '(战争|冲突|危机|袭击|死亡|警告|威胁).*\[\[🔗链接\]\]' memory/news-briefing-YYYY-MM-DD.md | grep -iE '\b(is|are|was|were|to|for|in|on|and|the|a|us|trump|iran|israel)\b'
# ❌ 如果有 → 标题中英混杂
```

### ✅ 格式检查

```bash
# 6. 检查链接格式
grep -c '\[\[🔗链接\]\]' memory/news-briefing-YYYY-MM-DD.md
# 必须 > 0

# 7. 检查新闻数量
grep -c '^\w.*\[\[🔗链接\]\]' memory/news-briefing-YYYY-MM-DD.md

# 8. 预览前30行
head -30 memory/news-briefing-YYYY-MM-DD.md
```

### 🚨 修复指南

| 问题 | 修复方法 |
|------|----------|
| 中英混杂 | 重新翻译该标题为纯中文 |
| 英文残留 | 检查 `news_categorized.json` 的 `title_cn` 字段，重新翻译 |
| `layout: default` | 编辑 md 文件，改为 `layout: post` |
| 无 `[[🔗链接]]` | 检查 `generate_briefing.py` 是否正确 |
| 标题过长 | 精简到 10-20 字 |

### 正确格式示例

```markdown
## 🔥 今日要闻

特朗普警告伊朗：霍尔木兹海峡最后期限前必须开放 [[🔗链接]](https://bbc.com/article)
教皇呼吁全球和平：领导人在复活节致辞中敦促选择和平 [[🔗链接]](https://france24.com/article)
美军特种部队深入伊朗成功营救失踪飞行员 [[🔗链接]](https://aljazeera.com/article)
```

### ❌ 错误格式示例

```markdown
# ❌ 错误：中英混杂
Trump threatens 伊朗 in expletive-laden post [[🔗链接]](...)
Us-以色列 战争 on 伊朗 [[🔗链接]](...)

# ❌ 错误：纯英文
Trump warns Iran to reopen Strait of Hormuz [[🔗链接]](...)

# ❌ 错误：标题过长
意大利海岸警卫队在爱琴海救起 500 名移民后发现其中包括大量妇女和儿童以及一些老年人 [[🔗链接]](...)
```

---

## Step 5: 发布

**⚠️ 警告：只有通过 Step 4 所有检查后，才能执行发布！**

Agent 发布到 GitHub Pages（两种方式）：

### 方式 A: 调用脚本
```bash
python3 scripts/publish_to_github.py --file memory/news-briefing-YYYY-MM-DD.md
```

### 方式 B: GitHub API
```bash
# 先 pull
cd ~/news-briefing && git pull

# 复制文件（⚠️ 文件名格式：日期在前）
cp memory/news-briefing-YYYY-MM-DD.md _posts/YYYY-MM-DD-news-briefing.md

# 推送
git add -A && git commit -m "📰 新闻简报 YYYY-MM-DD" && git push
```

### ⚠️ ⚠️ 文件名格式（必须遵守！）
**❌ 错误**：`news-briefing-2026-04-06.md`
**✅ 正确**：`2026-04-06-news-briefing.md`

Jekyll 的 permalink 规则是去掉日期前缀：
- `2026-04-06-news-briefing.md` → URL: `/2026/04/06/news-briefing.html`
- `news-briefing-2026-04-06.md` → URL: `/news-briefing-2026-04-06.html` ❌

---

## 一键执行

```bash
python3 scripts/run_full_pipeline.py
```

> **注意**：此脚本执行 Step 1-3 和 Step 5，但 **Step 2（翻译）和 Step 4（反思检查）需 Agent 手动执行**。

