# Perplexity API 技术画像

## 概览

Perplexity API Platform 是 Perplexity 面向开发者开放的 AI 推理与搜索基础设施，其核心产品线围绕自研 **Sonar 模型家族**展开。Sonar 家族按能力分为三个层次：**Search 层**（`sonar`、`sonar-pro`）面向快速事实查询、话题摘要与当前事件检索，提供网络溯源的轻量/高级搜索能力；**Reasoning 层**（`sonar-reasoning-pro`）引入 Chain of Thought（CoT）推理，适合多步分析与逻辑决策；**Research 层**（`sonar-deep-research`）执行穷举式网络研究并生成综合报告，支持市场分析、文献综述等场景。

除 Sonar API 外，平台还提供三类补充产品：**Agent API** 以直连提供商价格透明转发 OpenAI、Anthropic、Google、xAI 等第三方模型，并支持 `web_search`、`fetch_url` 工具调用；**Search API** 返回原始网页搜索结果，按请求计费，无 token 费用；**Embeddings API** 提供标准与上下文化两类文本嵌入模型，支持语义搜索与 RAG 应用。

Sonar API 的核心竞争力在于**完全兼容 OpenAI Chat Completions 接口格式**，开发者可直接复用现有 OpenAI SDK，仅需切换 base URL 与 API Key 即可接入网络溯源能力。差异化功能包括：学术与 SEC 文档域名过滤（`search_domain_filter`）、时效性过滤（`search_recency_filter`/日期区间过滤）、可控搜索上下文深度（`search_context_size`：low/medium/high）、Pro Search 多步工具调用模式，以及 Deep Research 专属的 citation tokens 与 reasoning tokens 计费体系。

---

## Base URL

```
https://api.perplexity.ai
```

所有 API 端点均以此为前缀，例如完整 Chat Completions 端点为 `https://api.perplexity.ai/v1/sonar`。

---

## 鉴权

Perplexity API 使用标准 Bearer Token 鉴权，在每个请求的 HTTP Header 中携带：

```
Authorization: Bearer PERPLEXITY_API_KEY
```

### cURL 鉴权示例

