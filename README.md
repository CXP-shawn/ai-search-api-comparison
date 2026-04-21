# AI 搜索 API 对比：Exa vs Perplexity vs Tavily

> **编写日期**：2026-04-21
> **数据来源**：Exa、Perplexity、Tavily 各家官方文档与定价页面
> **报告立场**：独立技术对比，所有基准 / 厂商披露数据均注明出处

---

## 执行摘要

- **Exa 适合需要高精度语义检索与结构化抽取的开发者。** 其神经搜索引擎基于 embedding 相似度而非关键词匹配，`outputSchema` 可直接返回 JSON Schema 约束的结构化数据，适合构建知识图谱、竞品监控等数据密集型场景。
- **Perplexity 是唯一将大语言模型推理与实时搜索深度融合的选项。** Sonar 系列模型原生支持引用溯源，`sonar-deep-research` 可执行数十步自主研究并返回带引文的长报告，适合需要 AI 生成综合答案的产品。
- **Tavily 是 RAG 与 Agent 管道的最低摩擦选项。** 免费层无需信用卡、99.99% SLA、p50 延迟 180 ms，五个端点覆盖搜索到完整网站爬取，内置 PII / prompt injection 过滤开箱即用。
- **定价结构差异显著，横向比较需区分计量单位。** Exa 按请求计费（$7–$15/1K）、Perplexity 按 token + 请求双维度计费、Tavily 统一折算为 credit（$0.008/credit），三者不可直接换算，须结合实际用量估算 TCO。
- **SDK 生态与兼容层影响集成成本。** Perplexity 与 Exa 均提供 OpenAI Chat Completions 兼容层，可零改动接入现有 LLM 工具链；Tavily 提供专属 AI SDK 适配器，与 LangChain、LlamaIndex 等主流 Agent 框架深度集成。

**默认推荐**：优先语义精度与结构化输出选 **Exa**，优先 AI 推理答案与引文选 **Perplexity**，优先 RAG/Agent 快速落地与低成本起步选 **Tavily**。

---

## 侧向对比表

