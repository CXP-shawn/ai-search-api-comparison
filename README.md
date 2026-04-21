# AI 搜索 API 对比：Exa vs Perplexity vs Tavily

> **编写日期**：2026-04-21  
> **数据来源**：Exa、Perplexity、Tavily 各家官方文档及定价页面（截取自 2026-04-21）  
> **报告立场**：本报告为独立技术对比，力求客观呈现各平台功能与定价差异。Exa 自家的 versus 页面数据已在报告中注明来源，供参考但不作为唯一依据。

---

## 执行摘要

- **定位清晰、各有所长**：Exa 以嵌入式神经搜索见长，适合语义检索与 AI 原生应用；Perplexity 提供端到端的 web 驱动问答，兼顾多模型 Agent 调用；Tavily 以 credit 计费、开箱即用的 RAG 工具链著称，在 RAG 原型开发中具有最低上手门槛。
- **定价结构差异显著**：Exa 采用 request 计费（Search $7/1K requests 起），Perplexity 采用 token + request 双重计费（Sonar $1/1M tokens + $5/1K requests 起），Tavily 采用 credit 制（基础搜索 $0.008/credit）。三者适合不同的调用频率和预算结构。
- **多步研究能力已成标配**：三家均提供深度研究端点（Exa `/search?type=deep-reasoning`、Perplexity `sonar-deep-research`、Tavily `/research`），但实现方式和计费差异明显。
- **OpenAI 兼容性普遍支持**：三家均可通过修改 `base_url` 与 OpenAI SDK 兼容，降低迁移成本。
- **稳定性指标仅 Tavily 公开承诺**：Tavily 明确公布 99.99% uptime SLA；Exa 和 Perplexity 在 Enterprise 计划下提供 SLA，但标准层未披露具体数值。

**默认推荐**：若你是正在构建 RAG 流水线的独立开发者，从 **Tavily** 的免费层起步最为顺畅；若需要高质量语义/学术检索，优先评估 **Exa**；若需要多模型 Agent + 问答一体化方案，**Perplexity** 是最完整的选项。

---

## 侧向对比表

