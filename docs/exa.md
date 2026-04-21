# Exa 技术画像

## 概览

Exa（前身为 Metaphor Systems）是一家专注于 AI 原生搜索的公司，其核心产品是为大型语言模型（LLM）、RAG 管道和 AI agent 量身定制的搜索 API。与传统关键词搜索引擎不同，Exa 以 neural embedding（神经嵌入）为底层核心，将查询语义映射到向量空间，从而实现基于语义相似度的文档检索，而非单纯的关键词匹配。这使得 Exa 特别适合处理自然语言问题、模糊意图查询以及需要深度语义理解的复杂场景。

Exa 提供了一系列端点，覆盖实时网页搜索（`/search`）、网页内容抓取（`/contents`）、相似页面发现（`/findSimilar`）、带引用的问答（`/answer`）、多步骤深度研究（`/research`）以及定期监控（`/monitors`）。每月前 1,000 次请求免费，适合快速原型验证。

在 AI 集成场景中，Exa 支持与 LLM 工具调用（tool calling）、RAG 检索增强生成、coding agent、chatbot 等无缝对接，并提供结构化输出（`outputSchema`）、流式响应（`stream`）等 AI 友好特性。延迟方面，`/search` 端点可配置为 180ms 至 1s，满足对延迟敏感的 agent 循环需求。

---

## Base URL

```
https://api.exa.ai
```

所有 API 请求均以此为前缀，例如 `https://api.exa.ai/search`。

---

## 鉴权

Exa API 支持两种鉴权方式，二者等效，任选其一即可：

**方式一：`x-api-key` 请求头**

```bash
curl -X POST 'https://api.exa.ai/search' \
  -H 'x-api-key: YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"query": "Latest LLM research"}'
```

**方式二：`Authorization: Bearer` 请求头**

```bash
curl -X POST 'https://api.exa.ai/search' \
  -H 'Authorization: Bearer YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"query": "Latest LLM research"}'
```