```bash
curl https://api.perplexity.ai/v1/sonar \
  -H "Authorization: Bearer $PERPLEXITY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sonar",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### API Key 管理

- 在 [Perplexity API 控制台](https://console.perplexity.ai/) 的 **API Keys** 标签页生成新密钥。
- 建议将 Key 存储为环境变量 `PERPLEXITY_API_KEY`，SDK 会自动读取该变量，无需在代码中硬编码。
- 官方文档提供 [API Key Management](https://docs.perplexity.ai/docs/admin/api-key-management) 页面，可在其中执行 Key rotation（轮换）、吊销旧 Key 等操作。

---

## 端点一览

| 端点 | 方法 | 用途 | 计费基准 |
|------|------|------|----------|
| `/v1/sonar` | `POST` | Chat Completions 兼容的网络溯源对话（Sonar 系列模型） | token 费 + 每千次请求费（按 search_context_size 分档） |
| `/v1/sonar/async` | `POST` | 异步提交深度研究任务（Deep Research），返回任务 ID | 同 Sonar Deep Research 计费规则 |
| `/v1/async/sonar` | `GET` | 列出 / 查询异步任务状态与结果（list/get） | 资料未明确说明单独计费项 |
| `/v1/agent` | `POST` | Agent API：调用第三方模型并支持工具使用（web_search、fetch_url） | 按提供商 token 费 + 工具调用费（$0.005/次 web_search，$0.0005/次 fetch_url） |
| `/v1/search` | `POST` | Search API：返回原始网页搜索结果，支持高级过滤 | $5.00 / 1K 请求，无 token 费 |
| `/v1/embeddings` | `POST` | Embeddings API：生成标准或上下文化文本嵌入向量 | 按 token 计费（$0.004–$0.05 / 1M tokens，视模型而定） |

---

## 端点详解

### POST /v1/sonar

**用途：** 网络溯源的 Chat Completions 端点，兼容 OpenAI Chat Completions 请求/响应格式。适用于快速事实查询、话题摘要、当前事件检索、复杂多步推理（搭配 `sonar-reasoning-pro`）以及深度研究报告（搭配 `sonar-deep-research`）。支持流式（SSE）与非流式两种响应模式。

---

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `model` | `string` | ✅ | — | 使用的模型名称，可选值：`sonar`、`sonar-pro`、`sonar-reasoning-pro`、`sonar-deep-research` |
| `messages` | `array` | ✅ | — | 对话消息数组，每条含 `role`（`system`/`user`/`assistant`）和 `content` 字段，兼容 OpenAI 格式 |
| `stream` | `boolean` | ❌ | `false` | 是否启用 SSE 流式输出；Pro Search（`search_type: pro`）要求 `stream: true` |
| `max_tokens` | `integer` | ❌ | — | 生成输出的最大 token 数 |
| `temperature` | `number` | ❌ | — | 采样温度，控制输出随机性 |
| `top_p` | `number` | ❌ | — | Nucleus sampling 概率阈值 |
| `top_k` | `integer` | ❌ | — | Top-K 采样，仅保留概率最高的 K 个候选 token |
| `presence_penalty` | `number` | ❌ | — | 存在惩罚，降低已出现 token 的再次出现概率 |
| `frequency_penalty` | `number` | ❌ | — | 频率惩罚，按出现频次进一步抑制重复 |
| `return_related_questions` | `boolean` | ❌ | `false` | 是否在响应中附加相关问题建议 |
| `return_images` | `boolean` | ❌ | `false` | 是否在响应中包含图片结果 |
| `search_domain_filter` | `array<string>` | ❌ | — | 限定或排除特定域名，支持学术期刊、SEC 等特殊域名过滤 |
| `search_recency_filter` | `string` | ❌ | — | 按时效性过滤搜索结果，如 `"day"`、`"week"`、`"month"`、`"year"` |
| `search_mode` | `string` | ❌ | — | 资料未明确说明具体可选值 |
| `search_domain` | `string` | ❌ | — | 资料未明确说明（与 `search_domain_filter` 关系资料未明确说明） |
| `search_after_date_filter` | `string` | ❌ | — | 仅返回指定日期之后发布的内容 |
| `search_before_date_filter` | `string` | ❌ | — | 仅返回指定日期之前发布的内容 |
| `last_updated_after_filter` | `string` | ❌ | — | 仅返回最后更新时间在指定日期之后的内容 |
| `last_updated_before_filter` | `string` | ❌ | — | 仅返回最后更新时间在指定日期之前的内容 |
| `web_search_options` | `object` | ❌ | — | 搜索行为高级配置对象，含以下子字段 |
| `web_search_options.search_context_size` | `string` | ❌ | `"low"` | 搜索上下文深度：`"low"`（默认，最快最便宜）、`"medium"`（均衡）、`"high"`（最深，适合研究），直接影响每千次请求费用 |
| `web_search_options.search_type` | `string` | ❌ | `"fast"` | Pro Search 类型：`"fast"`（标准 Sonar Pro 行为）、`"pro"`（多步工具调用，需 `stream: true`，仅 Sonar Pro 可用）、`"auto"`（按查询复杂度自动分类） |
| `reasoning_effort` | `string` | ❌ | — | 推理强度控制，影响 Sonar Deep Research 的搜索查询数量 |
| `response_format` | `object` | ❌ | — | 结构化输出格式配置（如 JSON Schema），详见 Agent API 文档 |
| `file_attachments` | `array` | ❌ | — | 文件附件列表，资料未明确说明具体格式 |

---

#### 响应字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 请求唯一标识符，UUID 格式，如 `"a1b2c3d4-e5f6-7890-abcd-ef1234567890"` |
| `model` | `string` | 实际使用的模型名称，如 `"sonar-pro"` |
| `object` | `string` | 对象类型，固定为 `"chat.completion"` |
| `created` | `integer` | 响应创建的 Unix 时间戳 |
| `choices` | `array` | 生成结果数组 |
| `choices[].index` | `integer` | 候选结果索引，从 `0` 开始 |
| `choices[].message` | `object` | 生成消息对象 |
| `choices[].message.role` | `string` | 消息角色，固定为 `"assistant"` |
| `choices[].message.content` | `string` | 生成的文本内容，含溯源引用标记 |
| `choices[].finish_reason` | `string` | 停止原因，如 `"stop"`、`"length"` |
| `usage` | `object` | token 用量与费用统计 |
| `usage.prompt_tokens` | `integer` | 输入 token 数（含 system message 及上下文） |
| `usage.completion_tokens` | `integer` | 输出 token 数 |
| `usage.total_tokens` | `integer` | 总 token 数（prompt + completion） |
| `usage.search_context_size` | `string` | 本次请求实际使用的搜索上下文深度档位 |
| `usage.num_search_queries` | `integer` | 模型执行的搜索查询次数（Deep Research 专属计费项） |
| `usage.reasoning_tokens` | `integer` | 推理过程消耗的 token 数（Deep Research 专属） |
| `usage.citation_tokens` | `integer` | 用于生成引用/参考文献的 token 数（Deep Research 专属） |
| `usage.cost` | `object` | 费用明细对象 |
| `usage.cost.total_cost` | `number` | 本次请求总费用（美元） |
| `usage.cost.input_tokens_cost` | `number` | 输入 token 费用 |
| `usage.cost.output_tokens_cost` | `number` | 输出 token 费用 |
| `usage.cost.request_cost` | `number` | 每千次请求分摊的请求费用 |
| `usage.cost.reasoning_cost` | `number` | 推理 token 费用（Deep Research 专属） |
| `usage.cost.citation_cost` | `number` | citation token 费用（Deep Research 专属） |
| `search_results` | `array` | 搜索结果来源列表 |
| `search_results[].title` | `string` | 来源页面标题 |
| `search_results[].url` | `string` | 来源页面 URL |
| `search_results[].date` | `string` | 来源页面发布或更新日期 |

---

#### cURL 示例

```bash
curl https://api.perplexity.ai/v1/sonar \
  -H "Authorization: Bearer $PERPLEXITY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sonar-pro",
    "messages": [
      {
        "role": "system",
        "content": "Be precise and concise."
      },
      {
        "role": "user",
        "content": "What are the latest developments in quantum computing?"
      }
    ],
    "web_search_options": {
      "search_context_size": "medium"
    },
    "return_related_questions": true,
    "stream": false
  }'
```

#### JSON 响应示例

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "model": "sonar-pro",
  "object": "chat.completion",
  "created": 1234567890,
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Recent developments in quantum computing include advances in error correction, new qubit architectures, and progress toward fault-tolerant systems..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 14,
    "completion_tokens": 287,
    "total_tokens": 301,
    "search_context_size": "medium",
    "num_search_queries": 2,
    "reasoning_tokens": 0,
    "citation_tokens": 0,
    "cost": {
      "total_cost": 0.0054,
      "input_tokens_cost": 0.000042,
      "output_tokens_cost": 0.004305,
      "request_cost": 0.01,
      "reasoning_cost": 0.0,
      "citation_cost": 0.0
    }
  },
  "search_results": [
    {
      "title": "Quantum Computing Breakthroughs in 2024",
      "url": "https://example.com/quantum-2024",
      "date": "2024-11-15"
    }
  ]
}
```

---

#### 计费说明

`POST /v1/sonar` 的总费用由两部分叠加：

**① Token 费用**（按模型分档）

| 模型 | 输入（$/1M tokens） | 输出（$/1M tokens） | Citation（$/1M tokens） | 搜索查询（$/1K 次） | 推理 tokens（$/1M tokens） |
|------|--------------------|--------------------|------------------------|--------------------|--------------------------|
| `sonar` | $1 | $1 | — | — | — |
| `sonar-pro` | $3 | $15 | — | — | — |
| `sonar-reasoning-pro` | $2 | $8 | — | — | — |
| `sonar-deep-research` | $2 | $8 | $2 | $5 | $3 |

**② 每千次请求费用**（按模型 × `search_context_size` 分档，`sonar-deep-research` 不适用此表）

| 模型 | Low（默认） | Medium | High |
|------|------------|--------|------|
| `sonar` | $5 / 1K 次 | $8 / 1K 次 | $12 / 1K 次 |
| `sonar-pro`（fast） | $6 / 1K 次 | $10 / 1K 次 | $14 / 1K 次 |
| `sonar-pro`（pro search） | $14 / 1K 次 | $18 / 1K 次 | $22 / 1K 次 |
| `sonar-reasoning-pro` | $6 / 1K 次 | $10 / 1K 次 | $14 / 1K 次 |

