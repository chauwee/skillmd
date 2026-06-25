# AP News 每日新闻简报生成器

## 1. 概述

从 [AP News](https://apnews.com/) 首页抓取当日头条新闻，生成结构化中文 Markdown 简报，并自动生成语音播报 MP3。

**工作文件夹**: 使用当前 AI agent 的工作目录，或由其他工具告知的交付文件夹，确保将最终产物提交给用户。

**最终产物**:
新闻简报（中英双语、含原文链接）: `APnews_YYYYMMDD_HHMM.md`
语音播报 : `APnews_YYYYMMDD_HHMM.mp3`

**文件名格式**: `YYYYMMDD` = 当前日期（GMT+8），`HHMM` = 当前时间（GMT+8）
示例: `APnews_20260625_2250.md`、`APnews_20260625_2250.mp3`

---

## 2. 环境准备

所有操作前，统一检查并安装 Python 依赖：

```bash
pip3 install beautifulsoup4 edge-tts --quiet
```

如已安装则自动跳过。若 `pip3` 不可用，尝试 `pip` 或 `python -m pip`。确认 `curl` 命令可用（通常系统自带）。

---

## 3. 新闻抓取与简报生成

### 3.1 首页抓取（curl + BeautifulSoup）

**步骤 A：curl 下载首页 HTML** 参考代码：

```bash
curl -sL --max-time 30 \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36" \
  "https://apnews.com/" \
  -o apnews_home.html
```

**步骤 B：BeautifulSoup 解析提取新闻列表** 参考代码：

```python
from bs4 import BeautifulSoup

with open('apnews_home.html', 'r', encoding='utf-8') as f:
    html = f.read()

soup = BeautifulSoup(html, 'html.parser')

article_links = []
for a in soup.find_all('a', href=True):
    href = a['href']
    if '/article/' in href and not href.startswith('#'):
        if href.startswith('/'):
            href = 'https://apnews.com' + href
        title = a.get_text(strip=True)
        if len(title) < 10:
            h = a.find(['h1', 'h2', 'h3', 'h4'])
            if h:
                title = h.get_text(strip=True)
        if title and len(title) > 10:
            article_links.append({'title': title, 'url': href})

# 去重，按出现顺序保留前 6-8 条
seen = set()
unique_articles = []
for a in article_links:
    if a['url'] not in seen:
        seen.add(a['url'])
        unique_articles.append(a)

articles = unique_articles[:8]
```

**提取规则**：
- 排名第 1 条为头条（最显著位置）
- 后续 5-7 条为重要新闻
- 每条包含：英文原标题 + 文章链接

---

### 3.2 头条详情获取（AMP 方案）

**仅对排名第一的头条**进入详情页获取深度内容。

将头条链接转换为 AMP URL（追加 `?outputType=amp`），使用 curl + BeautifulSoup 解析：参考代码：

```python
import subprocess
from bs4 import BeautifulSoup

amp_url = f"{TOP_ARTICLE_URL}?outputType=amp"
html = subprocess.check_output(
    ["curl", "-sL", "--max-time", "30",
     "-H", "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/125.0.0.0 Safari/537.36",
     amp_url]
).decode("utf-8")

soup = BeautifulSoup(html, 'html.parser')
article_body = soup.find('div', class_='RichTextStoryBody')

# 清理干扰节点
for tag in article_body.find_all(['script', 'style', 'template', 'noscript']):
    tag.decompose()
for node in article_body.select('#ap-readmore-embed, .ap-readmore-fade, .FreeStar'):
    node.decompose()

# 提取正文段落
paragraphs = []
for tag in article_body.find_all(['p', 'h2', 'h3', 'h4']):
    text = tag.get_text().strip()
    if len(text) > 30:
        paragraphs.append(text)

full_text = '\n\n'.join(paragraphs)
```

**提取内容**：完整标题（`<title>`）、作者（`article:author` meta）、发布日期（`article:published_time`）、文章全文。

**Fallback**：若 AMP 方案失败，使用 WebFetch 获取头条部分信息，并在简报末尾注明。

---

### 3.3 编译产出（.md 简报 + .txt 录音稿）

使用同一组新闻数据，生成两份文件：

| 要素 | .md 简报 | .txt 录音稿 |
|------|----------|:----------:|
| 中文标题 | ✅ | ✅ |
| 英文原标题 | ✅（blockquote） | — |
| 作者 / 时间戳 | ✅ | — |
| 原文链接 | ✅ | — |
| 正文摘要 | ✅ | ✅ |
| Markdown 标记 | ✅ | —（纯文本）|
| 编号格式 | `### N.` | `N、` |

**.md 简报模板**：

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

**.txt 录音稿格式**：

```text
AP News 新闻播报
简报时间为XXXX年XX月XX日XX时XX分

1、[中文标题]
[正文内容]

2、[中文标题]
[正文内容]

...
```

**录音稿规则**：
- 编号使用 `N、`（中文顿号），**禁止 `N.`**——edge-tts 对 `1.` 会连读（如"一委内瑞拉"），`1、` 则产生清晰停顿
- 不使用"接下来""欢迎收听"等连接词；开头仅两行标题和时间
- 头条完整 300-500 字，其他新闻 1-2 句；不出现任何链接

---

## 4. 语音播报生成

使用 edge-tts 直接生成 MP3，**不使用 ffmpeg**。

```bash
edge-tts --voice zh-CN-YunyangNeural --rate "+33%" \
  -f 录音稿.txt \
  --write-media APnews_YYYYMMDD_HHMM.mp3
```

| 参数 | 值 | 说明 |
|------|-----|------|
| 语音 | `zh-CN-YunyangNeural` | 男声新闻播报风格 |
| 语速 | `+33%` | 加速 33%，适合新闻播报节奏 |

生成 MP3 后删除 `.txt` 录音稿及 `apnews_home.html` 临时文件，最终仅保留 `.md` 和 `.mp3` 两个产物。

---

## 5. 写作规范与行为准则

### 写作规范

1. **中英对照**：每条新闻必须同时提供中文翻译和英文原文（.md 简报中）
2. **客观中立**：不添加主观评论、价值判断或情感化表达；不用"我们""令人震惊"等词
3. **术语处理**：关键术语首次出现时注明英文原文（如 Strait of Hormuz / 霍尔木兹海峡）
4. **摘要深度**：头条 300-500 字综合分析（事件核心、各方立场、背景、影响）；其他新闻 1-2 句

### 行为准则

**✅ 应当做的**：

1. 分步执行：先抓首页列表，再取头条详情
2. 进入头条新闻详情页获取深度内容
3. 所有新闻附带原文链接（.md 简报中）
4. 回复中仅报告执行结果，不重复简报内容
5. 最终仅保留 .md 和 .mp3 两个文件，清理所有中间产物。根据系统提示或其他工具要求，确保产物文件提交。

**❌ 不应做的**：

1. 不遗漏英文原标题（.md 简报中）
2. 不添加主观评论或篡改原文原意
3. 不保留中间产物（.txt 录音稿、临时 HTML 等）
4. 不跳过详情页直接用首页摘要写头条
5. 不编造链接——每条链接必须来自实际抓取

---

*此 skill 供 AI agent 阅读执行，无需安装。*