| 维度 | Exa | Perplexity | Tavily |
|---|---|---|---|
| 主站 / 文档 | [exa.ai](https://exa.ai) / [docs.exa.ai](https://docs.exa.ai) | [perplexity.ai](https://perplexity.ai) / [docs.perplexity.ai](https://docs.perplexity.ai) | [tavily.com](https://tavily.com) / [docs.tavily.com](https://docs.tavily.com) |
| 核心 endpoints | `/search` `/contents` `/findSimilar` `/answer` `/research` `/monitors` | `/chat/completions`（Sonar 系列）`/search`（Search API）Agent API | `/search` `/extract` `/crawl` `/map` `/research` |
| 定价模型 | 按请求计费（Search / Deep / Deep-Reasoning 分档） | 按 token + 按请求双维度计费 | 统一 credit 制（$0.008/credit） |
| 头部价格 | Search $7/1K、Deep $12/1K、Deep-Reasoning $15/1K | Sonar Pro 输入 $3/1M tokens + $6/1K requests；Sonar $1/1M tokens + $5/1K requests | 基础搜索 1 credit/次，即 $0.008/次 |
| 免费额度 | 1,000 requests/month | 无公开免费层（需付费订阅或企业洽谈） | 1,000 credits/month，无需信用卡 |
| 速率限制 | `/search` 10 QPS（默认） | 视模型与套餐，企业可协商 | Dev 100 RPM / Prod 1,000 RPM |
| SDK | `exa-py`、`exa-js`、OpenAI 兼容层 | OpenAI 兼容（直接复用 `openai` SDK） | `tavily-python`、`@tavily/core`、`@tavily/ai-sdk` |
| 鉴权方式 | API Key（`x-api-key` header） | API Key（`Authorization: Bearer` header） | API Key（`Authorization: Bearer` header / `X-Project-ID` 子项目隔离） |
| 引用 / 结果格式 | 返回带 URL、标题、摘要、全文的结构化结果列表 | 返回带 `citations` 数组的 LLM 答案文本 | 返回带 `url`、`title`、`content`、`score` 的结果列表 |
| 结构化输出 | ✅ `outputSchema`（JSON Schema 约束） | ❌ 原生不支持，需提示工程 | ❌ 原生不支持，需提示工程 |
| 流式输出 | ✅（`/answer`、`/research` 支持 SSE） | ✅（SSE，`stream: true`） | ❌ 当前不支持流式 |
| 神经 / 语义检索 | ✅ 核心能力，基于 embedding 向量相似度 | ⚠️ 依赖底层 LLM 理解，非独立向量检索层 | ⚠️ 关键词 + 相关性排序，非原生向量检索 |
| 爬取 / 抽取支持 | ✅ `/contents` 支持全文抽取 | ❌ 仅返回搜索摘要 | ✅ `/extract` `/crawl` `/map` 完整爬取链路 |
| 多步研究 | ✅ `/research`（自动多步） | ✅ `sonar-deep-research`（数十步自主研究） | ✅ `/research`（agentic 多步） |
| SLA | 未公开明确 SLA 数字 | 未公开明确 SLA 数字 | 99.99% 正常运行时间 |
| 主要集成 | LangChain、LlamaIndex、OpenAI Assistants | LangChain、OpenAI 生态、任意 OpenAI 兼容客户端 | LangChain、LlamaIndex、CrewAI、Vercel AI SDK、`@tavily/ai-sdk` |

---

## 每家 API 技术画像

### Exa

Exa 定位为「为 AI 而生的搜索引擎」，核心差异化在于其神经搜索技术：索引层基于 embedding 向量相似度而非传统 BM25 关键词匹配，因此对自然语言查询、语义模糊匹配与相似内容发现的支持远优于传统搜索 API。

**主要 endpoints**

| Endpoint | 功能 |
|---|---|
| `POST /search` | 神经语义搜索，返回 URL、标题、摘要，可选全文 |
| `POST /contents` | 按 URL 列表批量抽取页面正文 |
| `POST /findSimilar` | 给定 URL，返回语义相似网页 |
| `POST /answer` | 基于搜索结果生成带引文的 AI 答案，支持 SSE 流式 |
| `POST /research` | 自动多步研究，返回综合报告 |
| `POST /monitors` | 创建关键词 / 主题监控，定期推送新内容 |

**定价**（厂商披露，来源：[exa.ai/pricing](https://exa.ai/pricing)）

- Search：$7 / 1,000 requests
- Deep Search：$12 / 1,000 requests
- Deep-Reasoning Search：$15 / 1,000 requests
- Contents（页面抽取）：$1 / 1,000 pages
- Monitors：$15 / 1,000 triggers
- Answer：$5 / 1,000 requests

**免费额度**：1,000 requests/month，无需信用卡。

**速率限制**：`/search` 默认 10 QPS，企业套餐可协商提升。

**SDK 与集成**：官方维护 `exa-py`（Python）与 `exa-js`（TypeScript/Node.js），同时提供 OpenAI Chat Completions 兼容层，可直接替换 `openai` 客户端的 `base_url` 接入现有工具链。

**独特卖点**

- `outputSchema`：在请求体中传入 JSON Schema，搜索结果直接按 schema 约束返回结构化 JSON，无需额外解析层。
- 可配置延迟档位：`livecrawl` 参数支持 180 ms（缓存优先）到 1 s（强制实时爬取）的延迟 / 新鲜度权衡。
- 新闻监控（`/monitors`）：持续追踪指定主题，适合竞品情报、舆情监控场景。

详见 [docs/exa.md](./docs/exa.md)

---

### Perplexity

Perplexity 定位为「带引用的 AI 答案引擎 API」，核心差异在于将大语言模型推理与实时网络搜索深度融合于单一 API 调用，返回结果天然携带 `citations` 引文数组，适合需要 AI 综合答案且可溯源的产品场景。

**Sonar 系列模型**（厂商披露，来源：[docs.perplexity.ai](https://docs.perplexity.ai)）

| 模型 | 定位 |
|---|---|
| `sonar` | 轻量实时搜索，低延迟 |
| `sonar-pro` | 高质量搜索答案，支持更多引文 |
| `sonar-reasoning` | 集成推理链（Chain-of-Thought）与搜索 |
| `sonar-reasoning-pro` | 深度推理 + 高质量搜索 |
| `sonar-deep-research` | 自主多步研究，可执行数十步搜索并生成长报告 |

**定价**（厂商披露）

- Sonar：输入 $1 / 1M tokens，输出 $1 / 1M tokens，搜索请求 $5 / 1K requests
- Sonar Pro：输入 $3 / 1M tokens，输出 $15 / 1M tokens，搜索请求 $6 / 1K requests
- Search API（独立端点）：$5 / 1,000 queries

**Agent API**：支持多模型调度，适合构建自主 Agent 工作流。

**鉴权与兼容性**：完全兼容 OpenAI Chat Completions 接口规范，仅需替换 `base_url` 为 `https://api.perplexity.ai` 并更新 API Key，现有 `openai` SDK 代码无需修改。

**特色过滤器与功能**

- `search_domain_filter`：限定或排除特定域名（如仅检索学术来源、SEC EDGAR）。
- 文件附件：支持在请求中附带 PDF、DOC、DOCX、TXT、RTF 文件，模型可结合文件内容与网络搜索生成答案。
- 流式输出：`stream: true` 启用 SSE，适合实时对话界面。

详见 [docs/perplexity.md](./docs/perplexity.md)

---

### Tavily

Tavily 定位为「为 RAG 与 AI Agent 优化的搜索 API」，设计哲学是最小集成摩擦：免费层无需信用卡、单一 credit 计费单位、内置安全过滤，五个端点覆盖从单次搜索到完整网站爬取的全链路需求。

**五个核心 endpoints**

| Endpoint | 功能 |
|---|---|
| `POST /search` | 实时网络搜索，返回结构化结果列表，p50 延迟 180 ms |
| `POST /extract` | 按 URL 列表精准抽取页面正文与元数据 |
| `POST /crawl` | 递归爬取指定域名，支持深度与页面数限制 |
| `POST /map` | 返回指定域名的完整 URL 站点地图 |
| `POST /research` | 自主多步 agentic 研究，返回综合报告 |

**定价**（厂商披露，来源：[tavily.com/pricing](https://tavily.com/pricing)）

- 统一 credit 制：$0.008 / credit
- 免费层：1,000 credits/month，无需绑定信用卡

**SLA 与性能**：99.99% 正常运行时间（厂商披露）；`/search` p50 延迟 180 ms。

**速率限制**：Dev 套餐 100 RPM；Prod 套餐 1,000 RPM。

**SDK 与集成**

- `tavily-python`：Python 官方客户端
- `@tavily/core`：Node.js / TypeScript 官方客户端
- `@tavily/ai-sdk`：Vercel AI SDK 适配器，可直接作为 tool 注入 AI SDK 工作流
- 深度集成：LangChain、LlamaIndex、CrewAI

**独特功能**

- `auto_parameters`：自动推断最优搜索参数（深度、类别等），无需手动调优。
- `exact_match`：强制精确短语匹配，适合合规与引文校验场景。
- `X-Project-ID` header：支持多项目 API Key 隔离，便于多租户场景下的用量分账。
- 内置 PII 检测与 prompt injection 过滤：搜索结果在返回前自动过滤敏感信息与注入攻击内容，降低 Agent 管道安全风险。

详见 [docs/tavily.md](./docs/tavily.md)

## 推荐矩阵

| 场景 | 首选 | 次选 | 理由 |
|------|------|------|------|
| 最便宜的起步成本 | Tavily | Exa | Tavily 提供 1,000 免费 credits 无需绑卡，PAYGO 仅 $0.008/credit；Exa 免费层每月 1,000 req |
| 最稳定可靠（公开 SLA） | Tavily | Perplexity Sonar | Tavily 承诺 99.99% SLA、p50 延迟 180ms；Perplexity Sonar 可通过 AWS Marketplace 获得企业级支持 |
| 功能最丰富（端点 / 模型种类） | Perplexity Sonar | Exa | Perplexity 覆盖多 Sonar 模型、Agent API 多模型、Search、Embeddings、文件附件及 `academic`/`SEC` filter；Exa 提供 6 个独立端点 |
| RAG 流水线 | Tavily | Exa | Tavily 原生支持 `/extract`、`chunks_per_source`、`include_raw_content`，并与 LangChain/LlamaIndex 深度集成；Exa 提供 `/contents` + `outputSchema` |
| Agent 工具调用 | Tavily | Perplexity Sonar | Tavily 支持 MCP Server、Vercel AI SDK 及 Databricks MCP Marketplace；Perplexity Sonar 支持 MCP、n8n 与 OpenClaw |
| 学术 / 语义检索 | Exa | Perplexity Sonar | Exa 基于 embedding 神经检索并提供 research papers 自定义索引；Perplexity Sonar 支持 `search_mode=academic` filter |
| 已知 URL 的结构化抽取 | Tavily `/extract` | Exa `/contents` | Tavily 以 1 credit/5 URLs（basic）完成抽取并支持 query reranking；Exa 按内容类型计费 $1/1K pages |
| 长耗时多步研究 | Perplexity `sonar-deep-research` | Tavily `/research` | Perplexity 提供异步 API 与 `reasoning_effort` 参数；Tavily `/research` 支持 15–250 credits/req 的深度调节 |
| OpenAI SDK drop-in（最快接入） | Perplexity Sonar | Exa | Perplexity Sonar 完全兼容 Chat Completions 接口；Exa 同样支持 `base_url` override |
| 最低延迟（单次搜索） | Exa instant mode | Tavily `/search` fast/ultra-fast | Exa instant mode 延迟 <200ms；Tavily fast/ultra-fast 模式接近 180ms p50 |

### 快速决策树

- 若要对话式问答 + 引用 → 选 **Perplexity Sonar**，其生成式回答天然携带来源引用。
- 若要爬取 / 抽取任意 URL + 搜索 → 选 **Tavily**，`/extract` 端点可一步完成页面内容结构化。
- 若要语义 / 神经检索 + 结构化 JSON 输出 → 选 **Exa**，embedding 索引与 `outputSchema` 是其核心优势。
- 若对免费层无信用卡要求敏感 → 选 **Tavily**，1,000 免费 credits 无需绑卡即可上手。
- 若代码库已用 OpenAI SDK → 选 **Perplexity Sonar** 或 **Exa**，两者均支持 `base_url` override，零改造接入。

## 近六个月要点

### Exa

2026 年 4 月，Exa 发布重要 deprecation notice，涉及多个端点与字段的下线计划：

- **`/research` 端点下线**：将于 **2026-05-01** 正式停止服务，建议迁移至 `/search`（`type=deep-reasoning`）以获得等效功能。
- **`resolvedSearchType` 字段**：**2026-04-15 起**返回值强制变更为 `null`，**2026-05-01** 该字段将从响应中完全移除。
- **`highlightScores` 字段**：**2026-04-15 起**返回值强制变更为 `null`，请勿在业务逻辑中依赖该字段。
- **`startCrawlDate` / `endCrawlDate` 请求参数**：**2026-04-15 起**传入后将被服务端忽略，不再产生任何过滤效果。

> 详见官方 [changelog](https://docs.exa.ai/changelog)

---

### Perplexity

- **2026-04**：推出与 [n8n](https://n8n.io) 的原生集成，覆盖 Chat Completions、Agent、Search、Embeddings 四类节点，可在 n8n 工作流中直接调用 Sonar API，无需额外适配层。
- **2026-04**：完成与 OpenClaw 的集成，支持在 OpenClaw 平台内调用 Perplexity 搜索与推理能力。
- **2026-04**：正式上架 AWS Marketplace，支持通过 AWS 账单合并结算 API Credits，适合已在 AWS 生态内的企业用户。
- **2026**：`search_results` 字段转为 GA（General Availability），作为标准化字段替代旧版 legacy `citations` 字段，返回结构化的来源信息。
- **2026**：新增 academic mode，请求时传入 `search_mode: academic` 即可将检索范围收窄至学术论文与学术数据库。
- **2026**：新增 SEC filings filter，传入 `search_domain: sec` 可将搜索范围限定于 SEC 公开申报文件，适合金融合规场景。
- **2025**：发布 `sonar-deep-research` 异步 API，支持长耗时深度研究任务，并引入 `reasoning_effort` 参数以权衡推理深度与延迟。
- **2025**：发布 MCP Server，支持一键安装至 Cursor、VS Code 及 Claude Desktop，开发者可直接在 IDE 内调用 Sonar 搜索能力。

> 详见官方 [changelog](https://docs.perplexity.ai/docs/resources/changelog)

---

### Tavily

- **2026-03**：推出 Enterprise API key management 功能，新增 `generate-keys`、`deactivate-keys`、`key-info` 三个管理端点，支持企业级多团队 API key 的统一管控。
- **2026-02**：新增 `exact_match` 参数，支持精确短语匹配，可在搜索请求中指定必须逐字出现的关键词，提升结果精准度。
- **2026-01**：引入 `X-Project-ID` HTTP header 与 SDK 中的 `project_id` 参数，实现项目级用量隔离，便于多项目成本分摊与审计。
- **2025-12**：`/extract` 端点支持 Intent-Based Extraction，新增 `query` 字段，服务端将依据查询意图对提取的文本 chunk 进行重排序，返回与意图最相关的片段。
- **2025-12**：新增 `chunks_per_source` 参数，允许开发者控制每个来源页面返回的最大文本块数量，便于精细调控 RAG 管线中的上下文长度。
- **2025**：发布 `auto_parameters` 自动配参模式，Tavily 服务端可根据查询内容自动推断并设置最优搜索参数，降低调参成本。
- **2025**：与 Databricks MCP Marketplace 达成合作，Tavily 搜索工具可直接通过 Databricks MCP 市场安装并集成至数据与 AI 工作流中。

> 详见官方 [changelog](https://docs.tavily.com/changelog)

---

### 三方评测简要汇总

| 来源 | 结论 | 备注 |
|---|---|---|
| Exa vs Perplexity — 500 条查询评测（[exa.ai/versus/perplexity](https://exa.ai/versus/perplexity)） | Exa 64.8% vs Perplexity 60.1%，Exa 胜出 | 厂商自行披露，存在选题偏向风险 |
| Exa vs Tavily — WebWalker 多跳推理评测（[exa.ai/versus/tavily](https://exa.ai/versus/tavily)） | Exa 81% vs Tavily 71%，Exa 胜出 | 厂商自行披露，评测场景聚焦多跳检索 |
| websearchapi.ai 独立 8 题横评 | Perplexity 第一 > Exa 第二 > Tavily 第三 | 样本量极小（8 题），结论仅供参考 |
| Composio / Firecrawl 2026 榜单 | Exa ≈ 语义检索；Perplexity ≈ 综合答案生成；Tavily ≈ RAG 流水线集成 | 定性分类，非量化排名 |

## 参考资料

### Exa

- [https://exa.ai](https://exa.ai)
- [https://docs.exa.ai](https://docs.exa.ai)
- [https://exa.ai/pricing](https://exa.ai/pricing)
- [https://docs.exa.ai/reference/getting-started](https://docs.exa.ai/reference/getting-started)
- [https://docs.exa.ai/reference/search](https://docs.exa.ai/reference/search)
- [https://docs.exa.ai/reference/get-contents](https://docs.exa.ai/reference/get-contents)
- [https://docs.exa.ai/reference/answer](https://docs.exa.ai/reference/answer)
- [https://docs.exa.ai/reference/find-similar-links](https://docs.exa.ai/reference/find-similar-links)
- [https://docs.exa.ai/reference/research/create-a-task](https://docs.exa.ai/reference/research/create-a-task)
- [https://docs.exa.ai/reference/rate-limits](https://docs.exa.ai/reference/rate-limits)
- [https://docs.exa.ai/reference/quickstart](https://docs.exa.ai/reference/quickstart)
- [https://docs.exa.ai/changelog](https://docs.exa.ai/changelog)

### Perplexity

- [https://www.perplexity.ai](https://www.perplexity.ai)
- [https://docs.perplexity.ai](https://docs.perplexity.ai)
- [https://docs.perplexity.ai/docs/getting-started/pricing](https://docs.perplexity.ai/docs/getting-started/pricing)
- [https://docs.perplexity.ai/docs/getting-started/models](https://docs.perplexity.ai/docs/getting-started/models)
- [https://docs.perplexity.ai/docs/getting-started/rate-limits](https://docs.perplexity.ai/docs/getting-started/rate-limits)
- [https://docs.perplexity.ai/docs/sonar/quickstart](https://docs.perplexity.ai/docs/sonar/quickstart)
- [https://docs.perplexity.ai/docs/sonar/models](https://docs.perplexity.ai/docs/sonar/models)
- [https://docs.perplexity.ai/docs/search/quickstart](https://docs.perplexity.ai/docs/search/quickstart)
- [https://docs.perplexity.ai/docs/sdk/overview](https://docs.perplexity.ai/docs/sdk/overview)
- [https://docs.perplexity.ai/api-reference/chat-completions-post](https://docs.perplexity.ai/api-reference/chat-completions-post)
- [https://docs.perplexity.ai/docs/resources/changelog](https://docs.perplexity.ai/docs/resources/changelog)

### Tavily

- [https://tavily.com](https://tavily.com)
- [https://docs.tavily.com](https://docs.tavily.com)
- [https://docs.tavily.com/documentation/api-credits](https://docs.tavily.com/documentation/api-credits)
- [https://docs.tavily.com/documentation/rate-limits](https://docs.tavily.com/documentation/rate-limits)
- [https://docs.tavily.com/documentation/api-reference/introduction](https://docs.tavily.com/documentation/api-reference/introduction)
- [https://docs.tavily.com/documentation/api-reference/endpoint/search](https://docs.tavily.com/documentation/api-reference/endpoint/search)
- [https://docs.tavily.com/documentation/api-reference/endpoint/extract](https://docs.tavily.com/documentation/api-reference/endpoint/extract)
- [https://docs.tavily.com/documentation/api-reference/endpoint/crawl](https://docs.tavily.com/documentation/api-reference/endpoint/crawl)
- [https://docs.tavily.com/documentation/api-reference/endpoint/map](https://docs.tavily.com/documentation/api-reference/endpoint/map)
- [https://docs.tavily.com/changelog](https://docs.tavily.com/changelog)

### 第三方

- [Exa vs Perplexity（厂商披露）](https://exa.ai/versus/perplexity)
- [Exa vs Tavily（厂商披露）](https://exa.ai/versus/tavily)
- [Firecrawl — Exa alternatives](https://www.firecrawl.dev/blog/exa-alternatives)
- [Composio — 9 top AI search engine tools](https://composio.dev/content/9-top-ai-search-engine-tools)
- [websearchapi.ai 比较](https://websearchapi.ai/blog/compare-tavily-google-search-exa-perplexity)
- [versustool.com Exa vs Perplexity](https://versustool.com/exa-vs-perplexity-ai-api)
- [Bright Data Exa alternatives](https://brightdata.com/blog/ai/exa-alternatives)
- [AIMultiple agentic search](https://aimultiple.com/agentic-search)
- [Newscatcher Web Search API benchmark Q1 2026](https://www.newscatcherapi.com/blog-posts/web-search-api-benchmark-q1-2026)

> 定价、速率限制与功能可能变更。使用前请核对各家官方页面。