> **计费公式：** 总费用 = token 费 + 请求费。例如使用 `sonar`（medium context，500 input + 200 output tokens）时：input $0.0005 + output $0.0002 + 请求费 $0.008 = **$0.0087**。

> `sonar-deep-research` 还会额外产生 citation tokens 费与 reasoning tokens 费，单次查询总费用可达 $0.4–$1.3（参见资料中的 Deep Research 示例）。

更多搜索上下文深度指南请参见：[search-context-size-guide](https://docs.perplexity.ai/docs/guides/search-context-size-guide)。

---

### POST /v1/sonar/async

异步 Sonar 接口专为长耗时研究型任务设计，主要配合 `sonar-deep-research` 模型使用。与同步接口不同，提交请求后立即返回一个任务 ID，调用方可在稍后通过轮询端点获取结果，无需保持长连接。

#### 用途

- 适合需要大量搜索、多步推理的复杂研究查询（如市场调研、竞品分析、学术综述）
- 避免因网络超时导致请求失败
- 支持批量提交多个研究任务后统一取回结果

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `model` | string | 是 | — | 模型标识符，推荐使用 `sonar-deep-research` |
| `messages` | array | 是 | — | 对话消息列表，格式同 OpenAI Chat Completions |
| `reasoning_effort` | string | 否 | `"medium"` | 推理深度：`"low"`（速度快、token 少）/ `"medium"`（均衡）/ `"high"`（深度彻底、token 多）|
| `stream` | boolean | 否 | `false` | 是否流式返回（异步模式下建议 `false`）|
| `search_mode` | string | 否 | — | 搜索模式，如 `"academic"` |
| `search_domain` | string | 否 | — | 限定搜索域，如 `"sec"` |
| `search_recency_filter` | string | 否 | — | 结果时效过滤 |

#### 响应字段（提交后立即返回）

| 字段名 | 类型 | 说明 |
|---|---|---|
| `request_id` | string | 异步任务唯一标识符，用于后续轮询 |
| `status` | string | 任务当前状态，如 `"pending"`、`"processing"`、`"completed"`、`"failed"` |
| `created_at` | integer | 任务创建时间（Unix 时间戳）|

完成后通过轮询端点取回的响应结构与同步接口一致（包含 `choices`、`usage`、`citations` 等字段）。

#### 状态轮询端点

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/v1/async/sonar` | 列出当前用户所有异步任务 |
| `POST` | `/v1/async/sonar` | 创建异步任务（即本节所述接口）|
| `GET` | `/v1/async/sonar/{request_id}` | 查询特定任务的状态与结果 |

轮询时建议设置适当间隔（例如每隔 5–10 秒），直到 `status` 变为 `"completed"` 或 `"failed"`。

#### cURL 示例

**提交异步任务：**

```bash
curl --request POST \
  --url https://api.perplexity.ai/v1/async/sonar \
  --header 'accept: application/json' \
  --header 'authorization: Bearer $PERPLEXITY_API_KEY' \
  --header 'content-type: application/json' \
  --data '{
    "model": "sonar-deep-research",
    "messages": [
      {"role": "user", "content": "Provide a comprehensive analysis of the global EV battery supply chain in 2025."}
    ],
    "reasoning_effort": "high"
  }'
```

**轮询任务状态：**

```bash
curl --request GET \
  --url https://api.perplexity.ai/v1/async/sonar/{request_id} \
  --header 'accept: application/json' \
  --header 'authorization: Bearer $PERPLEXITY_API_KEY'
```

#### 计费

`sonar-deep-research` 的具体 per-token 价格及 request fee 资料未明确给出具体数字，请以 [Perplexity 定价页面](https://docs.perplexity.ai/docs/getting-started/pricing) 或控制台为准。`reasoning_effort` 参数会直接影响 reasoning tokens 消耗量，进而影响总成本。

---

### POST /v1/agent

Agent API 是 Perplexity 提供的通用智能体调用接口，核心用途是调用第三方大语言模型（OpenAI、Anthropic、Google、xAI 等），并通过内置工具（`web_search`、`fetch_url`）赋予这些模型实时搜索和网页抓取能力，从而构建具备工具调用能力的自主智能体。

#### 用途

- 调用第三方模型（GPT、Claude、Gemini、Grok 等）并附加 Perplexity 搜索能力
- 结构化输出（Structured Outputs）
- 多步工具调用与模型回退链（fallback chains）
- 兼容 OpenAI Responses API 格式，便于迁移已有系统

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `model` | string | 是 | — | 第三方模型标识符，如 `openai/gpt-4o`、`anthropic/claude-opus-4` 等，具体列表以 `GET /v1/models` 为准 |
| `messages` | array | 是 | — | 对话消息列表，与 OpenAI Chat Completions 格式兼容 |
| `tools` | array | 否 | — | 工具列表，可包含内置工具 `web_search`、`fetch_url`，也可传入自定义函数工具 |
| `stream` | boolean | 否 | `false` | 是否启用流式响应 |
| `temperature` | number | 否 | 模型默认值 | 采样温度 |
| `max_tokens` | integer | 否 | 模型默认值 | 最大输出 token 数 |
| `response_format` | object | 否 | — | 结构化输出格式，如 `{"type": "json_schema", ...}` |

#### 响应字段

响应结构与 OpenAI Chat Completions 格式兼容，工具调用相关字段说明如下：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | string | 响应唯一 ID |
| `model` | string | 实际使用的模型 |
| `choices[].message.role` | string | 消息角色（`assistant`、`tool` 等）|
| `choices[].message.content` | string | 模型生成的文本内容 |
| `choices[].message.tool_calls` | array | 模型请求调用的工具列表，每项含 `id`、`type`、`function.name`、`function.arguments` |
| `choices[].finish_reason` | string | 结束原因：`stop`、`tool_calls`、`length` 等 |
| `usage.prompt_tokens` | integer | 输入 token 数 |
| `usage.completion_tokens` | integer | 输出 token 数 |
| `usage.total_tokens` | integer | 总 token 数 |
| `usage.cost` | object | 费用分项（同 Sonar API 的 `usage.cost` 结构）|

#### cURL 示例

```bash
curl --request POST \
  --url https://api.perplexity.ai/v1/agent \
  --header 'accept: application/json' \
  --header 'authorization: Bearer $PERPLEXITY_API_KEY' \
  --header 'content-type: application/json' \
  --data '{
    "model": "openai/gpt-4o",
    "messages": [
      {"role": "user", "content": "What are the latest developments in fusion energy research?"}
    ],
    "tools": [
      {"type": "web_search"},
      {"type": "fetch_url"}
    ],
    "stream": false
  }'