| 维度 | Exa | Perplexity | Tavily |
|---|---|---|---|
| **主站** | https://exa.ai | https://www.perplexity.ai | https://tavily.com |
| **文档** | https://docs.exa.ai | https://docs.perplexity.ai | https://docs.tavily.com |
| **核心 endpoints** | `/search`、`/answer`、`/contents`、`/findsimilar`、`/monitors`、`/chat/completions`、`/research`（已废弃） | Sonar API `/v1/sonar`、Agent API `/v1/agent`、Search API、Embeddings API、Async Sonar `/v1/async/sonar` | `/search`、`/extract`、`/map`、`/crawl`、`/research`、`/usage` |
| **定价模式** | 按 request 计费，部分按 page 计费 | 按 token + request 双重计费；Agent API 按 token + 工具调用次数 | Credit 制；不同 endpoint 消耗不同 credit 数量 |
| **头部定价（标准搜索）** | Search $7/1K requests（≤10 结果）；Deep Search $12/1K；Deep-Reasoning $15/1K | Sonar $5/1K requests（low context）+ token 费用；Sonar Pro $6/1K（low context）+ token 费用 | Basic/Fast/Ultra-fast 搜索 1 credit/request（$0.008/credit PAYG）；Advanced 2 credits/request |
| **内容抽取定价** | Contents $1/1K pages（每种内容类型分别计费） | fetch_url 工具 $0.0005/次（Agent API） | Extract basic 1 credit/5 URLs；Advanced 2 credits/5 URLs |
| **免费额度** | 1,000 requests/月；另有 Startup/Education Grant $1,000 credits | 未在官方文档中披露 | 1,000 credits/月，无需信用卡 |
| **速率限制** | `/search` `/findsimilar` `/answer`：10 QPS；`/contents`：100 QPS；`/research`：15 并发任务 | Sonar 在线模型默认 50 requests/min（2024 年 11 月起）；详细分级见官方 Rate Limits 页面 | 默认 Development 100 RPM，Production 1,000 RPM；`/crawl` 100 RPM；`/research` 20 RPM |
| **Enterprise 速率限制** | 可定制 QPS，联系 hello@exa.ai | 按使用层级分级，具体数值见官方文档 | 企业计划自定义 |
| **官方 SDK** | Python (`exa-py`)、JavaScript/TypeScript (`exa-js`)、cURL | Python (`perplexityai`)、TypeScript (`@perplexity-ai/perplexity_ai`) | Python SDK、JavaScript SDK、`@tavily/ai-sdk`（Vercel AI SDK v5） |
| **OpenAI SDK 兼容** | ✅ 通过 `base_url` 覆写兼容 OpenAI Python/JS SDK | ✅ 通过指向 Perplexity endpoint | ✅ 与 OpenAI、Anthropic、Groq 等 drop-in 集成 |
| **鉴权方式** | `x-api-key` header 或 `Authorization: Bearer` | `Authorization: Bearer`（`PERPLEXITY_API_KEY`） | `Authorization: Bearer tvly-YOUR_API_KEY` |
| **引用/结果格式** | 结果对象含 `url`、`title`、`author`、`publishedDate`、`text`、`highlights`、`summary` 等字段；Answer endpoint 返回 `answer` + `citations` 数组 | 响应含 `search_results` 字段（title、URL、发布日期）；旧版 `citations` 字段已废弃 | JSON 含 `query`、`answer`、`results`（title/url/content/score/raw_content/favicon/images）、`response_time`、`usage`、`request_id` |
| **结构化输出** | ✅ `outputSchema` 参数支持结构化 JSON 输出 | ✅ 所有用户均可使用结构化 JSON 输出 | ✅ 结果以结构化 JSON 返回 |
| **流式响应** | ✅ Answer 及 Chat Completions 支持 `stream` 参数 | ✅ 支持流式，含异步迭代器 | ❌ 未在官方文档中明确披露流式支持 |
| **神经/语义搜索** | ✅ 核心能力，基于 embeddings 的神经搜索；支持 neural/fast/auto/deep-lite/deep/deep-reasoning/instant 多种类型 | ✅ 通过 Embeddings API（`pplx-embed-v1` 系列）支持语义搜索；Sonar 模型含 web grounding | ⚠️ 主要为关键词+排名搜索；无独立向量搜索接口 |
| **爬取/抽取** | ✅ `/contents` 支持 livecrawl；`/findsimilar` 支持相似页面发现 | ✅ Agent API `fetch_url` 工具（$0.0005/次） | ✅ `/extract`、`/crawl`、`/map` 完整爬取链路 |
| **多步研究** | ✅ `/search?type=deep-reasoning`（原 `/research` 端点 2026-05-01 下线） | ✅ `sonar-deep-research` 模型 + Async API；支持 `reasoning_effort` 参数 | ✅ `/research` 端点（pro 模式 15–250 credits，mini 模式 4–110 credits） |
| **Web 监控** | ✅ `/monitors` 按计划频次追踪 web 事件并通过 webhook 推送 | ❌ 未提供 | ❌ 未提供 |
| **SLA** | Enterprise 计划提供 SLA（标准层未披露） | 未披露（AWS Marketplace 企业采购可用） | ✅ 99.99% uptime SLA（公开承诺） |
| **MCP 支持** | ❌ 未在文档中明确提及 | ✅ MCP Server，支持一键安装（Cursor、VS Code、Claude Desktop、Claude Code） | ❌ 未在文档中明确提及 |
| **主要集成** | OpenAI 兼容 Chat Completions；企业级自定义数据集 | n8n、OpenClaw、AWS Marketplace；支持 GPT/Claude/Gemini/Nemotron 等第三方模型（Agent API） | Make、n8n、Vercel AI SDK；Databricks MCP Marketplace、IBM WatsonX、JetBrains |
| **安全/隐私** | Enterprise：零数据留存（zero data retention） | 不使用用户数据训练模型 | 内置安全层（拦截 PII 泄露、prompt injection、恶意来源）；Enterprise `safe_search` 过滤 |
| **Base URL** | `https://api.exa.ai` | `https://api.perplexity.ai` | `https://api.tavily.com` |

