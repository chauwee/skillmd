---
name: aihotdaily
description: AI HOT 每日日报生成 Skill。当用户想生成/查看 AI 日报时使用此 Skill，会调用 aihot.virxact.com 的日报 API 拉取数据并整理为中文 markdown 简报。
---

# AI HOT 每日日报

基于 aihot.virxact.com 的日报 API，拉取每日 AI HOT 日报并生成中文 markdown 简报。无需 token，匿名访问。

线上：https://aihot.virxact.com（公开匿名可访）

## 先决条件：必须带 User-Agent

`/api/public/*` 走 nginx UA 黑名单挡商业爬虫，默认 `curl/X.Y` UA 会被 403 Forbidden。**调 API 时所有 curl 都必须带浏览器 UA**：

```bash
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"

# 之后所有调 API 的 curl 都加 -H "User-Agent: $UA"
curl -sH "User-Agent: $UA" "https://aihot.virxact.com/api/public/daily"
```

后面 curl 例子为了简洁默认你已经设了 `$UA`——实际调用必须加 `-H "User-Agent: $UA"`，不要忘。

## 端点速览

| 端点 | 用途 | 参数 |
|---|---|---|
| `/api/public/daily` | 最新日报 | 无 |
| `/api/public/daily/{YYYY-MM-DD}` | 指定日期日报 | path: `date` |
| `/api/public/dailies` | 日报归档列表 | `take` (1-180, default 30) |

约定：
- Base URL: `https://aihot.virxact.com`
- 鉴权：无（匿名）
- 限流：600 req/min/IP（请串行调用，不要并发猛拉）
- 日报每天北京时间 08:00 自动生成，按 UTC 0 点为基准切日

## 工作流

### 拉最新日报

日报是 AI HOT 的"标题层"——每天北京时间 08:00 自动生成，按主题分版块（5 个固定版块）。已有"主编点评"导语段落，是按主题打包后的成品。

```bash
# 拉今日（或最新可用的）日报
curl -sH "User-Agent: $UA" "https://aihot.virxact.com/api/public/daily" \
  | jq '{date, lead: .lead.title, sections: [.sections[] | {label, n: (.items | length)}]}'
```

### 拉指定日期日报

```bash
# YYYY-MM-DD，UTC 0 点为基准
curl -sH "User-Agent: $UA" "https://aihot.virxact.com/api/public/daily/2026-05-07"
```

### 列日报归档

不知道有哪些日期可查时，先看归档：

```bash
# 最近 N 天日报索引（不含正文，只有日期 + 头条标题）
curl -sH "User-Agent: $UA" "https://aihot.virxact.com/api/public/dailies?take=14" \
  | jq '.items[] | {date, leadTitle}'
```

## 返回数据形态

### `/api/public/daily` 返回

```json
{
  "date": "2026-05-07",
  "generatedAt": "2026-05-07T00:01:23.456Z",
  "windowStart": "2026-05-06T00:00:00.000Z",
  "windowEnd":   "2026-05-07T00:00:00.000Z",
  "lead": { "title": "...", "leadParagraph": "..." },
  "sections": [
    {
      "label": "模型发布/更新",
      "items": [
        {
          "title": "...",
          "summary": "...",
          "sourceUrl": "https://...",
          "sourceName": "OpenAI Blog"
        }
      ]
    }
  ],
  "flashes": [
    { "title": "...", "sourceName": "...", "sourceUrl": "...", "publishedAt": "..." }
  ]
}
```

`sections[].label` 固定 5 个："模型发布/更新" / "产品发布/更新" / "行业动态" / "论文研究" / "技巧与观点"。`lead` 极少数日报为 `null`。

### `/api/public/dailies` 返回

```json
{
  "count": 14,
  "items": [
    { "date": "2026-05-07", "generatedAt": "...", "leadTitle": "..." }
  ]
}
```

## 给用户的输出格式

> 核心原则：这一节是**直接展示给用户的最终内容**——必须 markdown 格式 + 排版好 + 普通人能看得懂的人话。用户多数是非技术 AI 创业者 / 设计师 / 普通读者，看到的应该是中文资讯简报，不是 API 调试日志。
>
> 所有"端点路径 / raw 参数 / 限流 / 缓存"等基础设施细节都不能出现在用户看到的输出里。人话级元数据（时间窗 / 条数 / "按发布时间倒序"）可以保留。

### 日报式输出

```markdown
**AI HOT 日报 · 2026-05-07**

> <lead.leadParagraph>（如果有 lead）

## 模型发布/更新
1. **<title>** — <sourceName>
   <summary 简化版 50 字内>
   <sourceUrl>

## 产品发布/更新
2. ...

## 行业动态
3. ...

## 论文研究
4. ...

## 技巧与观点
5. ...

## 快讯（如果 flashes 有内容）
- <flash.title> — <flash.sourceName>（<flash.publishedAt 转人话>）
```

**编号贯穿全文**（1, 2, 3 ... N），不在每个 `##` 内重新计数——这样用户能一眼数到"今天 N 条"。

### 时间转人话

`publishedAt` 是 ISO 8601 UTC，展示时**必须**转成北京时间 + 用户能扫读的相对/绝对时间：

| 内部值 | 展示给用户 |
|---|---|
| `2026-05-08T01:48:00.000Z` | "今天上午 09:48" / "2 小时前" |
| `2026-05-07T18:08:17.000Z` | "今天凌晨 02:08" / "10 小时前" |
| `2026-05-06T16:43:00.000Z` | "5/7 00:43" / "昨天" |

不要直接展示 `2026-05-07T15:30:00.000Z` 这种 ISO 字符串——用户看不懂。

### title vs title_en

日报 items 使用 `title`（中文 normalize 过的）。`title_en` 只在用户明确要求英文版或 `title` 为空时才用。不要两个都展示。

### 副标题／元信息只写人话

**OK**（用户能直接懂的）：
- "今天 5/8 日报北京时间 08:00 后才生成，先看 5/7 这期"
- "按主题分版块"
- "共 N 条"
- "数据来自 aihot.virxact.com"

**不 OK**（基础设施泄漏，坚决不写）：
- 端点路径 / raw 参数名
- "限流 600 req/min" / "nginx 缓存"
- HTTP 状态码 / cache 状态
- 任何后端机制描述

数据源最多写一句："数据来自 aihot.virxact.com"，要么干脆不提。

## 错误处理

- `{"error":"No daily report available yet."}`（HTTP 404）：当天日报还没生成（北京时间 08:00 之前）。建议给用户：拉昨天日报 `GET /api/public/daily/{昨天日期}`
- `{"error":"Invalid date format..."}`（HTTP 400）：date 必须是 `YYYY-MM-DD`，UTC 基准
- HTTP 429（限流）：单 IP 超 600 req/min。串行调用 + 间隔 200ms 即可

## 不要做

- 不要试图猜测 / 编造内容 — 永远以 API 返回为准
- 不要把摘要当原文引用 — 摘要由 LLM 生成，引用需要回 `sourceUrl` 核对
- 不要做高频轮询 — 日报每天 08:00 才更新一次
- 不要在用户输出里暴露端点路径 / raw 参数 / 限流 / 缓存等基础设施细节
- 不要丢掉每条 item 的 sourceUrl — 用户需要追溯到原文