```

#### 计费说明

Agent API 的计费取决于所调用的第三方模型各自的 token 价格，加上工具调用产生的搜索费用。具体 per-token 费率以各模型在 [Perplexity 定价页面](https://docs.perplexity.ai/docs/getting-started/pricing) 或 `GET /v1/models` 返回信息为准，资料未明确给出统一的汇总表格。

#### 旧端点兼容性

`/v1/responses` 作为 `/v1/agent` 的 alias 继续有效，无需迁移。该设计旨在兼容 OpenAI Responses API 格式，方便已使用 OpenAI SDK 的团队无缝切换。

---

### POST /v1/search

原始 Search API 是一个独立的搜索结果接口，直接返回 Perplexity 搜索引擎从持续刷新的索引中检索到的排序网页列表，**不经过任何 LLM 处理**。响应速度比 Sonar API 更快，适合需要原始搜索数据而非 AI 总结的场景。

#### 用途

- 构建自定义搜索体验（自行处理搜索结果）
- 将搜索结果注入自有 RAG 流水线
- 同时发起多个查询（`query` 支持 array）
- 需要搜索功能但不需要 AI 回答的工作流

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `query` | string \| array | 是 | — | 搜索查询字符串，或查询字符串数组（支持多查询批量提交）|
| `max_results` | integer | 否 | — | 每个查询返回的最大结果数 |
| `max_tokens` | integer | 否 | — | 每次搜索从网页中提取的最大 token 数（控制响应大小与成本）|
| `max_tokens_per_page` | integer | 否 | — | 每个网页提取的最大 token 数 |
| `search_domain` | string | 否 | — | 限定搜索域，如 `"sec"`（SEC 财务文件）|
| `search_domain_filter` | array | 否 | — | 域名白名单或黑名单列表，精确控制来源范围 |
| `search_mode` | string | 否 | — | 搜索模式，如 `"academic"`（学术文献优先）|
| `language_preference` | string | 否 | — | 指定结果的首选语言 |
| `country` | string | 否 | — | 按国家过滤结果 |
| `last_updated_filter` | string | 否 | — | 按网页最后更新时间过滤，格式 `MM/DD/YYYY` |
| `search_after_date_filter` | string | 否 | — | 只返回该日期之后发布的内容，格式 `MM/DD/YYYY` |
| `search_before_date_filter` | string | 否 | — | 只返回该日期之前发布的内容，格式 `MM/DD/YYYY` |
| `search_recency_filter` | string | 否 | — | 结果时效性快捷过滤 |

#### 响应字段

| 字段名 | 类型 | 说明 |
|---|---|---|
| `results` | array | 搜索结果列表，已按相关性排序 |
| `results[].title` | string | 网页标题 |
| `results[].url` | string | 网页 URL |
| `results[].snippet` | string | 摘要文本片段 |
| `results[].date` | string | 内容发布日期 |
| `results[].last_updated` | string | 内容最后更新日期 |

#### cURL 示例

**单查询：**

```bash
curl --request POST \
  --url https://api.perplexity.ai/v1/search \
  --header 'accept: application/json' \
  --header 'authorization: Bearer $PERPLEXITY_API_KEY' \
  --header 'content-type: application/json' \
  --data '{
    "query": "Perplexity AI latest news",
    "max_results": 5,
    "search_mode": "academic"
  }'
```

**多查询批量：**

```bash
curl --request POST \
  --url https://api.perplexity.ai/v1/search \
  --header 'accept: application/json' \
  --header 'authorization: Bearer $PERPLEXITY_API_KEY' \
  --header 'content-type: application/json' \
  --data '{
    "query": [
      "What is Comet Browser?",
      "Perplexity AI",
      "Perplexity Changelog"
    ],
    "max_results": 3
  }'
