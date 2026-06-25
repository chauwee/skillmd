# AP News 每日新闻简报生成器

## 概述

从 [AP News](https://apnews.com/) 首页抓取当日头条新闻，生成结构化的中文 Markdown 简报文件（含头条深度摘要），并自动生成语音播报 MP3。

最终产物：
- `APnews_YYYYMMDD_HHMM.md` — 新闻简报
- `APnews_YYYYMMDD_HHMM.mp3` — 语音播报

---

## 输出文件规范

### 文件名

```
APnews_YYYYMMDD_HHMM.md
APnews_YYYYMMDD_HHMM.mp3
```

- `YYYYMMDD`: 当前日期（如 `20260617`）
- `HHMM`: 当前时间 GMT+8（如 `0828`）
- 示例: `APnews_20260617_0828.md`、`APnews_20260617_0828.mp3`
- 保存路径: `/workspace/`

---

## 一、新闻简报（.md）

### 文件结构

```markdown
# AP News 新闻简报

> **日期**: YYYY年MM月DD日 HH:MM (GMT+8)
> **来源**: [AP News](https://apnews.com/)

---

## 🔥 头条深度报道

### [中文标题]
> *[英文原文标题]*

**作者**: ... | **更新**: ...

🔗 [阅读全文](URL)

#### 📝 报道摘要
[300-500字深度摘要，含关键引述、背景、数据]

---

## 📰 其他要闻

### N. [中文标题]
> *[英文原文标题]*

[1-2句中文摘要]

**更新时间**: ...

🔗 [阅读全文](URL)
```

---

## 二、内容获取方法

### 第一步：抓取首页头条

使用 `WebFetch` 访问 `https://apnews.com/`，提取以下内容（按首页展示的显著性排序）：

1. 排名第 1 的头条（最显著位置）
2. 后续 5-7 条重要新闻（共 6-8 条）

每条需获取：
- 英文原标题
- 1-2 句简要描述
- 完整文章链接 URL

### 第二步：获取头条详情（AMP 方案）

> ⚠️ **重要**：AP News 详情页采用客户端渲染（React/Brightspot），内容通过 JS 动态加载。直接 WebFetch 普通页面仅能获取 og:meta 元数据，正文无法完整提取。此外，AP News 在移动端（≤767px）设有 Read More 折叠机制，但该机制并非付费墙 —— 内容实际存在于 HTML 中，仅被 CSS 隐藏。

**推荐方案：使用 AMP 版本 + HTML 解析**

1. 将头条文章链接转换为 AMP URL：在原 URL 后追加 `?outputType=amp`
   - 示例：`https://apnews.com/article/xxx` → `https://apnews.com/article/xxx?outputType=amp`
2. 使用 `curl` 下载 AMP 页面 HTML（AMP 页面为服务端渲染，内容完整嵌入 HTML）
3. 使用 Python BeautifulSoup 解析 `<div class="RichTextStoryBody">` 容器，提取所有 `<p>`、`<h2>`、`<h3>` 标签文本
4. 移除干扰节点：`#ap-readmore-embed`、`.ap-readmore-fade`、`.FreeStar` 广告容器
5. 如果使用 AMP 方案无法获取，可以 fallback 到 webfetch 方法提取头条新闻的部分信息。并在简报最后注明。

**Python 解析脚本模板：**

```python
import subprocess, sys
from bs4 import BeautifulSoup

amp_url = f"{ARTICLE_URL}?outputType=amp"
html = subprocess.check_output(
    ["curl", "-sL", "--max-time", "30",
     "-H", "User-Agent: Mozilla/5.0 (Linux; Android 13; Pixel 7) AppleWebKit/537.36 Chrome/125.0.0.0 Mobile Safari/537.36",
     amp_url]
).decode("utf-8")

soup = BeautifulSoup(html, 'html.parser')
article_body = soup.find('div', class_='RichTextStoryBody')

# 清理干扰节点
for tag in article_body.find_all(['script', 'style', 'template', 'noscript']):
    tag.decompose()
for node in article_body.select('#ap-readmore-embed, .ap-readmore-fade, .FreeStar'):
    node.decompose()

# 提取段落
paragraphs = []
for tag in article_body.find_all(['p', 'h2', 'h3', 'h4']):
    text = tag.get_text().strip()
    if len(text) > 30:
        paragraphs.append(text)

full_text = '\n\n'.join(paragraphs)
```

**提取内容包含：**
- 完整标题（从 `<title>` 标签）
- 作者（从 og:meta `article:author` 或 JSON-LD）
- 发布日期（从 `article:published_time` meta）
- 文章全文（Read More 之后的内容一并提取）

> 注意：AMP 页面不含 Read More 截断 —— Read More 仅在移动端（≤767px）通过 JS 隐藏后续元素，AMP HTML 中内容完整可见。

### 第三步：编译简报

按照文件模板格式，将所有信息整合为完整的 Markdown 文件，保存为 `APnews_YYYYMMDD_HHMM.md`。

---

## 三、语音播报（.mp3）

### 第四步：生成文字播报稿

从简报内容生成纯文本播报稿，格式如下：

```text
AP News 新闻播报稿
简报时间为XXXX年XX月XX日XX时XX分

1、[中文标题]
[正文内容]

2、[中文标题]
[正文内容]

...
```

**播报稿生成规则：**

1. **舍弃**：作者名、英文原标题、文章时间戳、原文链接、所有 Markdown 标记（`#`、`**`、`>` 等）
2. **编号格式**：使用 `N、`（数字 + 中文顿号），**禁止使用 `N.`（数字 + 英文句点）**。`、` 号可使 TTS 引擎在编号和标题之间产生自然停顿，避免连读
3. **无连接词**：不使用"接下来""下面是""欢迎收听"等任何连接词和语气词
4. **开头**：仅一行标题「AP News 新闻播报稿」，第二行「简报时间为XXXX年XX月XX日XX时XX分」，随后空一行直接开始编号正文
5. **头条**：保留完整 300-500 字深度摘要
6. **其他新闻**：保留 1-2 句简明摘要
7. 播报稿中不出现任何链接、不出现英文

> ⚠️ **编号格式说明**：edge-tts 对 `1.` 的朗读会将数字与后续文字连读（如"一点委内瑞拉"），使用 `1、` 则 TTS 朗读为"一、委内瑞拉……"，在编号与标题之间产生清晰的停顿边界。

### 第五步：生成语音 MP3

**前提**：先安装 edge-tts：
```bash
sudo pip3 install edge-tts --break-system-packages -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**生成语音**：
```bash
edge-tts --voice zh-CN-YunyangNeural --rate "+33%" -f /workspace/APnews_YYYYMMDD_HHMM_broadcast.txt --write-media /workspace/APnews_YYYYMMDD_HHMM_tmp.mp3
```

**压缩码率至 48kbps**（edge-tts 默认码率较高，须用 ffmpeg 转码）：
```bash
ffmpeg -i /workspace/APnews_YYYYMMDD_HHMM_tmp.mp3 -b:a 48k -y /workspace/APnews_YYYYMMDD_HHMM.mp3
```

**清理中间文件**：
```bash
rm /workspace/APnews_YYYYMMDD_HHMM_broadcast.txt /workspace/APnews_YYYYMMDD_HHMM_tmp.mp3
```

> 注意：`.txt` 播报稿和 `_tmp.mp3` 均为中间产物，最终只保留 `.md` 简报和 `.mp3` 语音两个文件。

### TTS 参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| 语音 | `zh-CN-YunyangNeural` | 男声新闻播报风格 |
| 语速 | `+33%` | 加速 33%，适合新闻播报节奏 |
| 输出码率 | 48kbps | 通过 ffmpeg 转码压缩 |

---

## 写作与排版规范

### 标题处理
- **简报中必须同时提供中文翻译和英文原文**
- 中文标题在前（大标题），英文原文以 blockquote 形式紧随其后
- 中文标题应准确传达原意，不添加主观评论

### 文笔要求
- 简洁、客观、新闻语体
- 不使用第一人称（"我""我们"）
- 不使用情感化表达（如"令人震惊""不可思议"）
- 关键术语首次出现时保留英文原文（如 Strait of Hormuz / 霍尔木兹海峡）

### 排版规则
- 使用 GitHub-flavored Markdown
- 层级清晰：`#` → `##` → `###` → `####`
- 每条新闻之间用 `---` 分隔
- 链接用 `🔗 [阅读全文](URL)` 格式
- 使用 blockquote（`>`）标注英文原标题和直接引语
- 头条摘要中使用 **粗体** 突出关键事实和数据
- 时间始终显示为 GMT+8

### 摘要深度
- **简报头条**: 300-500字的综合分析，包含：事件核心、各方立场、直接引语、背景关联、影响分析
- **简报其他新闻**: 1-2 句简明摘要

---

## 行为准则

### ✅ 应当做的
1. **先抓首页再取详情**，分步进行
2. **头条必须进入详情页**获取深度内容，其他新闻可以不进入详情页，根据首页摘要提供信息即可
3. **所有新闻必须附带原文链接**（见于简报 .md 中）
4. **时间戳使用 GMT+8**，精确到分钟
5. 文件保存后立即向用户展示结果
6. 回复中仅报告执行结果（成功/失败），不展示完整内容
7. **播报稿编号必须使用中文顿号 `N、`**，禁止英文句点 `N.`
8. **最终只保留两个文件**：`.md` 简报 + `.mp3` 语音，清理所有中间产物

### ❌ 不应做的
1. **不要跳过头条详情页抓取**——仅用首页摘要是不够的
2. **不要遗漏英文原标题**（简报中）——每条新闻必须中英对照
3. **不要添加主观评论或价值判断**
4. **不要修改 AP News 原意**——翻译应忠实于原文
5. **不要保留中间产物**（.txt 播报稿、_tmp.mp3 等）——最终仅保留 .md 和 .mp3
6. **不要在回复中重复全部简报内容**——用户直接查看文件即可
7. **不要安装此 skill 文件**——仅供其他 agent 阅读理解
8. **播报稿编号不要使用 `N.`**——TTS 会连读，必须使用 `N、`

---

## 完整执行流程

```
1. WebFetch(https://apnews.com/)                              → 获取 6-8 条头条列表
2. curl(头条#1详情URL?outputType=amp) + BeautifulSoup         → 获取完整文章
3. 整合 → 编译 Markdown → Write(APnews_日期_时间.md)           → 新闻简报
4. 从 .md 内容生成播报稿 → Write(APnews_日期_时间_broadcast.txt)
5. edge-tts 生成语音 → ffmpeg 压缩 48kbps → Write(APnews_日期_时间.mp3)
6. 清理中间文件（_broadcast.txt、_tmp.mp3）
7. open_result_view → 展示 .md 和 .mp3 两个文件
8. 回复: "成功执行。简报保存至 [.md路径]，语音播报保存至 [.mp3路径]。"
```

---

*此 skill 供 AI agent 阅读执行，无需安装。*
