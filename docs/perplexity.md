# Perplexity API 技术画像

## 概述

Perplexity 提供以 **Sonar** 系列模型为核心的 Web 增强型 AI API，将实时网络搜索与大语言模型生成能力深度融合，返回带引用来源的自然语言答案。除 Sonar API 外，还提供 Agent API（接入第三方模型）、Search API（纯搜索结果）、Embeddings API 以及异步深度研究接口，适用于问答、RAG、智能体等多类场景。

- **定位**：综合答案型搜索 AI，搜索与生成一体化
- **核心优势**：OpenAI Chat Completions API 兼容、内置多步推理（Pro Search）、深度研究异步任务、丰富的过滤与上下文控制
- **适用场景**：实时问答、摘要生成、学术研究、财务分析、多模态查询

---

## Base URL

```
https://api.perplexity.ai
```

---

## Endpoint 列表

| Endpoint | 路径 | 描述 | 主要参数 |
|---|---|---|---|
| Sonar API (Chat Completions) | `POST /v1/sonar` | 使用 Sonar 系列模型进行 Web 增强型 AI 问答，支持流式输出与工具调用 | `model`, `messages`, `stream`, `web_search_options`（`search_context_size`, `search_type`）, `search_domain`, `search_mode`, `reasoning_effort` |
| Agent API | `POST /v1/agent` | 接入 OpenAI、Anthropic、Google、xAI 等第三方模型，支持 `web_search` 和 `fetch_url` 工具 | `model`, `messages`, `tools`（`web_search`, `fetch_url`）, `stream` |
| Search API | — | 返回原始排名网页搜索结果，无 LLM 处理 | `query`（支持多查询）, `language_preference`, `search_domain_filter`, 日期过滤, `max_tokens`, `last_updated_filter` |
| Embeddings API | — | 生成高质量文本向量，用于语义搜索和 RAG 管道 | `model`, 输入文本 |
| Models List | `GET /v1/models` | 列出所有 Agent API 可用模型（OpenAI 兼容格式），无需鉴权 | 无需参数 |
| Async Sonar API | `POST /v1/async/sonar`（创建）<br>`GET /v1/async/sonar`（列表）<br>`GET /v1/async/sonar/{request_id}`（查询） | 异步任务提交与结果获取，专为 Sonar Deep Research 设计，结果保留 7 天（TTL） | `request_id` |

> `/v1/responses` 是 `/v1/agent` 的别名，继续有效，兼容 OpenAI Responses API 风格。

---

## 模型列表

### Sonar API 模型

| 模型 | 定位 | 特点 |
|---|---|---|
| **sonar** | 轻量、经济型搜索模型 | 适合快速事实查询、话题摘要、产品比较、实时事件；Web grounding 内置 |
| **sonar-pro** | 高级搜索模型 | 支持复杂查询与多轮追问；支持 Pro Search（多步自动化工具调用）和 Media Classifier |
| **sonar-reasoning-pro** | 精准推理模型 | 带有链式思维（Chain of Thought）；适合需要逐步推理的复杂分析场景 |
| **sonar-deep-research** | 专家级深度研究模型 | 执行大规模穷举搜索并生成综合报告；支持异步 API 和 `reasoning_effort` 参数 |

### Agent API 模型（示例，截至 2026 年 3 月）

Agent API 支持来自 OpenAI、Anthropic、Google、NVIDIA 等提供商的第三方模型，包括 GPT-5.4、Claude Sonnet 4.6、Gemini 3.1 Pro Preview、NVIDIA Nemotron 等。可通过 `GET /v1/models` 获取完整最新列表。

> **已弃用**：`sonar-reasoning` 已于 2025 年 12 月 15 日移除，请迁移至 `sonar-reasoning-pro`。

---

## 定价

### Sonar API（按 Token + 请求数计费）

| 模型 | 输入 Token | 输出 Token | 请求费（低/中/高上下文） |
|---|---|---|---|
| sonar | $1/1M | $1/1M | $5 / $8 / $12 per 1K requests |
| sonar-pro | $3/1M | $15/1M | $6 / $10 / $14 per 1K requests |
| sonar-pro（Pro Search） | $3/1M | $15/1M | $14 / $18 / $22 per 1K requests |
| sonar-reasoning-pro | $2/1M | $8/1M | $6 / $10 / $14 per 1K requests |
| sonar-deep-research | $2/1M 输入 / $8/1M 输出 | $2/1M 引用 Token<br>$3/1M 推理 Token | $5/1K search queries |

### Search API

- $5.00 per 1K requests，不额外收取 Token 费用

### Agent API

- Token 按第三方提供商直接定价，**不加价**
- `web_search` 工具：$0.005/次调用
- `fetch_url` 工具：$0.0005/次调用

### Embeddings API