```

#### JSON 响应示例

```json
{
  "results": [
    {
      "title": "Perplexity AI Raises $500M at $9B Valuation",
      "url": "https://techcrunch.com/2025/01/01/perplexity-funding",
      "snippet": "Perplexity AI has raised $500 million in a new funding round...",
      "date": "2025-01-01",
      "last_updated": "2025-01-02"
    },
    {
      "title": "Perplexity Launches Sonar API",
      "url": "https://research.perplexity.ai/articles/sonar-api",
      "snippet": "The Sonar API provides real-time web-grounded question answering...",
      "date": "2024-12-15",
      "last_updated": "2024-12-16"
    }
  ]
}
```

#### 计费

Search API 按查询次数计费：**$5 / 1,000 queries**。多查询 array 中每个查询项计为独立一次。

---

### POST /v1/embeddings

Embeddings API 将文本转换为高质量的向量表示，适用于语义检索、RAG（Retrieval-Augmented Generation）流水线、相似度计算等场景。Perplexity 提供两种 embedding 类型：

- **Standard Embeddings**：通用向量嵌入，适合大多数语义搜索任务
- **Contextualized Embeddings**：上下文感知向量，在检索任务中具有更强的语义区分能力

#### 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `model` | string | 是 | — | Embedding 模型标识符，具体模型名称以 [Embeddings 文档](https://docs.perplexity.ai/docs/embeddings/quickstart) 为准 |
| `input` | string \| array | 是 | — | 待嵌入的文本字符串，或字符串数组（批量）|
| `encoding_format` | string | 否 | `"float"` | 返回格式：`"float"`（浮点数组）或 `"base64"` |

#### 响应

响应结构与 OpenAI Embeddings API 格式兼容，包含 `data[].embedding`（向量数组）、`data[].index`（输入索引）、`model`（使用的模型）以及 `usage.total_tokens`（消耗 token 数）。

#### 定价说明

Embeddings API 的具体 per-token 价格资料未明确说明，需查看 [Perplexity 控制台](https://console.perplexity.ai/) 或 [定价页面](https://docs.perplexity.ai/docs/getting-started/pricing)。

---

## 模型列表

### Sonar 系列

Sonar 系列是 Perplexity 自研的 web-grounded 对话模型，所有模型均通过 `POST /v1/sonar` 调用（`sonar-deep-research` 推荐使用异步接口）。

#### `sonar`

轻量级 web-grounded 问答模型，适合日常搜索增强对话场景。延迟低、成本最低，适合高并发调用。输入 $1/1M tokens，输出 $1/1M tokens。

#### `sonar-pro`

增强版 Sonar，具备 Pro Search 能力：支持多步工具调用（自动执行多次 `web_search` 与 `fetch_url_content`），输出 token 上限更高，答案更详尽。适合复杂问题、实时信息检索。包含 Media Classifier，可自动判断查询是否需要图片/视频。输入 $3/1M tokens，输出 $15/1M tokens。

#### `sonar-reasoning`

资料说明：根据 Changelog，`sonar-reasoning` 已于 2025 年 12 月 15 日被弃用并从 API 中移除。建议迁移至 `sonar-reasoning-pro`。

#### `sonar-reasoning-pro`

更强推理能力的 Sonar 模型，结合链式思维（Chain-of-Thought）推理与实时 web 检索，适合需要逻辑推断、多步分析的复杂查询。定价资料未明确给出具体数字。

#### `sonar-deep-research`

长耗时研究型模型，专为深度调研设计。可自动执行数十次搜索、汇总多源信息、生成结构化研究报告。因处理时间较长，推荐通过 `POST /v1/sonar/async` 异步接口调用。支持 `reasoning_effort` 参数（`low`/`medium`/`high`）控制推理深度与成本。定价资料未明确给出具体数字。

---

### Agent API 可用模型

Agent API 通过 `POST /v1/agent` 调用，支持来自多家主要 AI 提供商的第三方模型。资料说明支持以下提供商（具体版本以 `GET /v1/models` 返回列表为准）：

| 提供商 | 代表模型（资料提及示例）|
|---|---|
| **OpenAI** | GPT 系列，包括 GPT-5.4（资料提及）|
| **Anthropic** | Claude 系列，包括 Claude Sonnet 4.6（资料提及）|
| **Google** | Gemini 系列，包括 Gemini 3.1 Pro Preview（资料提及）|
| **xAI** | Grok 系列（资料提及支持 xAI）|
| **NVIDIA** | Nemotron 系列（资料提及）|

> **注意**：具体可用模型随时变动（部分模型会被弃用，新模型持续加入），请通过 `GET /v1/models` 获取实时列表。该端点无需认证即可访问。

---

## 搜索与过滤参数

Perplexity API 在多个端点（Sonar API、Search API）中提供丰富的搜索过滤参数，以下逐一说明。

### `search_mode: "academic"`

学术模式：优先从同行评审论文、期刊文章、学术出版物中检索结果，过滤一般网页内容。适用于研究型查询、需要引用权威文献的场景。与 `search_context_size`、日期过滤器等参数兼容。

```json
{
  "search_mode": "academic"
}
```

### `search_domain: "sec"`

SEC 文件过滤：将搜索范围限定在美国证券交易委员会（SEC）官方文件，包括 10-K 年报、10-Q 季报、8-K 当期报告等监管文件。适合金融分析师、合规专员、投资尽职调查等场景。

```json
{
  "search_domain": "sec"
}
```

### `search_recency_filter`

结果时效性快捷过滤参数，用于限制返回内容的发布时间范围（具体可选值资料未明确枚举，常见实现为 `"day"`、`"week"`、`"month"` 等）。

### `search_after_date_filter` / `search_before_date_filter`

按发布日期范围过滤：
- `search_after_date_filter`：只返回该日期**之后**发布的内容
- `search_before_date_filter`：只返回该日期**之前**发布的内容

日期格式：**`MM/DD/YYYY`**

```json
{
  "search_after_date_filter": "01/01/2025",
  "search_before_date_filter": "06/30/2025"
}
```

### `last_updated_after_filter` / `last_updated_before_filter`（`last_updated_filter`）

按网页**最后更新时间**（而非发布日期）过滤。适合查找频繁更新内容的最新版本，如新闻、文档、博客。日期格式同为 **`MM/DD/YYYY`**。在 Changelog 中对应字段名为 `latest_updated`，具体参数名称请以最新 API Reference 为准。

### `search_domain_filter`

域名白名单/黑名单：传入域名数组，精确控制搜索结果来源。可实现仅从特定网站（白名单）或排除特定网站（黑名单）返回结果。

```json
{
  "search_domain_filter": ["reuters.com", "bloomberg.com"]
}
```

### `return_images`

控制是否在响应中返回相关图片。`sonar-pro` 搭配 Media Classifier 可自动检测查询是否适合图片展示，`return_images` 参数允许手动开启或关闭此行为（具体参数名以 API Reference 为准）。

### `return_related_questions`

控制是否在响应中返回相关推荐问题列表，适合构建问答界面的

---

## 速率限制

Perplexity API 采用分层速率限制（Usage Tier）机制。根据官方 changelog（2024-11），**新账户默认对 Sonar 在线模型的速率限制为 50 RPM（Requests Per Minute）**。随着账户用量增长，系统会自动评估并提升 Tier，从而获得更高的请求配额。

具体各层（Tier 0 / 1 / 2 / 3 / 4 / 5）的 RPM 数值以官方 dashboard 实时显示为准，资料未明确列出每一层的精确数字。建议在以下页面查看当前账户所属层级及对应限制：

- [Rate Limits & Usage Tiers](https://docs.perplexity.ai/docs/admin/rate-limits-usage-tiers)
- [API Portal / Console](https://console.perplexity.ai/)

> **注意**：2025-04 起，Perplexity 已移除基于消费层级的功能门控，**所有用户均可访问全部 API 功能**，速率限制仍然适用。

---

## SDK 与代码示例

### 安装

**Python**

```bash
pip install perplexityai
```

**TypeScript / Node.js**

```bash
npm install @perplexity-ai/sdk
```

### 环境变量配置

在项目根目录创建 `.env` 文件：

```env
PERPLEXITY_API_KEY=your_api_key_here
```

---

### 示例 1：Python 简单 sonar 调用

```python
from perplexity import Perplexity