---

## 推荐矩阵

| 场景 | 首选 | 次选 | 理由 |
|---|---|---|---|
| **最便宜（低频调用）** | Tavily | Exa | Tavily 免费 1,000 credits/月无需信用卡；Exa 亦提供 1,000 requests/月免费层 |
| **最稳定可靠（生产环境 SLA）** | Tavily | Exa Enterprise | Tavily 公开承诺 99.99% uptime SLA；Exa/Perplexity SLA 仅限 Enterprise |
| **功能最丰富（端点种类）** | Perplexity | Exa | Perplexity 覆盖 Sonar/Agent/Search/Embeddings/Async 多条产品线；Exa 拥有 Monitors 等独特端点 |
| **RAG 流水线原型** | Tavily | Exa | Tavily 专为 RAG 设计，credit 计费透明，SDK 成熟，与主流 LLM 框架 drop-in 集成 |
| **Agent 工具调用（多模型）** | Perplexity | Exa | Perplexity Agent API 支持 OpenAI/Anthropic/Google/xAI 模型 + web_search/fetch_url 工具 |
| **学术/语义检索** | Exa | Perplexity | Exa 基于 embeddings 的神经搜索针对语义理解优化；Perplexity 支持 `search_mode: academic` |
| **已知 URL 的结构化抽取** | Tavily | Exa | Tavily `/extract` 支持意图驱动抽取和 chunks_per_source 控制；Exa `/contents` 同样支持 livecrawl |
| **长耗时多步研究** | Perplexity | Exa | Perplexity `sonar-deep-research` 提供异步 API（7 天 TTL）+ `reasoning_effort` 控制；Exa `deep-reasoning` 类型亦可用 |
| **OpenAI SDK drop-in 替换** | Exa | Perplexity | Exa 提供 `model: exa` 的 Chat Completions 接口，`base_url` 覆写即可；Perplexity 同样兼容 |
| **最低延迟** | Tavily | Exa | Tavily 公开 p50 180ms 延迟承诺；Exa `instant` 类型搜索亦可达 180ms |

### 快速决策树

1. **如果**你正在快速验证 RAG 原型，且预算有限 → **选 Tavily**（免费层无需信用卡，SDK 与主流框架直接集成）。
2. **如果**你需要语义/向量级别的网页检索，或者关键词搜索无法满足查询理解需求 → **选 Exa**（embeddings-based neural search 是其核心竞争力）。
3. **如果**你需要同时调用多个 LLM 提供商（OpenAI/Anthropic/Google/xAI）并附带实时 web 搜索能力 → **选 Perplexity Agent API**。
4. **如果**你的应用要求明确的 SLA 保障且不在 Enterprise 预算范围内 → **选 Tavily**（唯一公开披露 99.99% uptime SLA 的标准层服务）。
5. **如果**你需要持续监控 web 上的特定事件或新闻并通过 webhook 推送 → **选 Exa**（`/monitors` 是三者中唯一提供此功能的端点）。

---

## 近六个月要点

### Exa

> Changelog：https://docs.exa.ai/changelog

- **2026-04-01**：发布 API 废弃公告，涉及多个遗留字段及 `/research` 端点。
- **2026-04-15**：`resolvedSearchType` 响应字段开始返回 `null`；`highlightScores` 响应字段返回 `null`；`startCrawlDate` / `endCrawlDate` 请求参数被静默忽略。
- **2026-05-01（即将生效）**：`/research` 端点正式下线，需迁移至 `/search?type=deep-reasoning`；`resolvedSearchType`、`highlightScores`、`startCrawlDate`/`endCrawlDate` 字段彻底移除。
- **持续**：`context` 响应字段已废弃，建议改用 `highlights` 或 `text`。
- **搜索类型扩展**：新增 `deep-lite`、`deep`、`deep-reasoning`、`instant` 搜索类型，延迟从 180ms 到 1s 可配置。

