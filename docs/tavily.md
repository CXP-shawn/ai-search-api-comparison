# Tavily 技术画像

## 概览

Tavily 是专为 AI agent 与 RAG 管道设计的实时 Web 接入层，通过单一安全 API 将大语言模型与真实互联网连接。平台围绕五大核心端点构建：**`/search`**（实时网页检索）、**`/extract`**（页面内容抽取）、**`/crawl`**（智能站点爬取）、**`/map`**（站点图谱映射）、**`/research`**（深度综合研究）。所有端点均采用统一的 credit 计费体系，Free 计划每月赠送 1,000 credits，PAYGO 按 $0.008/credit 计费，月度订阅计划最低可至 $0.005/credit。

在性能与可靠性方面，Tavily 承诺 **99.99% SLA**，`/search` 端点 p50 延迟仅 **180 ms**，平台每月处理超过 **1 亿次**请求，注册开发者超过 **100 万**。产品被 Databricks、IBM、JetBrains、LangChain、MongoDB、AWS、BCG 等众多头部企业与机构采用，并与 OpenAI、Anthropic、Groq 等主流 LLM 提供商实现开箱即用集成。所有请求均经过安全、隐私与内容校验层处理，可阻止 PII 泄露、提示注入及恶意来源。

---

## Base URL

```
https://api.tavily.com
```

所有 API 请求均以此为根路径，通过 HTTPS 传输。

---

## 鉴权

Tavily 所有端点均使用 API Key 鉴权，通过 HTTP `Authorization` 请求头以 Bearer Token 方式传递：

```http
Authorization: Bearer tvly-YOUR_API_KEY
```

**API Key 类型说明：**

| Key 类型 | 说明 | RPM 上限 |
|---|---|---|
| `Development` | 免费注册即可获取，无需信用卡 | 100 |
| `Production` | 需激活 **Paid Plan** 或启用 **PAYGO** 后方可使用 | 1,000 |

> **注意：** Production API Key 必须在账户已订阅付费计划或开启按量付费（PAYGO）的前提下才能激活并使用；未满足条件时仅能使用 Development Key。