client = Perplexity()  # 自动读取 PERPLEXITY_API_KEY

response = client.chat.completions.create(
    model="sonar",
    messages=[
        {"role": "system", "content": "你是一个简洁的助手。"},
        {"role": "user", "content": "量子计算的核心原理是什么？"}
    ]
)

print(response.choices[0].message.content)
```

---

### 示例 2：Python sonar-pro + web_search_options + 流式输出

```python
from perplexity import Perplexity

client = Perplexity()

stream = client.chat.completions.create(
    model="sonar-pro",
    messages=[
        {"role": "user", "content": "2025 年大模型领域有哪些重大进展？"}
    ],
    stream=True,
    web_search_options={
        "search_context_size": "high"  # 可选 low / medium / high
    }
)

for chunk in stream:
    delta = chunk.choices[0].delta
    if delta.content:
        print(delta.content, end="", flush=True)
```

---

### 示例 3：Python Search API 多 query 调用

```python
from perplexity import Perplexity

client = Perplexity()

results = client.search.create(
    query=["最新 AI 芯片进展", "OpenAI GPT-5 发布", "量子计算 2025"]
)

for item in results:
    print(item.title, item.url)
```

---

### 示例 4：OpenAI SDK 兼容写法（drop-in 替换）

```python
import os
from openai import OpenAI

client = OpenAI(
    base_url="https://api.perplexity.ai",
    api_key=os.environ["PERPLEXITY_API_KEY"]
)

response = client.chat.completions.create(
    model="sonar-pro",
    messages=[
        {"role": "system", "content": "请用中文回答。"},
        {"role": "user", "content": "今天全球科技头条是什么？"}
    ]
)

print(response.choices[0].message.content)
# search_results 包含引用来源
if hasattr(response, "search_results"):
    for src in response.search_results:
        print(src["title"], src["url"])
```

---

### 示例 5：cURL 调用 /v1/sonar

```bash
curl --request POST \
  --url https://api.perplexity.ai/v1/sonar \
  --header 'accept: application/json' \
  --header 'authorization: Bearer $PERPLEXITY_API_KEY' \
  --header 'content-type: application/json' \
  --data '{
    "model": "sonar",
    "messages": [
      {"role": "system", "content": "Be concise."},
      {"role": "user", "content": "What is the capital of France?"}
    ],
    "stream": false
  }'
```

---

## 结构化输出

Perplexity API 支持通过 `response_format` 参数强制模型输出符合指定 JSON Schema 的结果。

**GA 状态**：自 2025-03 起，结构化输出对**所有用户开放**，无需达到特定 Tier。

### 参数说明

在请求体中加入：

```json
{
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "schema": {
        "type": "object",
        "properties": {
          "company": {"type": "string"},
          "revenue_usd_billions": {"type": "number"},
          "fiscal_year": {"type": "integer"}
        },
        "required": ["company", "revenue_usd_billions", "fiscal_year"]
      }
    }
  }
}
```

### 完整示例：从新闻中抽取结构化财务数据

```python
import json
from perplexity import Perplexity

client = Perplexity()

schema = {
    "type": "object",
    "properties": {
        "company": {"type": "string", "description": "公司名称"},
        "revenue_usd_billions": {"type": "number", "description": "营收（十亿美元）"},
        "fiscal_year": {"type": "integer", "description": "财年"}
    },
    "required": ["company", "revenue_usd_billions", "fiscal_year"]
}

response = client.chat.completions.create(
    model="sonar-pro",
    messages=[
        {"role": "user",
         "content": "苹果公司最近一个完整财年的总营收是多少？"}
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {"schema": schema}
    }
)

data = json.loads(response.choices[0].message.content)
print(data)
# 输出示例: {"company": "Apple Inc.", "revenue_usd_billions": 391.0, "fiscal_year": 2024}
```

---

## 文件与图像附件

### 支持的文件格式

Perplexity API 支持将以下文件类型作为上下文附件传入请求：

- **PDF**（`.pdf`）
- **Word 文档**（`.doc`、`.docx`）
- **纯文本**（`.txt`）
- **RTF**（`.rtf`）

文件内容可通过 `file_attachments` 字段传入，模型会将其作为搜索与推理的额外上下文。

### 图像上传（多模态查询）

自 2025-04 起，图像上传功能对所有用户开放。可在 `messages` 数组中通过 `image_url` 传入图像用于多模态查询：

```python
response = client.chat.completions.create(
    model="sonar-pro",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image_url",
                    "image_url": {"url": "https://example.com/chart.png"}
                },
                {
                    "type": "text",
                    "text": "请分析这张图表中的趋势。"
                }
            ]
        }
    ]
)
```

详细用法参见官方指南：[Image Attachments Guide](https://docs.perplexity.ai/guides/image-attachments)

---

## 引用与 search_results

### 字段演进

自 **2025-05** 起，API 响应中新增 `search_results` 字段，**完全取代旧版 `citations` 字段**（`citations` 已被正式废弃并移除）。

`search_results` 每条记录包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `title` | string | 搜索结果页面标题 |
| `url` | string | 来源 URL |
| `date` | string | 内容发布日期 |
| `last_updated` | string | 网页最后更新时间 |
| `snippet` | string | 内容摘要片段 |
| `source` | string | 来源类型（如 `web`）|

### 响应示例

```json
{
  "search_results": [
    {
      "title": "Understanding Large Language Models",
      "url": "https://example.com/llm-article",
      "date": "2023-12-25",
      "last_updated": "2024-05-10",
      "snippet": "",
      "source": "web"
    },
    {
      "title": "Advances in AI Research",
      "url": "https://example.com/ai-research",
      "date": "2024-03-15",
      "last_updated": "2024-03-15",
      "snippet": "",
      "source": "web"
    }
  ]
}
```

### 在应用中解析引用

```python
response = client.chat.completions.create(
    model="sonar-pro",
    messages=[{"role": "user", "content": "最新的量子计算进展是什么？"}]
)