在 [Exa Dashboard](https://dashboard.exa.ai/api-keys) 可获取或管理 API Key。

---

## 端点一览

| 端点 | 方法 | 用途 | 计费基准 |
|---|---|---|---|
| `/search` | POST | 实时网页语义搜索，支持 neural/deep/auto 多种搜索模式，可同时返回网页内容 | $7/1k 请求（1–10 条结果）；超出每条 +$1/1k |
| `/contents` | POST | 根据 URL 或 ID 批量抓取网页全文、highlights、摘要 | $1/1k 页（每种内容类型独立计费）|
| `/findSimilar` | POST | 输入一个 URL，查找语义相似的页面 | $7/1k 请求 |
| `/answer` | POST | 基于实时网页检索生成带引用的直接答案，支持流式响应 | $5/1k 请求 |
| `/research` | POST | 多步骤 agent 工作流，适合复杂查询，返回结构化输出与引用 | $12/1k 请求（deep）/ $15/1k 请求（deep-reasoning）|
| `/monitors` | POST/GET | 按指定频率（每日/每周）运行搜索，通过 webhook 推送新事件 | $15/1k 请求 |

---

## 端点详解

### POST /search

#### 用途

`/search` 是 Exa 的核心端点，用于在实时网络中执行语义搜索。支持 neural（纯嵌入模型）、auto（智能混合）、fast（低延迟流线型）、deep-lite / deep / deep-reasoning（深度综合搜索）、instant（极低延迟）等多种搜索类型。请求体中可内嵌 `contents` 子对象，在同一次请求中同时获取网页正文、highlights 或摘要，避免二次请求。

#### 请求参数表

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `query` | string | ✅ | — | 搜索查询字符串，例如 `"Latest developments in LLM capabilities"` |
| `additionalQueries` | string[] | ❌ | — | 供 deep-search 变体使用的额外查询变体，与主查询协同检索以提升覆盖面 |
| `type` | enum | ❌ | `auto` | 搜索类型：`neural`、`fast`、`auto`、`deep-lite`、`deep`、`deep-reasoning`、`instant` |
| `category` | enum | ❌ | — | 聚焦特定数据类别：`company`、`research paper`、`news`、`personal site`、`financial report`、`people`。注意：`company` 和 `people` 类别不支持部分过滤参数（见下方说明）|
| `userLocation` | string | ❌ | — | 用户所在国家的两字母 ISO 代码，例如 `"US"` |
| `numResults` | integer | ❌ | `10` | 返回结果数量，最大 100。超过 10 条时额外计费 |
| `includeDomains` | string[] | ❌ | — | 仅从这些域名返回结果，最多 1200 个，例如 `["arxiv.org", "paperswithcode.com"]` |
| `excludeDomains` | string[] | ❌ | — | 从结果中排除这些域名，最多 1200 个。注意：`company` 和 `people` 类别不支持此参数 |
| `startCrawlDate` | string (ISO 8601) | ❌ | — | 仅返回在此日期之后被 Exa 爬取的链接 |
| `endCrawlDate` | string (ISO 8601) | ❌ | — | 仅返回在此日期之前被 Exa 爬取的链接 |
| `startPublishedDate` | string (ISO 8601) | ❌ | — | 仅返回发布日期在此日期之后的链接。`company` / `people` 类别不支持 |
| `endPublishedDate` | string (ISO 8601) | ❌ | — | 仅返回发布日期在此日期之前的链接。`company` / `people` 类别不支持 |
| `moderation` | boolean | ❌ | `false` | 启用内容安全过滤，过滤不安全内容 |
| `outputSchema` | object | ❌ | — | JSON Schema 对象，指定后响应将包含 `output` 字段，`output.content` 符合该 schema |
| `systemPrompt` | string | ❌ | — | 指导综合输出生成及 deep-search 搜索规划的系统级指令，例如偏好来源、去重约束等 |
| `stream` | boolean | ❌ | `false` | 为 `true` 时以 SSE（Server-Sent Events）流式返回 OpenAI 兼容的 chat completion 块 |
| `context` | boolean / object | ❌ | — | **已废弃**，请改用 `contents.highlights` 或 `contents.text` |
| `contents` | object | ❌ | — | 内嵌内容抓取配置，子参数见下方 contents 子树说明 |

**`contents` 子树参数（嵌套于 `/search` 请求体中）：**

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `contents.text` | boolean / object | ❌ | — | 请求返回页面全文。可传 `true` 或含 `maxCharacters` 等子参数的对象 |
| `contents.highlights` | boolean / object | ❌ | — | 请求返回 AI 优化的页面高亮片段。可含 `maxCharacters`、`numSentences`、`highlightsPerUrl` 等子参数 |
| `contents.summary` | boolean / object | ❌ | — | 请求返回 AI 生成的页面摘要 |
| `contents.livecrawl` | enum | ❌ | — | 实时爬取策略：`always`、`fallback`、`never` |
| `contents.livecrawlTimeout` | integer | ❌ | — | 实时爬取超时时间（毫秒）|
| `contents.filterEmptyResults` | boolean | ❌ | — | 过滤掉内容为空的结果 |
| `contents.subpages` | integer | ❌ | — | 为每条结果额外抓取的子页面数量 |
| `contents.subpageTarget` | string / string[] | ❌ | — | 子页面抓取目标锚点或路径 |
| `contents.extras` | object | ❌ | — | 额外内容配置，例如 `{ "links": 0 }` 返回页面链接 |

#### 响应字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `requestId` | string | 本次请求的唯一标识符，例如 `"b5947044c4b78efa9552a7c89b306d95"` |
| `searchType` | enum | 对于 `auto` 搜索，指示实际使用的搜索类型：`neural`、`deep`、`deep-reasoning` |
| `context` | string | **已废弃**，合并的上下文字符串，请改用 highlights 或 text |
| `output` | object | 综合输出，仅在提供 `outputSchema` 时返回。含 `content`（符合 schema 的结构化内容）和 `grounding`（引用依据）|
| `output.grounding[]` | array | 引用依据列表，每项含 `field`、`citations[].{url, title}`、`confidence`（`low`/`medium`/`high`）|
| `results[]` | array | 搜索结果列表 |
| `results[].id` | string | 结果的规范 URL / 唯一标识符 |
| `results[].url` | string | 结果页面的实际 URL |
| `results[].title` | string | 页面标题 |
| `results[].publishedDate` | string (ISO 8601) | 页面发布日期 |
| `results[].author` | string | 页面作者 |
| `results[].image` | string | 页面预览图 URL |
| `results[].favicon` | string | 网站 favicon URL |
| `results[].text` | string | 页面全文（需请求 `contents.text`）|
| `results[].highlights` | string[] | AI 优化的高亮文本片段数组（需请求 `contents.highlights`）|
| `results[].highlightScores` | number[] | 各 highlight 片段的相关性评分，与 `highlights` 数组一一对应 |
| `results[].summary` | string | AI 生成的页面摘要（需请求 `contents.summary`）|
| `results[].subpages[]` | array | 子页面内容，结构与主结果相同（需请求 `contents.subpages`）|
| `results[].extras.links` | array | 页面中的链接列表（需请求 `contents.extras`）|
| `costDollars.total` | number | 本次请求总费用（美元）|
| `costDollars.breakDown[]` | array | 费用明细，含 `search`、`contents`，以及 `breakdown.{neuralSearch, deepSearch, contentText, contentHighlight, contentSummary}` |
| `costDollars.perRequestPrices` | object | 各搜索类型的每千请求单价参考 |
| `costDollars.perPagePrices` | object | 各内容类型的每千页单价参考 |

#### cURL 示例

```bash
curl -X POST 'https://api.exa.ai/search' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "Latest research in LLMs",
    "type": "auto",
    "numResults": 5,
    "startPublishedDate": "2024-01-01T00:00:00.000Z",
    "contents": {
      "highlights": {
        "maxCharacters": 4000
      }
    }
  }'
```

#### JSON 响应示例

```json
{
  "requestId": "b5947044c4b78efa9552a7c89b306d95",
  "searchType": "neural",
  "results": [
    {
      "title": "A Comprehensive Overview of Large Language Models",
      "url": "https://arxiv.org/pdf/2307.06435.pdf",
      "id": "https://arxiv.org/abs/2307.06435",
      "publishedDate": "2023-11-16T01:36:32.547Z",
      "author": "Humza Naveed, University of Engineering and Technology (UET), Lahore, Pakistan",
      "image": "https://arxiv.org/pdf/2307.06435.pdf/page_1.png",
      "favicon": "https://arxiv.org/favicon.ico",
      "text": "Abstract Large Language Models (LLMs) have recently demonstrated remarkable capabilities...",
      "highlights": [
        "Such requirements have limited their adoption..."
      ],
      "highlightScores": [
        0.4600165784358978
      ],
      "summary": "This overview paper on Large Language Models (LLMs) highlights key developments...",
      "extras": {
        "links": []
      }
    }
  ],
  "costDollars": {
    "total": 0.007,
    "breakDown": [
      {
        "search": 0.007,
        "contents": 0,
        "breakdown": {
          "neuralSearch": 0.007,
          "deepSearch": 0.012,
          "contentText": 0,
          "contentHighlight": 0,
          "contentSummary": 0
        }
      }
    ],
    "perRequestPrices": {
      "neuralSearch_1_10_results": 0.007,
      "neuralSearch_additional_result": 0.001,
      "deepSearch": 0.012,
      "deepReasoningSearch": 0.015
    },
    "perPagePrices": {
      "contentText": 0.001,
      "contentHighlight": 0.001,
      "contentSummary": 0.001
    }
  }
}
```

#### 计费说明

- **基础费用**：neural / auto / fast 搜索，1–10 条结果时 $7/1k 请求。
- **额外结果**：超过 10 条，每额外 1 条 +$1/1k 请求。
- **deep 搜索**：$12/1k 请求；**deep-reasoning 搜索**：$15/1k 请求。
- **内容类型**：`text`、`highlights`、`summary` 各自独立计费，均为 $1/1k 页。
- 同一请求中同时启用多种内容类型，各类型费用叠加。
- 速率限制：默认 10 QPS，需更高限制请联系 [hello@exa.ai](mailto:hello@exa.ai)。

---

### POST /contents

#### 用途

`/contents` 端点用于根据已知的页面 URL 或 Exa 结果 ID 批量抓取网页内容，无需执行搜索。适合以下场景：已从 `/search` 获得 URL 列表、需要对特定页面进行全文抓取或 AI 摘要生成、需要实时抓取最新内容（livecrawl）等。每种内容类型（text、highlights、summary）单独计费，按每 1k 页 $1 计算。速率限制为 100 QPS。

#### 请求参数表

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `ids` | string[] | 与 `urls` 二选一 | — | Exa 结果 ID 列表（通常与 URL 相同，为规范化 URL）|
| `urls` | string[] | 与 `ids` 二选一 | — | 需要抓取内容的页面 URL 列表 |
| `text` | boolean / object | ❌ | — | 请求返回页面全文。可传 `true`，或含 `maxCharacters`（最大字符数）的对象 |
| `text.maxCharacters` | integer | ❌ | — | 全文最大字符数限制 |
| `text.includeHtmlTags` | boolean | ❌ | — | 是否在全文中保留 HTML 标签 |
| `highlights` | boolean / object | ❌ | — | 请求返回 AI 优化的高亮片段 |
| `highlights.numSentences` | integer | ❌ | — | 每个 highlight 片段的句子数 |
| `highlights.highlightsPerUrl` | integer | ❌ | — | 每个 URL 返回的 highlight 片段数量 |
| `highlights.maxCharacters` | integer | ❌ | — | 每个 highlight 片段的最大字符数 |
| `highlights.query` | string | ❌ | — | 用于生成 highlights 的查询语句，不填则使用通用摘要 |
| `summary` | boolean / object | ❌ | — | 请求返回 AI 生成的页面摘要 |
| `summary.query` | string | ❌ | — | 指导摘要生成方向的查询语句 |
| `livecrawl` | enum | ❌ | — | 实时爬取策略：`always`（始终实时爬取）、`fallback`（缓存未命中时实时爬取）、`never`（仅用缓存）|
| `livecrawlTimeout` | integer | ❌ | — | 实时爬取超时时间（毫秒）|
| `filterEmptyResults` | boolean | ❌ | — | 为 `true` 时过滤掉无法成功抓取内容的结果 |
| `subpages` | integer | ❌ | — | 为每个 URL 额外抓取的子页面数量 |
| `subpageTarget` | string / string[] | ❌ | — | 子页面抓取的目标锚点或路径 |
| `extras` | object | ❌ | — | 额外内容配置，例如 `{ "links": 0 }` 请求返回页面中的链接 |

#### 响应结构

| 字段 | 类型 | 说明 |
|---|---|---|
| `results[]` | array | 抓取内容的结果列表，每项结构与 `/search` 的 `results[]` 一致 |
| `results[].id` | string | 页面规范 URL / Exa ID |
| `results[].url` | string | 页面实际 URL |
| `results[].title` | string | 页面标题 |
| `results[].publishedDate` | string (ISO 8601) | 页面发布日期 |
| `results[].author` | string | 页面作者 |
| `results[].image` | string | 页面预览图 URL |
| `results[].favicon` | string | 网站 favicon URL |
| `results[].text` | string | 页面全文（需请求 `text`）|
| `results[].highlights` | string[] | AI 优化的高亮片段数组（需请求 `highlights`）|
| `results[].highlightScores` | number[] | 各 highlight 片段的相关性评分 |
| `results[].summary` | string | AI 生成的页面摘要（需请求 `summary`）|
| `results[].subpages[]` | array | 子页面内容（需请求 `subpages`）|
| `results[].extras` | object | 额外内容，例如 `links` 数组 |
| `costDollars` | object | 费用明细，结构同 `/search` 响应中的 `costDollars` |

#### cURL 示例

```bash
curl -X POST 'https://api.exa.ai/contents' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "urls": [
      "https://arxiv.org/abs/2307.06435",
      "https://arxiv.org/abs/2303.17580"
    ],
    "text": {
      "maxCharacters": 10000
    },
    "highlights": {
      "numSentences": 3,
      "highlightsPerUrl": 2
    },
    "summary": true,
    "livecrawl": "fallback"
  }'
```

#### JSON 响应示例

```json
{
  "results": [
    {
      "id": "https://arxiv.org/abs/2307.06435",
      "url": "https://arxiv.org/pdf/2307.06435.pdf",
      "title": "A Comprehensive Overview of Large Language Models",
      "publishedDate": "2023-11-16T01:36:32.547Z",
      "author": "Humza Naveed, University of Engineering and Technology (UET), Lahore, Pakistan",
      "image": "https://arxiv.org/pdf/2307.06435.pdf/page_1.png",
      "favicon": "https://arxiv.org/favicon.ico",
      "text": "Abstract Large Language Models (LLMs) have recently demonstrated remarkable capabilities in natural language understanding...",
      "highlights": [
        "Such requirements have limited their adoption in resource-constrained environments."
      ],
      "highlightScores": [
        0.4600165784358978
      ],
      "summary": "This overview paper on Large Language Models (LLMs) highlights key developments in training, fine-tuning, and deployment methodologies."
    }
  ],
  "costDollars": {
    "total": 0.003,
    "breakDown": [
      {
        "search": 0,
        "contents": 0.003,
        "breakdown": {
          "neuralSearch": 0,
          "deepSearch": 0,
          "contentText": 0.001,
          "contentHighlight": 0.001,
          "contentSummary": 0.001
        }
      }
    ],
    "perRequestPrices": {
      "neuralSearch_1_10_results": 0.007,
      "neuralSearch_additional_result": 0.001,
      "deepSearch": 0.012,
      "deepReasoningSearch": 0.015
    },
    "perPagePrices": {
      "contentText": 0.001,
      "contentHighlight": 0.001,
      "contentSummary": 0.001
    }
  }
}
```

#### 计费说明

- `/contents` 不收取请求级别的固定费用，仅按内容类型和页面数量计费。
- `text`、`highlights`、`summary` 各自独立计费，均为 **$1/1k 页**（即每页 $0.001）。
- 同时请求三种内容类型时，每页最高费用为 $0.003。
- `livecrawl` 策略不影响计费单价，但会影响内容新鲜度。
- 速率限制：默认 **100 QPS**，为所有端点中最高，适合批量内容处理场景。

---

### POST /findSimilar

`/findSimilar` 端点根据给定 URL 查找互联网上与之内容相似的页面，适用于竞品分析、相关内容推荐、研究扩展等场景。其计费规则与 `/search` 相同，按搜索请求数计算。

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `url` | string | 是 | — | 用于查找相似内容的源页面 URL |
| `numResults` | integer | 否 | 10 | 返回结果数量 |
| `includeDomains` | string[] | 否 | — | 限定结果只来自这些域名 |
| `excludeDomains` | string[] | 否 | — | 排除这些域名的结果 |
| `excludeSourceDomain` | boolean | 否 | false | 若为 `true`，则从结果中排除源 URL 所在域名 |
| `startPublishedDate` | string | 否 | — | 结果发布时间下限（ISO 8601 格式） |
| `endPublishedDate` | string | 否 | — | 结果发布时间上限（ISO 8601 格式） |
| `includeText` | string | 否 | — | 结果正文中必须包含的文本片段 |
| `excludeText` | string | 否 | — | 结果正文中不得包含的文本片段 |
| `contents` | object | 否 | — | 内容获取选项，结构与 `/search` 的 `contents` 参数相同，支持 `text`、`highlights`、`summary` 子参数 |

#### 响应字段

响应结构与 `/search` 一致，包含 `requestId`、`results` 数组（每项含 `id`、`url`、`title`、`publishedDate`、`author`、`score` 等字段）以及 `costDollars` 对象。

#### cURL 示例

```bash
curl -X POST 'https://api.exa.ai/findSimilar' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://amistrongeryet.substack.com/p/are-we-on-the-brink-of-agi",
    "numResults": 3,
    "excludeSourceDomain": true
  }'
```

#### 响应示例

```json
{
  "requestId": "a1b2c3d4e5f6",
  "results": [
    {
      "id": "https://example.com/agi-article",
      "url": "https://example.com/agi-article",
      "title": "The Road to AGI: What We Know So Far",
      "publishedDate": "2024-03-10T00:00:00.000Z",
      "author": "Jane Doe",
      "score": 0.91
    }
  ],
  "costDollars": {
    "total": 0.007
  }
}
```

#### 计费

`/findSimilar` 按与 `/search` 相同的规则计费：前 10 条结果 $0.007/请求，每额外一条结果 $0.001。如同时请求内容（text / highlights / summary），每页额外计费 $0.001。速率限制为 10 QPS。

---

### POST /answer

`/answer` 端点接收一个自然语言问题，自动在互联网上执行搜索，并基于检索到的页面生成带引用的答案。适用于问答机器人、报告生成、事实核查等场景。支持流式返回（SSE）及结构化输出（`outputSchema`）。

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `query` | string | 是 | — | 要回答的问题或查询，最小长度 1 个字符 |
| `text` | boolean | 否 | false | 若为 `true`，在引用结果中附带完整正文 |
| `stream` | boolean | 否 | false | 若为 `true`，以 Server-Sent Events（SSE）流式返回响应 |
| `outputSchema` | object | 否 | — | [JSON Schema Draft 7](https://json-schema.org/draft-07) 规范对象；提供后 `answer` 字段将返回符合该 schema 的结构化对象，而非纯字符串 |

#### 响应字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `answer` | string \| object | 生成的答案。默认为字符串；提供 `outputSchema` 时返回结构化对象 |
| `citations` | object[] | 用于生成答案的搜索结果列表，每项包含 `id`、`url`、`title`、`author`、`publishedDate`、`text`、`image`、`favicon` 等字段 |
| `costDollars` | object | 费用明细，见下方说明 |

**`costDollars` 结构说明：**

`costDollars` 包含以下字段：

- `total`：本次请求总费用（美元）
- `breakDown`：数组，每项包含：
  - `search`：搜索部分费用
  - `contents`：内容获取费用
  - `breakdown`：细分字段，包括：
    - `neuralSearch`：神经搜索费用（$0.007/请求，前 10 条结果）
    - `deepSearch`：深度搜索费用（$0.012/请求）
    - `contentText`：正文内容费用（$0.001/页）
    - `contentHighlight`：高亮内容费用（$0.001/页）
    - `contentSummary`：摘要内容费用（$0.001/页）
- `perRequestPrices`：各搜索类型单价参考
- `perPagePrices`：各内容类型单页价格参考

#### cURL 示例

```bash
curl -X POST 'https://api.exa.ai/answer' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "What is the latest valuation of SpaceX?",
    "text": true
  }'
```

#### 响应示例

```json
{
  "answer": "$350 billion.",
  "citations": [
    {
      "id": "https://www.theguardian.com/science/2024/dec/11/spacex-valued-at-350bn-as-company-agrees-to-buy-shares-from-employees",
      "url": "https://www.theguardian.com/science/2024/dec/11/spacex-valued-at-350bn-as-company-agrees-to-buy-shares-from-employees",
      "title": "SpaceX valued at $350bn as company agrees to buy shares from ...",
      "author": "Dan Milmon",
      "publishedDate": "2023-11-16T01:36:32.547Z",
      "text": "SpaceX valued at $350bn as company agrees to buy shares from ...",
      "image": "https://i.guim.co.uk/img/media/...",
      "favicon": "https://assets.guim.co.uk/static/frontend/icons/homescreen/apple-touch-icon.svg"
    }
  ],
  "costDollars": {
    "total": 0.007,
    "breakDown": [
      {
        "search": 0.007,
        "contents": 0,
        "breakdown": {
          "neuralSearch": 0.007,
          "deepSearch": 0.012,
          "contentText": 0,
          "contentHighlight": 0,
          "contentSummary": 0
        }
      }
    ],
    "perRequestPrices": {
      "neuralSearch_1_10_results": 0.007,
      "neuralSearch_additional_result": 0.001,
      "deepSearch": 0.012,
      "deepReasoningSearch": 0.015
    },
    "perPagePrices": {
      "contentText": 0.001,
      "contentHighlight": 0.001,
      "contentSummary": 0.001
    }
  }
}
```

#### 计费

`/answer` 的费用根据内部实际执行的搜索类型计算，与 `/search` 定价一致。速率限制为 10 QPS。若启用 `text: true`，还会叠加内容获取费用（$0.001/页）。

---

### POST /research

> ⚠️ **重要：该端点已于 2026-05-01 正式下线。** 请迁移至 `/search`（`type: "deep-reasoning"`）。详见下方迁移建议。

`/research` 端点用于执行长时间运行的深度研究任务，支持复杂、多步骤的信息检索与综合。该端点采用异步任务模型：提交后返回任务 ID，通过轮询获取状态与结果。

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `instructions` | string | 是 | — | 研究任务的具体指令或问题描述 |
| `model` | string | 否 | — | 使用的研究模型，可选值：`exa-research-fast`、`exa-research`、`exa-research-pro` |

#### 任务状态流转

研究任务按以下状态顺序流转：

```
submitted → running → completed
```

- **submitted**：任务已接收，排队等待执行
- **running**：任务正在执行中，正在进行搜索与综合
- **completed**：任务已完成，结果可读取

速率限制为最多 15 个并发任务。

#### 响应字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `researchId` | string | 研究任务的唯一标识符 |
| `status` | string | 当前任务状态：`submitted` / `running` / `completed` |
| `model` | string | 执行任务所用的模型名称 |
| `instructions` | string | 提交时的研究指令 |
| `data` / `result` | object | 任务完成后返回的研究结果（资料未明确区分 `data` 与 `result` 的具体结构差异） |

#### cURL 示例

**提交研究任务：**

```bash
curl -X POST 'https://api.exa.ai/research' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "instructions": "Summarize the latest advances in quantum error correction as of 2024.",
    "model": "exa-research"
  }'
```

**轮询任务状态：**

```bash
curl -X GET 'https://api.exa.ai/research/{researchId}' \
  -H 'x-api-key: YOUR-EXA-API-KEY'
```

#### 响应示例

**提交后响应（status: submitted）：**

```json
{
  "researchId": "res_abc123xyz",
  "status": "submitted",
  "model": "exa-research",
  "instructions": "Summarize the latest advances in quantum error correction as of 2024."
}
```

**完成后响应（status: completed）：**

```json
{
  "researchId": "res_abc123xyz",
  "status": "completed",
  "model": "exa-research",
  "instructions": "Summarize the latest advances in quantum error correction as of 2024.",
  "result": {
    "summary": "Recent advances in quantum error correction include..."
  }
}
```

#### 计费

资料未明确说明 `/research` 端点的具体单价。根据 Changelog 信息，该端点已于 2026-05-01 下线，建议迁移至 `/search` 的 `deep-reasoning` 类型（定价 $0.015/请求）。

#### ⚠️ 迁移建议：从 /research 迁移至 /search（type: deep-reasoning）

根据 [Exa API Deprecation Notice（April 2026）](https://exa.ai/docs/changelog/may-2026-api-deprecations)：

- **下线时间节点**：
  - 2026-04-01：发布弃用通知
  - 2026-04-15：已弃用功能返回 null 或被忽略
  - **2026-05-01**：`/research` 端点硬删除，不再可用

- **迁移方式**：将原来调用 `/research` 的代码改为调用 `/search`，并设置 `type: "deep-reasoning"`。

**迁移前（使用 /research）：**

```bash
curl -X POST 'https://api.exa.ai/research' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "instructions": "Latest news on Nvidia",
    "model": "exa-research"
  }'
```

**迁移后（使用 /search + type: deep-reasoning）：**

```bash
curl -X POST 'https://api.exa.ai/search' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "Latest news on Nvidia",
    "type": "deep-reasoning",
    "highlights": {
      "maxCharacters": 4000
    },
    "outputSchema": {
      "description": "Answer in detail with multiple sections and specific details",
      "type": "text"
    }
  }'
```

`deep-reasoning` 搜索类型提供与原 `/research` 端点相近的深度推理与综合能力，并且是同步返回结果，无需轮询状态，开发体验更简洁。

---

### /monitors

`/monitors` 端点用于创建定时重复搜索任务，并通过 webhook 将结果推送到指定 URL。适用于监控特定话题的最新动态、竞品信息追踪、新闻告警等持续性信息获取场景。

#### 用途说明

Monitors 将搜索任务持久化，按设定的频率（cadence）自动执行，每次搜索完成后将结果以 HTTP POST 方式推送到用户提供的 `webhookUrl`，无需手动重复调用 API。

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `query` | string | 是 | — | 需要定期执行的搜索查询语句 |
| `cadence` | string | 是 | — | 执行频率（资料未明确说明具体可选值，如 `daily`、`weekly` 等） |
| `webhookUrl` | string | 是 | — | 每次搜索完成后，结果推送的目标 URL |
| `outputSchema` | object | 否 | — | JSON Schema Draft 7 对象，用于结构化搜索结果（资料未明确说明在 Monitors 中的具体行为） |

> 注意：`/monitors` 端点的完整参数列表资料未明确说明，以上参数基于资料中的有限描述整理。如需完整参数说明，请参考 [Exa 官方文档](https://exa.ai/docs)。

#### cURL 示例

```bash
curl -X POST 'https://api.exa.ai/monitors' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "Latest AI research breakthroughs",
    "cadence": "daily",
    "webhookUrl": "https://your-server.example.com/webhook/exa"
  }'
```

#### 计费

Monitors 的计费价格为 **$15 / 1,000 次请求**，与 `deep-reasoning` 搜索类型定价相同（$0.015/请求）。

---

## 搜索类型详解

`/search` 端点的 `type` 参数支持多种搜索模式，不同模式在延迟、适用场景和价格上存在差异。下表对各类型进行对比说明：

| 搜索类型 | 技术原理 | 典型延迟 | 适用场景 | 价格（前10条结果） |
|---|---|---|---|---|
| `neural` | 基于 embedding 的语义向量搜索，理解查询意图 | 快速 | 概念性、主题性查询；需要理解语义的场景 | $0.007/请求 |
| `fast` | 关键词匹配，速度优先 | 快速 | 精确关键词检索、已知 URL 查询 | 资料未明确说明（与 neural 同档或更低） |
| `auto` | 自动选择 neural / deep / deep-reasoning 中最合适的类型 | 快速 | 通用场景，不确定用哪种类型时推荐使用 | 取决于实际选用类型 |
| `deep-lite` | 轻量级深度搜索，比 deep 更快 | 中等 | 需要一定深度但对速度有要求的场景 | 资料未明确说明 |
| `deep` | 深度搜索，适合复杂查询 | 较慢 | 复杂研究型查询，需要综合多源信息 | $0.012/请求 |
| `deep-reasoning` | 深度推理搜索，最强综合与推理能力 | 最慢（资料未明确具体数值） | 复杂多步骤推理、研究报告生成；替代原 `/research` 端点 | $0.015/请求 |
| `instant` | 极速搜索，延迟 <200ms | <200ms | 对延迟极度敏感的实时场景，如自动补全、实时建议 | 资料未明确说明 |

**补充说明：**

- `auto` 类型会在响应的 `searchType` 字段中返回实际选用的搜索类型（可能为 `neural`、`deep`、`deep-reasoning` 之一）。
- `deep-reasoning` 是 `/research` 端点的替代方案，价格 $0.015/请求，与 Monitors 计费单价相同。
- `instant` 类型延迟低于 200ms，适合对响应时间要求严苛的在线产品场景。
- `deep-lite` 和 `fast` 的具体定价资料未明确说明。

---

## 结构化输出（outputSchema）

### 概述

`outputSchema` 参数允许调用方指定一个 [JSON Schema Draft 7](https://json-schema.org/draft-07) 格式的对象，告知 Exa API 以结构化 JSON 格式返回结果，而非自由文本。

**支持 `outputSchema` 的端点：**

- **`/answer`**：提供 `outputSchema` 后，`answer` 字段将返回符合 schema 的结构化对象，而非默认的字符串。
- **`/search`**：提供 `outputSchema` 后，响应中包含 `output` 对象，包含基于搜索结果合成的结构化输出。
- **`/monitors`**：资料中提及支持 `outputSchema` 参数，具体行为资料未明确说明。

### 工作原理

当请求中携带 `outputSchema` 时：

1. Exa 在执行搜索或回答后，将结果按照用户定义的 JSON Schema 进行结构化处理。
2. 响应中的 `answer`（`/answer` 端点）或 `output`（`/search` 端点）字段将是一个符合 schema 结构的 JSON 对象，而非普通字符串。
3. 这使得下游系统可以直接解析 JSON 结构，无需对自由文本做后处理。

### Python 示例（exa-py）

以下示例展示如何在 `/search` 中使用 `outputSchema` 获取结构化输出：

```python
from exa_py import Exa
from dotenv import load_dotenv
import os

load_dotenv()
exa = Exa(os.getenv('EXA_API_KEY'))

# 定义期望的结构化输出格式（JSON Schema Draft 7）
output_schema = {
    "type": "object",
    "properties": {
        "summary": {
            "type": "string",
            "description": "A concise summary of the latest findings"
        },
        "key_points": {
            "type": "array",
            "items": {
                "type": "string"
            },
            "description": "List of key points from the search results"
        },
        "sources_count": {
            "type": "integer",
            "description": "Number of sources referenced"
        }
    },
    "required": ["summary", "key_points"]
}

result = exa.search(
    "Latest news on Nvidia",
    type="deep-reasoning",
    contents={
        "highlights": {
            "max_characters": 4000
        }
    },
    output_schema=output_schema
)

# output 字段将是符合上述 schema 的结构化对象
print(result.output)
# 示例输出：
# {
#   "summary": "Nvidia continues to dominate the AI chip market...",
#   "key_points": ["H100 demand remains strong", "New Blackwell architecture..."],
#   "sources_count": 8
# }
```

以下示例展示如何在 `/answer` 中使用 `outputSchema`：

```python
from exa_py import Exa
from dotenv import load_dotenv
import os

load_dotenv()
exa = Exa(os.getenv('EXA_API_KEY'))

# 使用 outputSchema 要求结构化的答案格式
output_schema = {
    "type": "object",
    "properties": {
        "valuation": {
            "type": "string",
            "description": "The company valuation as a string"
        },
        "valuation_date": {
            "type": "string",
            "description": "The date of the valuation"
        },
        "context": {
            "type": "string",
            "description": "Brief context around the valuation"
        }
    },
    "required": ["valuation"]
}

result = exa.answer(
    "What is the latest valuation of SpaceX?",
    text=True,
    output_schema=output_schema
)

# answer 字段现在是一个结构化对象而非字符串
print(result.answer)
# 示例输出：
# {
#   "valuation": "$350 billion",
#   "valuation_date": "December 2024",
#   "context": "SpaceX reached this valuation as the company agreed to buy shares from employees."
# }
print(result.citations)  # 引用来源列表照常返回
```

### 注意事项

- `outputSchema` 必须符合 [JSON Schema Draft 7](https://json-schema.org/draft-07) 规范。
- 提供 `outputSchema` 时，`/answer` 端点的 `answer` 字段类型从 `string` 变为 `object`。
- 结构化输出不影响 `citations` 等其他响应字段的返回。
- 在 `/search` 端点中，结构化输出通过响应顶层的 `output` 对象返回，而非 `results` 数组。

---

## 定价总表

| 端点 / 功能 | 定价规则 | 超额规则 |
|---|---|---|
| **Search**（`type=auto` / `neural` / `fast`） | $7 / 1K requests（≤10 条结果） | 每超过 10 条结果 +$1 / 1K requests |
| **Deep Search**（`type=deep-lite` / `deep`） | $12 / 1K requests（≤10 条结果） | 每超过 10 条结果 +$1 / 1K requests |
| **Deep-Reasoning Search**（`type=deep-reasoning`） | $15 / 1K requests（≤10 条结果） | 每超过 10 条结果 +$1 / 1K requests |
| **Contents**（`/contents` 端点） | $1 / 1K pages per content type | 无超额，按页计费 |
| &nbsp;&nbsp;↳ `contentText` | $0.001 / page | — |
| &nbsp;&nbsp;↳ `contentHighlight` | $0.001 / page | — |
| &nbsp;&nbsp;↳ `contentSummary` | $0.001 / page | — |
| **Monitors**（`/monitors`） | $15 / 1K requests（≤10 条结果） | 每超过 10 条结果 +$1 / 1K requests |
| **Answer**（`/answer`） | $5 / 1K requests（≤10 条结果） | 每超过 10 条结果 +$1 / 1K requests |
| **AI page summaries**（所有端点） | $1 / 1K pages | — |

> **注意**：以上价格均以美元计价，1K = 1,000 次请求或页面。

### 特殊计划

- **Startup Grants**：符合条件的初创公司可申请 **$1,000 credits** 免费额度。
- **Education Grants**：高校/教育机构可申请 **$1,000 credits** 免费额度。
- **Enterprise 版本**：提供 volume discounts（批量折扣）、custom rate limits（自定义速率限制）、MSAs（主服务协议）、zero data retention（零数据留存）、1:1 onboarding（一对一入驻支持）。如需了解，请联系 [hello@exa.ai](mailto:hello@exa.ai)。

---

## 速率限制

| 端点 | 速率限制 |
|---|---|
| `/search` | 10 QPS（每秒 10 次请求） |
| `/findSimilar` | 10 QPS |
| `/contents` | 100 QPS |
| `/answer` | 10 QPS |
| `/research` | 15 个并发任务（concurrent tasks） |

> **Enterprise 用户**：速率限制可根据业务需求定制，详情请联系 Exa 销售团队。

---

## SDK 与代码示例

### 安装

**Python**

```bash
pip install exa-py
```

**JavaScript / TypeScript**

```bash
npm install exa-js
```

### 环境变量配置（`.env`）

```dotenv
EXA_API_KEY=your-exa-api-key-here
```

---

### 示例 1：Python 简单搜索并打印结果

```python
import os
from exa_py import Exa

client = Exa(api_key=os.environ["EXA_API_KEY"])

results = client.search(
    "latest research on large language models",
    num_results=5,
    type="neural"
)

for r in results.results:
    print(r.title)
    print(r.url)
    print(r.published_date)
    print("---")
```

---

### 示例 2：Python search with contents（highlights、summary、text）

```python
import os
from exa_py import Exa

client = Exa(api_key=os.environ["EXA_API_KEY"])

results = client.search_and_contents(
    "AI safety research 2024",
    num_results=3,
    type="neural",
    highlights={"num_sentences": 3, "highlights_per_url": 2},
    summary={"query": "What are the key findings?"},
    text=True
)

for r in results.results:
    print("Title:", r.title)
    print("URL:", r.url)
    print("Highlights:", r.highlights)
    print("Summary:", r.summary)
    print("Text (first 300 chars):", r.text[:300] if r.text else "N/A")
    print("===")
```

---

### 示例 3：Python answer 端点（带 `text=True`）

```python
import os
from exa_py import Exa

client = Exa(api_key=os.environ["EXA_API_KEY"])

response = client.answer(
    "What are the main differences between GPT-4 and Claude 3?",
    text=True
)

print("Answer:", response.answer)
print("\nSource citations:")
for source in response.sources:
    print(" -", source.title, source.url)
```

---

### 示例 4：JavaScript / TypeScript — search + contents

```typescript
import Exa from "exa-js";

const exa = new Exa(process.env.EXA_API_KEY);

async function main() {
  const result = await exa.searchAndContents(
    "machine learning papers on transformers",
    {
      type: "neural",
      numResults: 5,
      highlights: { numSentences: 2, highlightsPerUrl: 2 },
      summary: true,
      text: true,
    }
  );

  for (const item of result.results) {
    console.log("Title:", item.title);
    console.log("URL:", item.url);
    console.log("Highlights:", item.highlights);
    console.log("Summary:", item.summary);
    console.log("---");
  }
}

main();
```

---

### 示例 5：cURL — findSimilar

```bash
curl -X POST 'https://api.exa.ai/findSimilar' \
  -H 'x-api-key: YOUR-EXA-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://arxiv.org/abs/2307.06435",
    "numResults": 5,
    "contents": {
      "text": true,
      "highlights": {
        "numSentences": 2
      }
    }
  }'
```

---

## OpenAI SDK 兼容

Exa 提供与 OpenAI SDK 兼容的接口。只需将 `base_url` 设置为 `https://api.exa.ai`，并将 `model` 指定为 `"exa"`，即可使用熟悉的 `openai` Python 包调用 Exa 搜索能力。

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["EXA_API_KEY"],
    base_url="https://api.exa.ai",
)

response = client.chat.completions.create(
    model="exa",
    messages=[
        {
            "role": "user",
            "content": "What are the latest breakthroughs in quantum computing?"
        }
    ]
)

print(response.choices[0].message.content)
```

> 这一兼容性使得已有 OpenAI 集成的应用无需大幅改动代码，即可切换或叠加使用 Exa 的搜索能力。

---

## 独特功能

- **基于 Embedding 的神经搜索**：Exa 的核心搜索算法采用深度语义 embedding，能够理解查询意图而非仅匹配关键词，适合处理自然语言风格的问题式查询，搜索质量显著优于传统关键词引擎。
- **可配置的内容时效性（`maxAgeHours`）**：通过 `maxAgeHours` 参数精确控制缓存内容的最大年龄，支持