| 模型 | 维度 | 价格 |
|---|---|---|
| pplx-embed-v1-0.6b | 1024 | $0.004/1M tokens |
| pplx-embed-v1-4b | 2560 | $0.03/1M tokens |
| pplx-embed-context-v1-0.6b | 1024 | $0.008/1M tokens |
| pplx-embed-context-v1-4b | 2560 | $0.05/1M tokens |

> **说明**：Citation Token 费用仅针对 sonar-deep-research 收取。搜索上下文大小（low/medium/high）影响结果深度及请求费档位。

---

## 速率限制

- 默认速率限制按使用量分级（Usage Tier）
- 自 2024 年 11 月起，所有用户 Sonar 在线模型默认速率限制提升至 **50 requests/min（RPM）**
- 详细分级数值请参阅官方 [Rate Limits & Usage Tiers](https://docs.perplexity.ai/docs/getting-started/rate-limits) 文档
- 所有用户均可访问全部 API 功能（自 2025 年 4 月起取消功能按级别门控）

---

## SDK

| 语言 | 安装方式 | 备注 |
|---|---|---|
| Python（3.8+） | `pip install perplexityai` | 官方 SDK，2025 年 10 月发布 |
| TypeScript/Node.js | `npm install @perplexity-ai/perplexity_ai` | 官方 SDK，2025 年 10 月发布 |
| OpenAI Python SDK | 将 `base_url` 指向 Perplexity endpoint | 兼容方式 |
| OpenAI TypeScript SDK | 同上 | 兼容方式 |

此外，支持 n8n 原生集成（覆盖 Chat Completions、Agent、Search、Embeddings 四大 API），以及 MCP Server 一键安装（Cursor、VS Code、Claude Desktop、Claude Code）。

---

## 鉴权

使用 **Bearer Token** 方式，通过环境变量 `PERPLEXITY_API_KEY` 或在请求时显式传入 API Key：

```http
Authorization: Bearer <PERPLEXITY_API_KEY>
```

- 支持 API Key 轮换机制
- `GET /v1/models` 接口**无需鉴权**即可访问

---

## 结果 / 引用格式

- 响应体包含 `search_results` 字段（自 2025 年 5 月起），每条结果含 **title**、**URL**、**发布日期**
- 旧版 `citations` 字段已废弃并移除，请使用 `search_results`
- 响应 `usage` 字段提供详细的成本分解（Token 数、请求类型等）
- 支持流式输出（Streaming）及异步迭代器

---

## 结构化输出

- 所有用户均可使用 **Structured JSON Outputs**（自 2025 年 4 月功能门控取消后）
- 可通过标准 JSON schema 约束输出格式

---

## 独特功能

- **Web 增强型 AI 答案（Sonar API）**：搜索与生成一体化，内置实时网页搜索
- **OpenAI Chat Completions API 兼容**：无缝迁移，改动最小
- **搜索上下文大小控制**（low / medium / high）：灵活权衡结果深度与成本
- **Pro Search**：Sonar Pro 支持多步自动化工具调用，实时推理流式输出
- **Sonar Deep Research**：异步 API，支持 `reasoning_effort` 参数，执行大规模穷举研究
- **文件附件支持**：PDF、DOC、DOCX、TXT、RTF
- **图片上传**：多模态查询支持
- **Media Classifier**（Sonar Pro）：自动视觉内容检测与引入
- **学术过滤**（`search_mode: academic`）：聚焦学术来源
- **SEC 文件过滤**（`search_domain: sec`）：专项财务研究
- **地理位置过滤 & 日期范围过滤**
- **MCP Server**：一键安装，支持 Cursor、VS Code、Claude Desktop
- **n8n 原生集成**：全 API 覆盖
- **AWS Marketplace**：支持合并账单的企业采购
- **不使用用户数据训练模型**

---

## 免费额度

官方文档中未说明免费额度信息。可通过 [Interactive Search API Playground](https://docs.perplexity.ai)（无需 API Key）进行功能体验。

---

## 优势

1. **答案质量高**：搜索与生成深度融合，引用来源清晰，适合需要综合分析的场景
2. **OpenAI 兼容**：与 OpenAI SDK 完全兼容，迁移成本低
3. **模型梯队完整**：从轻量 sonar 到重型 sonar-deep-research，覆盖不同复杂度需求
4. **功能全面开放**：自 2025 年 4 月起所有用户均可访问全部 API 功能，无等级门控
5. **多模态与多源支持**：文件、图片、学术、SEC 等多类过滤器
6. **Agent API 零加价**：接入第三方模型按原始价格计费
7. **隐私保护**：不使用用户数据训练模型
8. **生态丰富**：MCP Server、n8n、AWS Marketplace、Vercel 等主流工具集成

---

## 劣势

1. **免费额度不明确**：官方未明示免费试用配额，新用户上手门槛相对较高
2. **定价结构复杂**：Token 费 + 请求费 + 上下文档位三维叠加，成本预估较繁琐
3. **速率限制分级细节不透明**：具体分级数值需查阅独立文档，资料中未完整披露
4. **搜索结果控制粒度有限**：相比 Exa 的 embeddings-based 向量检索，对检索逻辑的定制化程度较低
5. **近期模型废弃较频繁**：sonar-reasoning、多个 Gemini 模型短期内陆续移除，集成稳定性需关注
6. **不提供独立爬取/抓取 endpoint**：无类似 Tavily `/extract`、`/crawl` 的网页内容直接抓取能力

---

## 近期更新

| 时间 | 更新内容 |
|---|---|
| 2026 年 4 月 | n8n 原生集成发布，覆盖 Chat Completions、Agent、Search、Embeddings 全部 API |
| 2026 年 4 月 | OpenClaw 集成，支持将 Perplexity Search API 作为原生网络搜索提供商 |
| 2026 年 4 月 | API Credits 通过 AWS Marketplace 开放，支持合并账单 |
| 2026 年 4 月 | 新增 `GET /v1/models` 接口，列出所有 Agent API 模型（OpenAI 兼容格式，无需鉴权） |
| 2026 年 3 月 | Agent API 新增支持 GPT-5.4、NVIDIA Nemotron、Claude Sonnet 4.6、Gemini 3.1 Pro Preview |
| 2026 年 3 月 | 模型废弃：`google/gemini-2.5-flash`（3 月 20 日）、`google/gemini-2.5-pro` 和 `google/gemini-3-pro-preview`（4 月 1 日）移除 |
| 2026 年 3 月 | Agent API 正式路径变更为 `/v1/agent`，`/v1/responses` 继续作为别名有效 |
| 2026 年 2 月 | Agent API 正式 GA |
| 2026 年 2 月 | Embeddings API 正式 GA，标准向量与上下文向量两种类型均可用 |
| 2025 年 12 月 | `sonar-reasoning` 于 12 月 15 日废弃移除，请迁移至 `sonar-reasoning-pro` |
| 2025 年 12 月 | Media Classifier 上线（Sonar Pro），自动视觉内容检测与引入 |
| 2025 年 12 月 | Search API 增强：新增 `max_tokens` 参数、`last_updated_filter` 支持、Vercel AI SDK 支持 |
| 2025 年 11 月 | Pro Search for Sonar Pro 正式 GA，支持多步推理、实时推理流和自动分类 |
| 2025 年 11 月 | MCP Server 一键安装，支持 Cursor、VS Code、Claude Desktop、Claude Code |
| 2025 年 10 月 | Python 和 TypeScript 官方 SDK 正式发布 |
| 2025 年 10 月 | Interactive Search API Playground 上线，无需 API Key 即可体验 |
| 2025 年 10 月 | Search API 新增 `language_preference`、`search_domain_filter`、增强日期/时间过滤 |

---

## 备注

- **用户数据政策**：Perplexity 不使用用户 API 数据训练模型
- **功能门控**：自 2025 年 4 月起，所有 API 功能对全体用户开放，不再按消费层级限制，但速率限制仍按用量分级
- **引用字段迁移**：旧版 `citations` 字段已废弃并移除，请使用 `search_results`（自 2025 年 5 月起）
- **搜索引用 Token 费用**：Citation Token 费用**仅**针对 sonar-deep-research 收取
- **异步任务 TTL**：Async Sonar API 任务结果保留 **7 天**
- **企业采购**：可通过 AWS Marketplace 进行 API Credits 采购，支持合并账单

---

## 官方文档链接

- 官网：[https://www.perplexity.ai](https://www.perplexity.ai)
- API 文档主页：[https://docs.perplexity.ai](https://docs.perplexity.ai)
- 定价：[https://docs.perplexity.ai/docs/getting-started/pricing](https://docs.perplexity.ai/docs/getting-started/pricing)
- 模型列表：[https://docs.perplexity.ai/docs/getting-started/models](https://docs.perplexity.ai/docs/getting-started/models)
- 速率限制：[https://docs.perplexity.ai/docs/getting-started/rate-limits](https://docs.perplexity.ai/docs/getting-started/rate-limits)
- Sonar 快速入门：[https://docs.perplexity.ai/docs/sonar/quickstart](https://docs.perplexity.ai/docs/sonar/quickstart)
- Sonar 模型：[https://docs.perplexity.ai/docs/sonar/models](https://docs.perplexity.ai/docs/sonar/models)
- SDK 概览：[https://docs.perplexity.ai/docs/sdk/overview](https://docs.perplexity.ai/docs/sdk/overview)
- 更新日志：[https://docs.perplexity.ai/docs/resources/changelog](https://docs.perplexity.ai/docs/resources/changelog)