可在 [app.tavily.com](https://app.tavily.com/) 创建与管理 API Key。

---

## 端点一览

| 端点 | 方法 | 类别 | 说明 |
|---|---|---|---|
| `/search` | POST | 核心 | 实时网页搜索，支持 basic/advanced/fast/ultra-fast 四种深度 |
| `/extract` | POST | 核心 | 从指定 URL 列表抽取结构化内容 |
| `/crawl` | POST | 核心 | 智能站点爬取（Mapping + Extraction 组合） |
| `/map` | POST | 核心 | 站点图谱映射，返回站点页面 URL 列表 |
| `/research` | POST/GET | 核心 | 深度综合研究，支持 `mini`/`pro` 两种模型 |
| `/usage` | GET | 管理 | 查询 API credit 使用量（每 10 分钟最多 10 次） |
| `/generate-keys` | POST | Enterprise | 以编程方式为组织批量生成 API Key |
| `/deactivate-keys` | POST | Enterprise | 以编程方式停用指定 API Key |
| `/key-info` | GET | Enterprise | 查询指定 API Key 的详细信息 |

> Enterprise 端点仅限 **Enterprise Plan** 用户使用。

---

## 端点详解

### POST /search

Tavily 核心搜索端点，针对 AI agent 与 RAG 场景优化，返回结构化、已评分的搜索结果，支持生成摘要答案。

**请求地址：**

```
POST https://api.tavily.com/search
```

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `query` | string | ✅ | — | 搜索查询语句。使用双引号包裹短语（如 `"John Smith" CEO`）可配合 `exact_match` 使用 |
| `search_depth` | string | ❌ | `basic` | 搜索深度：`basic`（1 credit）、`advanced`（2 credits）、`fast` BETA（1 credit，低延迟）、`ultra-fast` BETA（1 credit，极致延迟优先） |
| `topic` | string | ❌ | `general` | 搜索主题类型：`general`（通用网页）、`news`（新闻）、`finance`（财经） |
| `max_results` | integer | ❌ | 资料未明确说明 | 返回搜索结果的最大条数 |
| `time_range` | string | ❌ | — | 时间范围过滤（如 `day`、`week`、`month`、`year`）；与 `start_date`/`end_date` 配合使用 |
| `start_date` | string | ❌ | — | 结果时间范围起始日期（ISO 8601 格式） |
| `end_date` | string | ❌ | — | 结果时间范围截止日期（ISO 8601 格式） |
| `days` | integer | ❌ | — | 返回最近 N 天内的结果，与 `topic=news` 配合使用 |
| `include_answer` | boolean/string | ❌ | `false` | 是否在响应中包含 AI 生成摘要答案：`false`（不包含）、`basic`（简洁答案）、`advanced`（详细答案） |
| `include_raw_content` | boolean | ❌ | `false` | 是否返回每条结果的原始页面内容 |
| `include_images` | boolean | ❌ | `false` | 是否返回搜索相关图片列表 |
| `include_image_descriptions` | boolean | ❌ | `false` | 是否为图片附加描述文本（需同时启用 `include_images`） |
| `include_favicon` | boolean | ❌ | `false` | 是否返回每条结果来源网站的 favicon URL |
| `include_domains` | array[string] | ❌ | — | 限定搜索范围的域名白名单列表（如 `["nytimes.com", "bbc.com"]`） |
| `exclude_domains` | array[string] | ❌ | — | 从结果中排除的域名黑名单列表 |
| `country` | string | ❌ | — | 指定搜索结果偏向的国家/地区（ISO 3166-1 alpha-2 代码） |
| `auto_parameters` | boolean | ❌ | `false` | 启用后由 Tavily 自动推断最佳搜索参数（如 `topic`、`time_range` 等） |
| `exact_match` | boolean | ❌ | `false` | 仅返回包含查询中精确引用短语的结果，适用于尽职调查、法律合规等场景 |
| `chunks_per_source` | integer | ❌ | 资料未明确说明 | 每个来源返回的内容片段数量上限（与 `include_raw_content` 配合） |
| `include_usage` | boolean | ❌ | `false` | 是否在响应中包含本次请求的 credit 消耗信息 |
| `safe_search` | boolean/string | ❌ | 资料未明确说明 | 安全搜索过滤设置 |

**请求头（可选）：**

| Header | 说明 |
|---|---|
| `X-Project-ID` | 项目追踪 ID，用于在多项目场景下区分请求来源，可在 `/usage` 及控制台按项目过滤 |

#### 响应字段

| 字段名 | 类型 | 说明 |
|---|---|---|
| `query` | string | 原始查询语句 |
| `answer` | string/null | AI 生成的摘要答案（`include_answer` 不为 `false` 时返回） |
| `results` | array | 搜索结果列表 |
| `results[].title` | string | 结果页面标题 |
| `results[].url` | string | 结果页面 URL |
| `results[].content` | string | 页面内容摘要（已清洗，适合 LLM 输入） |
| `results[].score` | float | Tavily 相关性评分（0–1） |
| `results[].raw_content` | string/null | 原始页面内容（`include_raw_content=true` 时返回） |
| `results[].favicon` | string/null | 来源网站 favicon URL（`include_favicon=true` 时返回） |
| `results[].images` | array/null | 页面相关图片列表（`include_images=true` 时返回） |
| `response_time` | float | 本次请求响应耗时（秒） |
| `auto_parameters` | object/null | `auto_parameters=true` 时返回自动推断使用的参数 |
| `usage` | object/null | credit 消耗信息（`include_usage=true` 时返回），包含 `credits` 字段 |
| `request_id` | string | 本次请求的唯一标识符 |

#### cURL 示例

```bash
curl -X POST https://api.tavily.com/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer tvly-YOUR_API_KEY" \
  -H "X-Project-ID: my-project" \
  -d '{
    "query": "Latest AI agent frameworks 2025",
    "search_depth": "advanced",
    "topic": "general",
    "max_results": 5,
    "include_answer": "basic",
    "include_images": false,
    "include_favicon": true,
    "include_usage": true
  }'
```

#### JSON 响应示例（节选）

```json
{
  "query": "Latest AI agent frameworks 2025",
  "answer": "In 2025, leading AI agent frameworks include LangGraph, AutoGen, and CrewAI, each offering distinct approaches to multi-agent orchestration.",
  "results": [
    {
      "title": "Top AI Agent Frameworks in 2025",
      "url": "https://example.com/ai-agent-frameworks",
      "content": "LangGraph, AutoGen, and CrewAI are among the most widely adopted frameworks for building production AI agents in 2025...",
      "score": 0.9231,
      "raw_content": null,
      "favicon": "https://example.com/favicon.ico",
      "images": null
    }
  ],
  "response_time": 0.83,
  "usage": {
    "credits": 2
  },
  "request_id": "req_abc123xyz"
}
```

#### Credit 消耗

| `search_depth` | 每次请求消耗 |
|---|---|
| `basic` | **1 credit** |
| `advanced` | **2 credits** |
| `fast` (BETA) | **1 credit**（低延迟优化，兼顾相关性） |
| `ultra-fast` (BETA) | **1 credit**（极致延迟优先） |

---

### POST /extract

Tavily 内容抽取端点，接收一组 URL，返回每个页面经过解析、清洗的结构化文本内容。支持按查询意图对内容片段进行重排序，适用于构建精准 RAG 数据源。

**请求地址：**

```
POST https://api.tavily.com/extract
```

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `urls` | array[string] | ✅ | — | 需要抽取内容的 URL 列表 |
| `extract_depth` | string | ❌ | `basic` | 抽取深度：`basic`（1 credit/5 URLs）、`advanced`（2 credits/5 URLs） |
| `query` | string | ❌ | — | 用户意图描述，提供后将对抽取内容片段按与该查询的相关性进行重排序 |
| `chunks_per_source` | integer | ❌ | `3` | 每个来源返回的最大内容片段数（范围 1–5）。片段为最长 500 字符的内容摘录，仅在提供 `query` 时生效。多片段在 `raw_content` 中以 `<chunk 1> […] <chunk 2>` 格式拼接 |
| `format` | string | ❌ | 资料未明确说明 | 返回内容格式：`markdown` 或 `text` |
| `timeout` | float | ❌ | 资料未明确说明 | 请求超时时间（秒）；可用于控制单次抽取操作的最长等待时间 |
| `include_favicon` | boolean | ❌ | `false` | 是否在每条结果中返回来源网站的 favicon URL |
| `include_usage` | boolean | ❌ | `false` | 是否在响应中包含本次请求的 credit 消耗信息 |
| `include_images` | boolean | ❌ | `false` | 是否返回页面中的图片列表 |

#### 响应字段

| 字段名 | 类型 | 说明 |
|---|---|---|
| `results` | array | 成功抽取的结果列表 |
| `results[].url` | string | 成功抽取的页面 URL |
| `results[].raw_content` | string | 抽取到的页面文本内容（若提供 `query` 则为重排序后的片段拼接） |
| `results[].favicon` | string/null | 来源网站 favicon URL（`include_favicon=true` 时返回） |
| `results[].images` | array/null | 页面图片列表（`include_images=true` 时返回） |
| `failed_results` | array | 抽取失败的 URL 列表及失败原因（**失败的 URL 不计费**） |
| `usage` | object/null | credit 消耗信息（`include_usage=true` 时返回），包含 `credits` 字段。注意：若成功抽取数量未达计费阈值（5 个），该值可能为 0 |

#### cURL 示例

```bash
curl -X POST https://api.tavily.com/extract \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer tvly-YOUR_API_KEY" \
  -d '{
    "urls": [
      "https://docs.tavily.com/documentation/about",
      "https://docs.tavily.com/documentation/api-credits"
    ],
    "extract_depth": "basic",
    "query": "Tavily credit pricing model",
    "chunks_per_source": 3,
    "format": "markdown",
    "include_favicon": true,
    "include_usage": true
  }'
```

#### JSON 响应示例（节选）

```json
{
  "results": [
    {
      "url": "https://docs.tavily.com/documentation/api-credits",
      "raw_content": "Tavily operates on a simple, credit-based model. Free plan includes 1,000 credits/month... [...] Pay-as-you-go is priced at $0.008 per credit... [...] Monthly plans range from $0.0075 to $0.005 per credit.",
      "favicon": "https://docs.tavily.com/favicon.ico",
      "images": null
    },
    {
      "url": "https://docs.tavily.com/documentation/about",
      "raw_content": "Tavily is the web access layer for AI agents, providing real-time search, extraction, and crawling capabilities...",
      "favicon": "https://docs.tavily.com/favicon.ico",
      "images": null
    }
  ],
  "failed_results": [],
  "usage": {
    "credits": 1
  }
}
```

#### Credit 消耗

计费单位为**每 5 个成功抽取的 URL**，抽取失败的 URL **不计入计费**：

| `extract_depth` | 计费规则 |
|---|---|
| `basic` | 每 5 个成功抽取的 URL 消耗 **1 credit** |
| `advanced` | 每 5 个成功抽取的 URL 消耗 **2 credits** |

**示例：** 提交 10 个 URL，其中 8 个成功抽取，`extract_depth=advanced`，则计费为 `ceil(8/5) × 2 = 4 credits`（资料未明确说明是否向上取整，请以实际账单为准）。

---

### POST /crawl

**用途：** 从指定起始 URL 开始，像遍历图结构一样爬取网站，自动发现并抽取多个关联页面的内容。适用于大规模内容聚合、竞争研究、知识库构建等场景。

**端点：** `POST https://api.tavily.com/crawl`

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `url` | `string` | ✅ | — | 爬取起始 URL |
| `max_depth` | `integer` | ❌ | — | 从起始 URL 出发的最大爬取深度（跳数） |
| `max_breadth` | `integer` | ❌ | — | 每层深度最多访问的链接数 |
| `limit` | `integer` | ❌ | — | 本次爬取任务最多处理的页面总数上限 |
| `instructions` | `string` | ❌ | — | 自然语言指令，用于引导爬虫聚焦特定内容或页面类型；提供后可启用 `chunks_per_source` |
| `select_paths` | `string[]` | ❌ | — | 仅爬取与指定路径模式匹配的 URL |
| `select_domains` | `string[]` | ❌ | — | 仅爬取来自指定域名的链接 |
| `exclude_paths` | `string[]` | ❌ | — | 排除与指定路径模式匹配的 URL |
| `exclude_domains` | `string[]` | ❌ | — | 排除来自指定域名的链接 |
| `allow_external` | `boolean` | ❌ | — | 是否允许爬取起始域名以外的外部链接 |
| `extract_depth` | `enum<string>` | ❌ | `basic` | 内容抽取深度：`basic`（1 credit/页）或 `advanced`（2 credits/页） |
| `format` | `enum<string>` | ❌ | `markdown` | 返回内容格式：`markdown` 或 `text` |
| `include_favicon` | `boolean` | ❌ | `false` | 是否在每条结果中返回页面 favicon URL |
| `chunks_per_source` | `integer` | ❌ | `3` | 每个来源最多返回的相关内容片段数（1–5），每段最长 500 字符；仅当提供 `instructions` 时有效 |
| `categories` | `string[]` | ❌ | — | 限定爬取内容的分类类型 |
| `include_images` | `boolean` | ❌ | `false` | 是否在响应中返回页面图片 |
| `include_image_descriptions` | `boolean` | ❌ | `false` | 当 `include_images` 为 `true` 时，为每张图片附加描述文本 |
| `timeout` | `float` | ❌ | `150` | 爬取操作超时秒数，范围 10–150 秒 |
| `include_usage` | `boolean` | ❌ | `false` | 是否在响应中返回本次请求的 credit 消耗信息 |

#### 响应字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `base_url` | `string` | 爬取起始 URL |
| `results` | `object[]` | 爬取到的页面结果数组 |
| `results[].url` | `string` | 页面 URL |
| `results[].raw_content` | `string` | 页面抽取内容；若启用 `chunks_per_source`，格式为 `<chunk 1> [...] <chunk 2> [...]` |
| `results[].favicon` | `string` | 页面 favicon URL（仅当 `include_favicon` 为 `true` 时返回） |
| `results[].images` | `object[]` | 页面图片列表（仅当 `include_images` 为 `true` 时返回） |
| `response_time` | `number` | 请求完成耗时（秒） |
| `usage` | `object` | credit 消耗信息（仅当 `include_usage` 为 `true` 时返回），格式：`{ "credits": N }` |

#### cURL 示例

```bash
curl -X POST https://api.tavily.com/crawl \
  -H "Authorization: Bearer tvly-YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://docs.tavily.com",
    "max_depth": 2,
    "max_breadth": 10,
    "limit": 20,
    "extract_depth": "basic",
    "format": "markdown",
    "include_favicon": true
  }'
```

#### JSON 响应示例

```json
{
  "base_url": "https://docs.tavily.com",
  "results": [
    {
      "url": "https://docs.tavily.com/documentation/about",
      "raw_content": "Tavily is an AI-powered search API designed for LLM agents...",
      "favicon": "https://docs.tavily.com/favicon.png",
      "images": []
    },
    {
      "url": "https://docs.tavily.com/documentation/api-reference/introduction",
      "raw_content": "The Tavily API provides endpoints for Search, Extract, Crawl, Map, and Research...",
      "favicon": "https://docs.tavily.com/favicon.png",
      "images": []
    }
  ],
  "response_time": 12.43,
  "usage": {
    "credits": 20
  }
}
```

#### Credit 消耗

Crawl 的总消耗 = **mapping 费用 + extraction 费用**：

- **mapping（站点链接发现）**：每 10 个 URL 消耗 1 credit
- **extraction（内容抽取）**：
  - `extract_depth=basic`：**1 credit / 页**
  - `extract_depth=advanced`：**2 credits / 页**

> **注意：** `include_usage` 响应中的值在总成功调用量未达到最低计费阈值时可能为 0，详见 Credit 制度详解章节。

---

### POST /map

**用途：** 从指定起始 URL 出发，映射网站链接结构，**仅返回链接列表（URL 树），不抽取页面内容**。适用于站点结构探索、链接分析、为后续 crawl/extract 筛选目标 URL 等场景。

**端点：** `POST https://api.tavily.com/map`

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `url` | `string` | ✅ | — | 映射起始 URL |
| `max_depth` | `integer` | ❌ | — | 从起始 URL 出发的最大链接跳数 |
| `max_breadth` | `integer` | ❌ | — | 每层深度最多跟踪的链接数 |
| `limit` | `integer` | ❌ | — | 本次任务最多返回的 URL 总数上限 |
| `instructions` | `string` | ❌ | — | 自然语言指令，引导映射聚焦特定路径或内容类型 |
| `select_paths` | `string[]` | ❌ | — | 仅映射与指定路径模式匹配的 URL |
| `select_domains` | `string[]` | ❌ | — | 仅映射来自指定域名的链接 |
| `exclude_paths` | `string[]` | ❌ | — | 排除与指定路径模式匹配的 URL |
| `exclude_domains` | `string[]` | ❌ | — | 排除来自指定域名的链接 |
| `allow_external` | `boolean` | ❌ | — | 是否允许映射起始域名以外的外部链接 |
| `categories` | `string[]` | ❌ | — | 限定映射范围的内容分类 |
| `timeout` | `float` | ❌ | `150` | 映射操作超时秒数，范围 10–150 秒 |
| `include_usage` | `boolean` | ❌ | `false` | 是否在响应中返回本次请求的 credit 消耗信息 |

#### 响应字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `base_url` | `string` | 映射起始 URL |
| `results` | `string[]` | 发现的 URL 数组（不含页面内容） |
| `response_time` | `number` | 请求完成耗时（秒） |
| `usage` | `object` | credit 消耗信息（仅当 `include_usage` 为 `true` 时返回） |

#### cURL 示例

```bash
curl -X POST https://api.tavily.com/map \
  -H "Authorization: Bearer tvly-YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://docs.tavily.com",
    "max_depth": 2,
    "max_breadth": 20,
    "limit": 50
  }'
```

#### JSON 响应示例

```json
{
  "base_url": "https://docs.tavily.com",
  "results": [
    "https://docs.tavily.com/documentation/about",
    "https://docs.tavily.com/documentation/api-reference/introduction",
    "https://docs.tavily.com/documentation/api-reference/endpoint/search",
    "https://docs.tavily.com/documentation/api-reference/endpoint/extract",
    "https://docs.tavily.com/documentation/api-reference/endpoint/crawl"
  ],
  "response_time": 5.21,
  "usage": {
    "credits": 5
  }
}
```

#### Credit 消耗

- **每发现 10 个 URL**：消耗 **1 credit**（mapping 计费）
- Map 端点不进行内容抽取，因此**不产生 extraction 费用**

---

### POST /research

**用途：** 执行综合深度研究任务，包含自动规划（planning）、多轮检索（search）、内容抽取（extraction）与结果综合（synthesis）等全流程。适用于需要深度分析、多源交叉验证的复杂研究场景。

**端点：** `POST https://api.tavily.com/research`

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `query` | `string` | ✅ | — | 研究问题或主题 |
| `model` | `enum<string>` | ❌ | — | 使用的研究模型：`pro`（高质量，消耗更多 credit）或 `mini`（轻量，消耗更少 credit） |

> **注意：** 其他详细研究配置参数（如研究深度、来源数量等）资料未明确说明具体字段名，请参考 [官方 Research API 文档](https://docs.tavily.com/documentation/api-reference/endpoint/research)。

#### 响应字段

资料未明确说明 Research 端点的完整响应字段结构，请参考官方文档。

#### cURL 示例

```bash
curl -X POST https://api.tavily.com/research \
  -H "Authorization: Bearer tvly-YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the latest advancements in quantum computing in 2025?",
    "model": "pro"
  }'
```

#### 异步状态轮询

Research 任务通常为异步执行。提交请求后，API 返回一个任务 ID，可通过轮询对应状态端点查询任务进度与最终结果。具体轮询端点与字段资料未明确说明，请参考 [官方 Research API 文档](https://docs.tavily.com/documentation/api-reference/endpoint/research)。

#### Credit 消耗

Research 端点采用**动态计费**，实际消耗取决于研究复杂度（检索轮数、抽取页面数等）：

| 模型 | Credit 消耗范围 |
|---|---|
| `pro` | **15 – 250 credits / 请求** |
| `mini` | **4 – 110 credits / 请求** |

---

### GET /usage

**用途：** 实时检查当前 API key 的用量与订阅计划余额，便于监控账户消耗情况。

**端点：** `GET https://api.tavily.com/usage`

#### 响应字段

资料未明确说明 `/usage` 端点的完整响应字段结构。根据官方说明，该端点可用于监控账户实时用量，请参考 [官方 Usage API 文档](https://docs.tavily.com/documentation/api-reference/endpoint/usage)。

#### cURL 示例

```bash
curl -X GET https://api.tavily.com/usage \
  -H "Authorization: Bearer tvly-YOUR_API_KEY"
```

---

### Enterprise API Key Management

> **上线时间：** 2026 年 3 月（资料标注为 2026-03 上线）

Enterprise 级别的 API Key 管理功能，提供编程化的 key 生成、停用与查询能力，适用于需要批量管理多个 API key 的企业客户。

#### POST /generate-keys

**用途：** 为 Enterprise 账户生成新的 API key。可通过编程方式批量创建 key，便于多团队、多项目隔离管理。

```bash
curl -X POST https://api.tavily.com/generate-keys \
  -H "Authorization: Bearer tvly-YOUR_ENTERPRISE_API_KEY" \
  -H "Content-Type: application/json"
```

> 具体请求/响应参数资料未明确说明，请参考官方 Enterprise API 文档。

#### POST /deactivate-keys

**用途：** 停用指定的 API key，使其立即失效。适用于 key 泄露、员工离职、项目下线等需要及时撤销访问权限的场景。

```bash
curl -X POST https://api.tavily.com/deactivate-keys \
  -H "Authorization: Bearer tvly-YOUR_ENTERPRISE_API_KEY" \
  -H "Content-Type: application/json"
```

> 具体请求/响应参数资料未明确说明，请参考官方 Enterprise API 文档。

#### GET /key-info

**用途：** 查询指定 API key 的详细信息，包括状态、创建时间、关联项目等元数据。

```bash
curl -X GET https://api.tavily.com/key-info \
  -H "Authorization: Bearer tvly-YOUR_ENTERPRISE_API_KEY"
```

> 具体响应参数资料未明确说明，请参考官方 Enterprise API 文档。

---

## search_depth 对比

| `search_depth` | 延迟 / 准确性特征 | 适用场景 | Credit 消耗 |
|---|---|---|---|
| `basic` | 均衡延迟与相关性；每个 URL 返回一条 NLP 摘要 | 通用搜索、快速信息检索 | **1 credit / 请求** |
| `advanced` | 最高相关性，延迟较高；每个 URL 返回多条语义相关片段（可通过 `chunks_per_source` 配置，最多 3 条） | 精度要求高的深度查询、专业研究 | **2 credits / 请求** |
| `fast`（BETA） | 低延迟同时保持较高相关性；每个 URL 返回多条语义相关片段（可通过 `chunks_per_source` 配置） | 对速度有要求但仍需较高准确性的场景 | **1 credit / 请求** |
| `ultra-fast`（BETA） | 严格优化延迟，速度最快；每个 URL 返回一条 NLP 摘要 | 时间极敏感的场景，如实时应用、低延迟 pipeline | **1 credit / 请求** |

> **注意：** `fast` 和 `ultra-fast` 目前处于 BETA 阶段。`safe_search` 过滤功能不支持 `fast` 和 `ultra-fast` 深度（Enterprise 专属功能）。

---

## Credit 制度详解

### 什么是 Credit

Credit 是 Tavily API 的计量单位。每次成功调用 API 均会根据所用端点和参数配置消耗相应数量的 credit。**仅成功调用计费**，失败请求（如 4xx/5xx 错误）不消耗 credit。

**PAYGO 单价：** **$0.008 / credit**（即 1,000 credits = $8.00）

### 各端点 Credit 成本汇总

| 端点 | 配置 | Credit 消耗 |
|---|---|---|
| `/search` | `search_depth=basic` | 1 credit / 请求 |
| `/search` | `search_depth=advanced` | 2 credits / 请求 |
| `/search` | `search_depth=fast`（BETA） | 1 credit / 请求 |
| `/search` | `search_depth=ultra-fast`（BETA） | 1 credit / 请求 |
| `/extract` | `extract_depth=basic` | 1 credit / 5 个成功 URL |
| `/extract` | `extract_depth=advanced` | 2 credits / 5 个成功 URL |
| `/crawl` | mapping 部分 | 1 credit / 10 个发现 URL |
| `/crawl` | extraction 部分（`extract_depth=basic`） | 1 credit / 页 |
| `/crawl` | extraction 部分（`extract_depth=advanced`） | 2 credits / 页 |
| `/map` | — | 1 credit / 10 个发现 URL |
| `/research` | `model=pro` | 15–250 credits / 请求（动态） |
| `/research` | `model=mini` | 4–110 credits / 请求（动态） |

### 实际计费触发时机

- **Search 端点**：每次成功请求立即计费。
- **Extract 端点**：按**每 5 个成功抽取 URL** 计费一次。若成功 URL 数量未达到 5 的倍数阈值，当次不计费该部分 credit。`include_usage` 响应值可能为 0，表示尚未达到计费阈值。
- **Crawl 端点**：mapping 部分按每 10 个 URL 计费；extraction 部分按成功抽取的页面数即时计费。
- **Map 端点**：按每 10 个发现 URL 计费。
- **Research 端点**：动态计费，根据实际执行的检索与抽取步骤数量计算，完成后汇总扣除。

> **提示：** 可在 Search、Extract、Crawl、Map 请求中设置 `include_usage: true`，在每次响应中实时查看本次调用消耗的 credit 数量（`usage.credits` 字段）。

---

## 订阅计划

### Free（免费计划）

- **额度：** 1,000 credits / 月
- **无需绑定信用卡**，注册即可使用
- 适合个人开发者评估与小规模测试
- 仅限测试 API key，不支持生产环境部署

### Pay-as-you-go（按量付费）

- **单价：** $0.008 / credit
- 无月度承诺，按实际用量计费
- **需要启用生产（production）API key**，需绑定支付方式
- 适合用量不稳定或处于快速增长阶段的项目

### 月度订阅（Researcher 等套餐）

- 提供预购 credit 包，月度固定费用，通常比 PAYGO 更具成本优势
- 资料未明确说明具体价格，请核对 [官方 pricing 页面](https://tavily.com/) 获取最新套餐详情

### Enterprise（企业计划）

- **自定义 credit 额度**，支持高并发与大规模调用需求
- 开放 **`safe_search`** 内容安全过滤功能（Enterprise 专属）
- 提供 **API Key Management** 功能（`/generate-keys`、`/deactivate-keys`、`/key-info`），支持编程化 key 管理
- 支持**团队角色与权限管理**（Owner / Admin / Member）
- **自定义定价**，请联系 Tavily 商务团队获取报价
- 支持专属 SLA、优先技术支持等增值服务

---

## 速率限制

下表列出各端点在不同计划下的每分钟请求数（RPM）限制：

| 端点 | Development RPM | Production RPM | 说明 |
|------|----------------|----------------|------|
| `/search` | 100 | 1,000 | 标准搜索端点 |
| `/extract` | 100 | 1,000 | 内容提取端点 |
| `/map` | 100 | 1,000 | 站点地图端点 |
| `/usage` | 100 | 1,000 | 用量查询端点 |
| `/crawl` | 100 | 100 | 爬取端点，两种计划相同 |
| Research（Tier 0） | 5 | 5 | 入门层级固定 5 RPM |
| Research（Tier 1+） | 资料未明确说明 | 资料未明确说明 | 更高 Tier 分层限制请参考控制台 |

### 429 错误处理建议

当请求超过速率限制时，API 将返回 HTTP `429 Too Many Requests`。建议采取以下措施：

- **读取 `Retry-After` header**：响应头中的 `Retry-After` 字段会指示需等待的秒数，客户端应在该时间后重试请求。
- **指数退避（Exponential Backoff）**：若未收到 `Retry-After`，建议以指数退避策略（如 1s → 2s → 4s）进行重试，避免进一步加剧限流。
- **请求队列**：对于批量任务，建议在客户端侧维护请求队列并控制并发，主动将发送速率控制在限额之内。
- **升级计划**：如持续触碰限制，可考虑升级至 Production 或更高 Tier 计划以获取更高配额。

---

## SDK 与代码示例

### 安装

**Python SDK**

```bash
pip install tavily-python
```

**JavaScript / TypeScript SDK**

```bash
npm install @tavily/core
```

**Vercel AI SDK 集成**

```bash
npm install @tavily/ai-sdk
```

### 示例 1：Python 简单搜索

```python
import os
from tavily import TavilyClient

client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])
response = client.search(query="latest LLM papers", search_depth="basic")
print(response)
```

### 示例 2：Python 高级答案 + 分块 + 新闻主题

```python
import os
from tavily import TavilyClient

client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])
response = client.search(
    query="AI regulation news 2025",
    include_answer="advanced",
    chunks_per_source=3,
    topic="news"
)
print(response)
```

> `include_answer="advanced"` 会生成更详尽的 LLM 摘要；`chunks_per_source=3` 在 `search_depth="advanced"` 下每个来源最多返回 3 个内容片段；`topic="news"` 聚焦实时新闻源。

### 示例 3：Python Extract 多 URL + Reranking Query

```python
import os
from tavily import TavilyClient

client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])
response = client.extract(
    urls=[
        "https://en.wikipedia.org/wiki/Artificial_intelligence",
        "https://openai.com/research/gpt-4"
    ],
    query="large language model architecture"  # 用于对提取结果重排序
)
print(response)
```

### 示例 4：JavaScript 搜索

```javascript
import { tavily } from "@tavily/core";

const client = tavily({ apiKey: process.env.TAVILY_API_KEY });

const response = await client.search("latest LLM papers", {
  searchDepth: "basic",
  maxResults: 5
});
console.log(response);
```

### 示例 5：cURL 调用 `/crawl`

```bash
curl -X POST https://api.tavily.com/crawl \
  -H "Authorization: Bearer $TAVILY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://docs.tavily.com",
    "max_depth": 2,
    "max_breadth": 10,
    "limit": 50
  }'
```

---

## 结构化输出与查询优化

### `auto_parameters`

`auto_parameters`（布尔值，默认 `false`）启用后，Tavily 会根据查询内容与意图自动配置搜索参数。用户手动指定的参数优先级高于自动值，始终覆盖自动配置。需注意：`include_answer`、`include_raw_content` 和 `max_results` 三个参数因直接影响响应体大小，**必须始终手动设置**。此外，`search_depth` 可能被自动设为 `advanced`（消耗 2 个 API Credit），若需避免额外费用，请显式将其设为 `basic`。

### `exact_match`（2026-02）

`exact_match`（布尔值，默认 `false`）确保搜索结果中仅返回包含查询中精确引用短语的页面，绕过同义词或语义变体匹配。使用方式：在查询字符串中用英文双引号包裹目标短语，例如 `"John Smith" CEO Acme Corp`。引号内的标点符号通常被忽略。

### `include_answer`：`basic` / `advanced`

`include_answer` 控制是否在响应中附带 LLM 生成的答案：
- 设为 `true` 或 `"basic"` 时返回简短摘要答案；
- 设为 `"advanced"` 时返回更详细、更完整的答案。

默认值为 `false`（不生成答案）。

### `chunks_per_source`（1–5）

`chunks_per_source`（整数，默认 `3`，有效范围 `1 ≤ x ≤ 3`）定义每个来源最多返回的内容片段数量。每个 chunk 最多 500 个字符，多个 chunk 在 `content` 字段中以 `<chunk 1> [...] <chunk 2> [...] <chunk 3>` 格式呈现。**仅在 `search_depth` 为 `advanced` 时可用。**

> 注：资料中原始范围描述存在出入（文档正文一处写 `1 < x < 3`，另一处写 `1 <= x <= 3`，API 参考页最终标注为 `1 <= x <= 3`），以 API 参考页为准。

### `X-Project-ID` header（2026-01）

资料未明确说明 `X-Project-ID` header 的具体用途与使用方式，仅记录其引入时间为 2026 年 1 月。

### `safe_search`（Enterprise 专属）

`safe_search`（布尔值，默认 `false`）为 🔒 **Enterprise 计划专属**功能，启用后将过滤搜索结果中的成人或不安全内容。**不支持** `fast` 或 `ultra-fast` 搜索深度。

---

## 安全与内容过滤

Tavily 在多个层面内置了安全与内容保护机制：

- **PII 过滤**：内置个人身份信息（PII）识别与过滤，防止敏感数据出现在搜索结果中。
- **Prompt Injection 防护**：内置针对 prompt injection 攻击的检测与拦截，保障 AI 应用在使用搜索结果时的安全性。
- **恶意源过滤**：自动识别并排除已知恶意或不可信来源，确保返回内容的可靠性。
- **`safe_search`**：Enterprise 计划用户可通过启用 `safe_search` 参数进一步过滤成人或不安全内容（详见上节）。
- **网络隐私与校验层**：网络隐私保护与内容校验层在所有端点默认启用，无需额外配置。

---

## SLA 与性能指标

| 指标 | 数值 | 说明 |
|------|------|------|
| 服务可用性 SLA | 99.99% | uptime 保障 |
| `/search` P50 延迟 | 180 ms | 中位数响应时间 |
| 月请求量 | >1 亿次（>100M） | 平台月均处理请求规模 |
| 开发者数量 | 100 万+（1M+） | 使用 Tavily API 的开发者数 |
| 单次查询最大文档数 | 10 | 每次查询最多返回 10 份文档 |

> 以上数据来自 Tavily 官方公开资料。具体 SLA 条款以企业合同为准，资料未明确说明具体合同细则。

---

## 集成生态

- **LangChain**：Tavily 作为官方工具节点内置于 LangChain，可直接通过 `TavilySearchResults` 工具在 Agent 链中调用，无需额外封装。
- **LlamaIndex**：提供原生 `TavilyToolSpec`，可作为 LlamaIndex Agent 的检索工具，支持结构化搜索结果直接注入上下文。
- **Vercel AI SDK v5**：官方发布 `@tavily/ai-sdk` 包，作为 Vercel AI SDK v5 的原生 provider，支持在 Next.js 等前端框架中开箱即用地调用 Tavily 搜索能力。
- **MCP Server**：提供 Model Context Protocol (MCP) Server 实现，允许兼容 MCP 的客户端（如 Claude Desktop）直接调用 Tavily 的搜索与提取功能。
- **Databricks MCP Marketplace**：Tavily 与 Databricks 合作，已上架 Databricks MCP Marketplace，企业用户可在 Databricks 数据智能平台内直接集成 Tavily。
- **Composio**：通过 Composio 平台集成，支持在 Composio 的 Agent 工具链中快速接入 Tavily 搜索，降低配置成本。
- **Make**：在 Make（原 Integromat）自动化平台中提供官方模块，可在无代码工作流中触发 Tavily 搜索。
- **n8n**：n8n 工作流平台中可通过 HTTP Request 节点或社区节点调用 Tavily API，适合构建自动化信息采集流水线。
- **Agno**：Tavily 作为 Agno Agent 框架的内置搜索工具之一，可直接在 Agno 的 Agent 定义中引用。
- **Pydantic AI**：Pydantic AI 框架原生支持将 Tavily 作为工具函数注入，利用 Pydantic 的类型校验确保参数安全传递。
- **OpenAI / Anthropic / Groq drop-in**：Tavily 的搜索结果格式兼容 OpenAI、Anthropic、Groq 等主流模型的工具调用（function calling / tool use）格式，可作为这些模型的外部工具直接挂载，无需额外适配层。

---

## 独特功能

- **Credit 统一计费，$0.008/credit 透明定价**：所有端点统一以 credit 计量，每个 credit 固定价格为 $0.008，不同端点按消耗 credit 数计费，账单清晰可预测，无隐藏费用。
- **99.99% SLA 承诺**：在付费计划中提供 99.99% 可用性 SLA，这在搜索 API 市场中属于明确承诺，适合对可靠性要求极高的生产环境。
- **五端点开箱即用**：平台提供 `/search`（网络搜索）、`/extract`（网页内容提取）、`/crawl`（多页爬取）、`/map`（站点链接映射）、`/research`（深度研究）五个语义明确的端点，覆盖 AI Agent 信息获取的全链路需求。
- **`auto_parameters` 自动参数推断**：根据查询的语义意图自动调整搜索参数（如搜索深度、结果数量等），减少手动调参负担，适合查询多样性高的场景。
- **`exact_match` 精确短语匹配**：支持在搜索请求中指定必须精确出现的短语，确保结果与关键词高度相关，适合合规审计、引用核查等对精确度要求严格的场景。
- **`X-Project-ID` 项目级用量隔离**：通过请求头 `X-Project-ID` 或 SDK 参数 `project_id` 为不同项目标记用量，支持多租户或多产品线场景下的用量分账与审计。
- **内置 PII / prompt injection / 恶意源过滤**：平台在返回结果前自动过滤个人身份信息（PII）、检测 prompt injection 攻击及恶意来源，为 AI 应用提供安全防护层，降低安全合规成本。
- **`@tavily/ai-sdk` Vercel AI SDK v5 原生支持**：专为 Vercel AI SDK v5 设计的官方 npm 包，与 Next.js Server Actions、Edge Runtime 深度兼容，是全栈 AI 应用的首选集成方式。
- **Certification 积分奖励**：用户完成 Tavily 官方认证课程后可获得额外 credit 奖励，降低学习与试用门槛。
- **免费 1,000 credits/month，无需信用卡**：免费套餐每月提供 1,000 credits，注册即用，无需绑定信用卡，适合个人开发者与原型验证。

---

## 免费额度

- **每月 1,000 credits 免费**：注册账号后自动获得，每月重置，无需提供信用卡信息，可调用所有端点。
- **Certification 额外积分**：完成 Tavily 官方认证考试后，可获得额外的 credit 奖励（具体数额以官方公告为准），适合希望扩大免费用量的开发者。

---

## 优点

- **专为 AI Agent 设计的结构化输出**：返回结果经过清洗与结构化处理，包含标题、URL、摘要、全文内容等字段，可直接注入 LLM context，无需额外解析，显著降低 Agent 开发复杂度。
- **生态覆盖广泛，集成成本极低**：原生支持 LangChain、LlamaIndex、Vercel AI SDK、MCP、Composio、n8n 等主流 AI 框架与自动化平台，开发者几乎不需要编写额外适配代码即可在现有工作流中引入 Tavily。
- **安全性内置，适合企业级场景**：平台层面提供 PII 过滤、prompt injection 检测、恶意源屏蔽等安全能力，以及 `X-Project-ID` 项目级用量隔离和 Enterprise API key 管理，满足企业合规与多团队管理需求。
- **定价透明且可预测**：统一的 credit 计费模型（$0.008/credit）配合 99.99% SLA 承诺，使得生产环境的成本估算与服务可靠性预期都有明确依据，适合将搜索能力嵌入正式产品的团队。
- **快速迭代、功能持续演进**：2025–2026 年间密集发布 `exact_match`、`auto_parameters`、`chunks_per_source`、Intent-Based Extraction、Enterprise key 管理等功能，产品演进速度快，能够跟上 AI 应用快速变化的需求。

---

## 缺点

- **不暴露 embedding 或神经检索接口**：Tavily 仅提供基于网络搜索与内容提取的结果，不提供向量 embedding 生成或语义相似度检索能力，若需要对私有知识库进行神经检索，需另行集成向量数据库与 embedding 服务。
- **生产 API key 需付费计划或 PAYGO**：免费套餐的 1,000 credits/month 仅适合测试与原型开发，生产环境需升级至付费订阅计划或按量付费（PAYGO）模式，才能获得足够的配额与 SLA 保障。
- **单次查询最多返回 10 documents**：`/search` 端点单次请求的 `max_results` 上限为 10，若需要更大规模的结果集，需要多次请求并自行合并，增加了延迟与 credit 消耗。
- **`/search` 端点不支持 streaming**：搜索请求为同步阻塞模式，结果一次性返回，不支持流式输出（streaming），在搜索耗时较长时用户体验受限，需要在应用层自行实现等待状态处理。

---

## 近期更新

- **2026-03**：发布 Enterprise API key 管理功能，新增 `generate-keys`、`deactivate-keys`、`key-info` 三个管理端点，支持企业在平台内自助生成、停用与查询 API key，满足多团队、多服务的 key 生命周期管理需求。
- **2026-02**：新增 `exact_match` 参数，支持在搜索请求中指定必须精确匹配的短语，提升结果对特定关键词的覆盖精度，适用于合规审计、事实核查等场景。
- **2026-01**：推出 `X-Project-ID` 请求头与 SDK `project_id` 参数，实现项目级用量追踪与隔离，支持多产品线或多客户场景下的分账统计与用量审计。
- **2025-12**：`/extract` 端点引入 Intent-Based Extraction，支持传入 `query` 参数对抓取内容进行 chunk 级重排序，使提取结果与查询意图更高度相关，减少无关噪声。
- **2025-12**：新增 `chunks_per_source` 参数，允许开发者控制每个来源返回的文本块数量，在精度与 token 用量之间灵活权衡。
- **2025**：推出 `auto_parameters` 模式，系统根据查询语义意图自动推断并设置最优搜索参数，降低手动调参门槛，提升开箱即用体验。
- **2025**：与 Databricks 达成合作，Tavily MCP Server 上架 Databricks MCP Marketplace，企业用户可在 Databricks 数据智能平台内直接集成 Tavily 搜索能力。

完整变更记录请参阅：[https://docs.tavily.com/changelog](https://docs.tavily.com/changelog)

---

## 典型场景

### 何时选择 Tavily

- **构建实时信息感知的 AI Agent**：当 Agent 需要访问互联网获取最新新闻、价格、法规、技术文档等实时信息时，Tavily 的结构化搜索结果可直接注入 LLM context，是最低摩擦的选择。
- **多框架 AI 应用快速原型**：项目已使用 LangChain、LlamaIndex、Vercel AI SDK 或 Pydantic AI 等框架时，Tavily 的原生集成使得添加网络搜索能力只需数行代码。
- **企业级多团队管理**：需要对不同项目或客户的 API 用量进行隔离统计、独立 key 管理时，`X-Project-ID` 与 Enterprise key 管理功能提供了开箱即用的解决方案。
- **对安全合规有较高要求**：内置 PII 过滤与 prompt injection 防护的场景，如医疗、金融、法律类 AI 应用，可借助 Tavily 的安全层降低合规风险。
- **内容提取与深度研究任务**：需要从指定 URL 提取结构化内容、爬取站点多页或生成综合研究报告时，`/extract`、`/crawl`、`/map`、`/research` 端点提供了完整的工具链。

### 何时不选择 Tavily

- **需要向量 embedding 或私有知识库语义检索**：Tavily 不提供 embedding 生成或向量检索能力，此类需求应选择 Pinecone、Weaviate、OpenAI Embeddings 等专门服务。
- **需要超大规模搜索结果集**：单次查询最多 10 条结果的限制不适合需要批量抓取数百条结果的场景，此时可考虑 SerpAPI 等支持更大 `num` 参数的服务。
- **对流式搜索响应有强需求**：`/search` 不支持 streaming，若应用需要逐步呈现搜索进度（如流式 Agent 思考过程的实时展示），需在应用层做额外处理或选择支持 streaming 的替代方案。
- **预算极度敏感且用量极高**：免费额度用尽后需付费，高频场景下 credit 消耗可观，需提前评估成本。

---

## 官方文档链接

- [Tavily 官网](https://tavily.com)：产品介绍、定价方案与注册入口。
- [文档首页](https://docs.tavily.com)：完整技术文档的导航起点，涵盖快速入门、API 参考与 SDK 指南。
- [Credits & Pricing](https://docs.tavily.com/documentation/api-credits)：详细说明各端点的 credit 消耗规则与计费标准。
- [Rate Limits](https://docs.tavily.com/documentation/rate-limits)：各套餐的请求速率限制与并发配额说明。
- [API Reference 概览](https://docs.tavily.com/documentation/api-reference/introduction)：所有端点的整体介绍、认证方式与通用请求规范。
- [Search 端点参考](https://docs.tavily.com/documentation/api-reference/endpoint/search)：`/search` 的完整参数说明与响应格式。
- [Extract 端点参考](https://docs.tavily.com/documentation/api-reference/endpoint/extract)：`/extract` 的请求参数、Intent-Based Extraction 用法与响应结构。
- [Crawl 端点参考](https://docs.tavily.com/documentation/api-reference/endpoint/crawl)：`/crawl` 多页爬取的配置选项与结果格式。
- [Map 端点参考](https://docs.tavily.com/documentation/api-reference/endpoint/map)：`/map` 站点链接映射的参数与返回结构说明。
- [Usage 端点参考](https://docs.tavily.com/documentation/api-reference/endpoint/usage)：查询账户 credit 用量的接口说明。
- [Changelog](https://docs.tavily.com/changelog)：平台功能更新的完整历史记录，按时间倒序排列。
- [Python SDK 快速入门](https://docs.tavily.com/sdk/python/quick-start)：Python 客户端安装、认证与基础用法示例。
- [JavaScript SDK 快速入门](https://docs.tavily.com/sdk/javascript/quick-start)：JavaScript/TypeScript 客户端安装、认证与基础用法示例。