### Perplexity

> Changelog：https://docs.perplexity.ai/docs/resources/changelog

- **2026-04 月**：n8n 原生集成上线（覆盖 Chat Completions、Agent、Search、Embeddings）；OpenClaw 集成支持 Perplexity Search API 作为原生 web 搜索提供商；API Credits 通过 AWS Marketplace 上线（统一账单）；新增 `GET /v1/models` 端点（无需鉴权，返回 Agent API 全部模型列表）。
- **2026-03 月**：Agent API 规范端点更改为 `/v1/agent`（`/v1/responses` 继续作为别名）；新增 GPT-5.4、NVIDIA Nemotron、Claude Sonnet 4.6、Gemini 3.1 Pro Preview 支持；`google/gemini-2.5-flash`（3 月 20 日）、`google/gemini-2.5-pro` 及 `google/gemini-3-pro-preview`（4 月 1 日）相继废弃移除。
- **2026-02 月**：Agent API 正式 GA；Embeddings API 正式 GA，提供标准及上下文化 embeddings（`pplx-embed-v1` 系列）。
- **2025-12 月**：`sonar-reasoning` 于 12 月 15 日废弃，需迁移至 `sonar-reasoning-pro`；Media Classifier 上线（Sonar Pro）；Search API 新增 `max_tokens`、`last_updated_filter` 参数及 Vercel AI SDK 支持。
- **2025-11 月**：Pro Search for Sonar Pro 正式 GA，支持多步推理与实时思考流；MCP Server 上线，支持一键安装至 Cursor、VS Code、Claude Desktop、Claude Code；官方 Python/TypeScript SDK 发布。
- **2025-10 月**：Interactive Search API Playground 上线（无需 API key）；Search API 新增 `language_preference`、`search_domain_filter`、增强的日期时间过滤器。

### Tavily

> Changelog：https://docs.tavily.com/changelog

- **2026-03 月**：Enterprise API key 管理端点上线（`POST /generate-keys`、`POST /deactivate-keys`、`GET /key-info`，仅限 Enterprise 计划）。
- **2026-02 月**：Search 端点新增 `exact_match` 布尔参数，支持逐字短语匹配。
- **2026-01 月**：通过 `X-Project-ID` header 及 SDK `project_id` 参数引入项目追踪功能，便于多项目用量管理。
- **2025-12 月**：Search 新增 `fast` 和 `ultra-fast`（BETA）两种 `search_depth` 选项，均计 1 credit；Extract 新增意图驱动抽取（`query` 参数 + `chunks_per_source`）；`include_usage` 参数扩展至 Search、Extract、Crawl、Map 端点。
- **2025-11 月**：通过 `@tavily/ai-sdk` 包发布 Vercel AI SDK v5 集成；Crawl 和 Map 端点新增 `timeout` 参数（10–150 秒，默认 150 秒）。

### 三方评测简要汇总

> ⚠️ 以下 Exa versus 页面数据属于**厂商自行披露**，存在方法论偏差风险，仅供参考。

| 来源 | 结论摘要 |
|---|---|
| Exa 自家对比（https://exa.ai/versus/perplexity） | 500 条查询评测中 Exa 64.8% vs Perplexity 60.1% |
| Exa 自家对比（https://exa.ai/versus/tavily） | WebWalker 多跳基准 Exa 81% vs Tavily 71% |
| 独立 8 题小样本（websearchapi.ai） | 排名：Perplexity 第一 > Exa 第二 > Tavily 第三（样本量较小，仅供参考） |
| Composio 2026 榜单定位 | Exa = 语义搜索；Perplexity = 综合答案；Tavily = RAG 原型 |
| Firecrawl 