print(response.choices[0].message.content)

# 解析引用来源
if hasattr(response, "search_results") and response.search_results:
    print("\n--- 引用来源 ---")
    for i, src in enumerate(response.search_results, 1):
        print(f"[{i}] {src['title']}")
        print(f"    URL  : {src['url']}")
        print(f"    日期 : {src.get('date', 'N/A')}")
```

**迁移建议**：所有仍使用旧版 `citations` 字段的应用应立即切换至 `search_results`，以获取更丰富的元数据（标题、日期等）。

---

## 集成生态

### MCP Server

Perplexity 提供官方 MCP（Model Context Protocol）Server，支持一键安装到主流 AI 开发工具：

- **Cursor**：通过 MCP 配置文件接入，即可在 Cursor IDE 中调用 Perplexity 搜索能力
- **VS Code**：同样支持 MCP Server 集成
- **Claude Desktop**：在 `claude_desktop_config.json` 中添加 Perplexity MCP Server 配置后即可使用

MCP Server 让 AI Agent 能够直接调用 Perplexity 的实时搜索，无需手动管理 API 调用链。

### n8n 原生集成（2026-04）

[n8n](https://n8n.io/) 工作流自动化平台已原生支持 Perplexity，覆盖以下四个节点类型：

| 节点类型 | 说明 |
|----------|------|
| **Chat Completions** | 对话式问答，支持上下文多轮 |
| **Agent** | 具备工具调用能力的 Agent 节点 |
| **Search** | 直接调用 Perplexity Search API |
| **Embeddings** | 文本向量化，用于 RAG 流程 |

无需编写代码即可将 Perplexity 集成进复杂自动化流水线。

### AWS Marketplace 合并计费（2026-04）

Perplexity API Credits 已上架 AWS Marketplace，企业用户可通过 AWS 账单统一结算，无需单独维护 Perplexity 付款方式。详见：[AWS Marketplace](https://docs.perplexity.ai/docs/resources/aws-marketplace)

### OpenClaw 集成（2026-04）

Perplexity 已与 OpenClaw 平台完成集成，资料中提及此集成已上线，具体功能细节以 [OpenClaw 文档](https://docs.perplexity.ai/docs/getting-started/integrations/openclaw)为准。

---

## 独特功能

1. **OpenAI Chat Completions API 完全兼容**：只需将 `base_url` 改为 `https://api.perplexity.ai`，现有 OpenAI SDK 代码无需其他修改即可切换至 Perplexity，实现零成本 drop-in 替换。

2. **Academic 过滤器**（`search_mode="academic"`）：将搜索范围限定于同行评审论文、期刊文章和学术出版物，为研究人员和学生提供高可信度的学术来源。

3. **SEC Filings 过滤器**（`search_domain="sec"`）：专门搜索 SEC 官方监管文件，包括 10-K、10-Q、8-K 等报告，适合金融分析师、合规人员和投资专业人士。

4. **多步 Pro Search**：`sonar-pro` 和 `sonar-deep-research` 支持多步推理搜索，自动分解复杂查询，逐步检索并整合多源信息，得出更深入的结论。

5. **`reasoning_effort` 可调**：针对 `sonar-deep-research`，可通过 `reasoning_effort`（`"low"` / `"medium"` / `"high"`）在速度、成本与推理深度之间灵活权衡。

6. **文件与图像附件**：支持 PDF、DOC/DOCX、TXT、RTF 作为上下文文件；支持图像 URL 传入用于多模态查询，覆盖文档分析与视觉推理场景。

7. **`usage.cost` 成本分项明细**：API 响应的 `usage.cost` 字段提供完整成本透明度，包含 `input_tokens_cost`、`output_tokens_cost`、`request_cost`、`reasoning_tokens_cost`、`search_queries_cost` 及 `total_cost`，便于精细化成本监控。

8. **API Key Rotation 机制**：支持无中断密钥轮换，可在旧密钥有效期内生成新密钥、迁移应用、再停用旧密钥，支持审计追踪和自动化轮换调度，满足企业合规要求。

9. **MCP Server**：官方 MCP Server 可一键接入 Cursor、VS Code、Claude Desktop，使任意支持 MCP 协议的 AI 工具具备实时搜索能力。

10. **AWS Marketplace 采购**：支持通过 AWS Marketplace 统一采购和合并计费，方便企业整合云支出管理。

---

## 免费额度

新账户通常获得少量免费 credits 用于试用，具体额度以 [dashboard](https://console.perplexity.ai/) 为准。资料未明确公开具体免费 credits 数字。

---

## 优点

1. **OpenAI 兼容，迁移成本极低**：现有 OpenAI SDK 用户只需修改 `base_url` 和 `api_key`，无需重写代码即可接入，是最快速的 drop-in 替换方案之一。

2. **内置实时搜索与引用**：所有 Sonar 模型原生集成网络搜索，响应自动附带 `search_results` 来源引用，大幅降低 RAG 架构搭建复杂度。

3. **丰富的专业搜索过滤器**：Academic 模式、SEC 过滤器、日期范围过滤、地理位置过滤、域名白名单/黑名单等开箱即用，无需额外工程工作。

4. **全功能对所有用户开放**：自 2025-04 起取消 Tier 门控，新用户注册即可使用结构化输出、多模态、异步 API 等所有高级功能。

5. **成本透明度高**：`usage.cost` 字段提供细粒度成本分项，开发者可在每次请求粒度监控费用，无需依赖月度账单反推成本。

6. **生态集成丰富**：官方 MCP Server、n8n 原生节点、AWS Marketplace 合并计费、OpenClaw 集成，覆盖从个人开发者到企业级部署的多种场景。

---

## 缺点

1. **定价结构复杂**：不同模型（sonar / sonar-pro / sonar-deep-research / sonar-reasoning-pro）的计费维度各异，涉及 input tokens、output tokens、request cost、reasoning tokens、search queries 等多个分项，初次使用时难以直觉预估成本。

2. **免费额度未公开透明**：官方未明确公示新账户免费 credits 的具体数额，开发者需登录 dashboard 自行查看，缺乏与竞品横向比较的公开基准。

3. **原生 Search API 生态较新**：独立的 Search API（返回有序搜索结果而非 AI 合成回答）于 2025 年推出，与 Sonar 系列相比，周边示例、社区问答和第三方集成相对较少，开发者参考资源尚不丰富。

4. **模型选择复杂度上升**：随着 sonar、sonar-pro、sonar-deep-research、sonar-reasoning-pro、Agent API 等多条产品线并行存在，且历史模型名称曾多次废弃（llama-3.x-sonar-*、R1-1776 等），维护长期项目时需持续关注 deprecation 通知。

5. **深度研究任务延迟较高**：`sonar-deep-research` 的多步推理搜索耗时较长，虽有异步 API 缓解，但对实时性要求极高的场景仍存在局限。

---

## 近期更新

完整更新日志请见：[https://docs.perplexity.ai/docs/resources/changelog](https://docs.perplexity.ai/docs/resources/changelog)

| 日期 | 类别 | 更新内容 |
|------|------|----------|
| **2026-04** | 集成 | **n8n 原生集成**：覆盖 Chat Completions、Agent、Search、Embeddings 四类节点，无代码工作流可直接接入 Perplexity |
| **2026-04** | 集成 | **OpenClaw 集成**正式上线 |
| **2026-04** | 计费 | **AWS Marketplace API Credits**：支持通过 AWS 账单合并结算 Perplexity API 用量 |
| **2025-05** | 搜索 | **`search_results` 字段 GA**：完全取代旧版 `citations` 字段，新增 title、url、date、last_updated、snippet 等元数据；`citations` 字段同步废弃移除 |
| **2025-06** | 搜索 | **Academic Filter GA**（`search_mode="academic"`）：将搜索限定于同行评审期刊和学术出版物 |
| **2025-07** | 搜索/金融 | **SEC Filings Filter**（`search_domain="sec"`）：专项搜索 SEC 官方监管文件，支持 10-K、10-Q、8-K 等报告 |
| **2025-07** | 计费 | **`usage.cost` 详细成本分项**：API 响应直接返回 input_tokens_cost、output_tokens_cost、request_cost、total_cost 等字段 |
| **2025-05** | 模型 | **`reasoning_effort` 参数**：针对 sonar-deep-research，支持 low/medium/high 三档推理深度控制，平衡速度与成本 |
| **2025-05** | 模型 | **sonar-deep-research 异步 API**：新增 `POST/GET /v1/async/sonar` 端点，异步任务 TTL 为 7 天 |
| **2025-09** | 安全 | **API Key Rotation 机制**：支持无中断密钥轮换、自动化轮换调度、操作审计追踪 |
| **2025-03** | 功能 | **结构化输出对所有用户开放**：移除 Tier 3 门控限制，JSON Schema 输出全用户可用 |
| **2025-04** | 功能 | **全功能解锁**：移除所有基于消费 Tier 的功能门控，所有用户获得完整 API 能力 |
| **2025 年** | 安全/工具 | **MCP Server 发布**：官方 MCP Server 支持一键接入 Cursor、VS Code、Claude Desktop |

---

## 典型场景

### ✅ 适合选择 Perplexity 的场景

- **对话式问答 + 引用溯源**：需要 AI 回答实时问题并自动提供可验证来源链接的应用（新闻摘要、研究助手、客服问答等），`search_results` 字段让引用展示开箱即用。

- **OpenAI SDK drop-in 替换**：现有基于 OpenAI Chat Completions API 构建的项目，希望快速切换至带实时搜索能力的模型，只需修改 `base_url` 即可零改造接入。

- **学术与 SEC 合规研究**：学术研究人员（`search_mode="academic"`）、金融分析师/合规人员（`search_domain="sec"`）需要搜索范围受控、来源权威的专业信息检索场景。

- **多模型 Agent 架构**：通过 Agent API 调用 OpenAI、Anthropic、Google 等多家模型并结合 Perplexity 实时搜索工具，构建具备网络感知能力的复杂 Agent 工作流。

- **n8n / 自动化流水线**：使用 n8n 等 no-code 工具构建信息聚合、定期报告生成等自动化场景，可直接使用 n8n 原生 Perplexity 节点。

### ❌ 不适合选择 Perplexity 的场景

- **超细粒度语义搜索**：若需要对私有文档库进行高精度向量相似度检索（如大规模企业知识库 RAG），专用向量数据库（Pinecone、Weaviate 等）搭配专业 Embedding 模型可能更合适，Perplexity 的核心优势在于公开网络实时搜索而非私有数据检索。

- **极端价格敏感场景**：多维度分项计费结构（tokens + request cost + search queries）在高频调用场景下成本预估难度较高；若预算极为敏感且查询量巨大，需仔细对比实际成本。

---

## 官方文档链接

- [官网](https://www.perplexity.ai/)
- [Docs 首页](https://docs.perplexity.ai/docs/getting-started/overview)
- [Pricing 定价](https://docs.perplexity.ai/docs/getting-started/pricing)
- [Models 模型列表](https://docs.perplexity.ai/docs/sonar/models)
- [Rate Limits & Usage Tiers 速率限制](https://docs.perplexity.ai/docs/admin/rate-limits-usage-tiers)
- [Sonar Quickstart](https://docs.perplexity.ai/docs/sonar/quickstart)
- [Sonar Models](https://docs.perplexity.ai/docs/sonar/models)
- [SDK Overview](https://docs.perplexity.ai/docs/sdk/overview)
- [Changelog 更新日志](https://docs.perplexity.ai/docs/resources/changelog)
- [Sonar API 参考（Create Chat Completion）](https://docs.perplexity.ai/api-reference/sonar-post)
- [Search API 参考（Search the Web）](https://docs.perplexity.ai/api-reference/search-post)
- [Search API Quickstart](https://docs.perplexity.ai/docs/search/quickstart)
- [API Console / Portal](https://console.perplexity.ai/)
- [AWS Marketplace 集成](https://docs.perplexity.ai/docs/resources/aws-marketplace)
- [Academic Filter Guide](https://docs.perplexity.ai/guides/academic-filter-guide)
- [SEC Guide](https://docs.perplexity.ai/guides/sec-guide)
- [Image Attachments Guide](https://docs.perplexity.ai/guides/image-attachments)
- [Pricing Guide](https://docs.perplexity.ai/guides/pricing)