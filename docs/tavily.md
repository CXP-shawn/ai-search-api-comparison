# Tavily API 技术文档

## 概述

Tavily 是一款面向 AI 应用与 RAG 管道设计的实时网络搜索 API，主打低延迟、高可靠的检索能力。平台提供搜索、内容抽取、网站映射、爬取与深度研究等多类 endpoint，并内置安全过滤、智能缓存与内容验证层。据官方数据，每月处理超 1 亿次请求，承诺 99.99% 的 uptime SLA，p50 搜索延迟为 180ms。

Composio 2026 榜单将 Tavily 定位为 **RAG 原型**首选工具，适合快速构建以检索为核心的 AI 应用。

---

## Base URL

```
https://api.tavily.com
```

> **注意**：Production API key 需要开通付费计划（Paid Plan）或启用 PAYGO 功能后才能使用。

---

## Endpoint 列表

| Endpoint | 方法 | 功能简介 |
|---|---|---|
| `/search` | POST | 实时网络搜索，返回排序结果，可附带 LLM 生成的摘要答案 |
| `/extract` | POST | 从指定 URL 列表中抽取并解析页面内容 |
| `/map` | POST | 映射网站结构，发现站内链接页面 |
| `/crawl` | POST | 从起始 URL 遍历网站图，批量发现并抽取多页内容 |
| `/research` | POST | 综合深度研究，按研究深度动态消耗 credits |
| `/usage` | GET | 实时查询 API 用量与计划限额 |
| `/generate-keys` | POST | 生成 API key（企业版专属） |
| `/deactivate-keys` | POST | 停用 API key（企业版专属） |
| `/key-info` | GET | 查询 API key 信息（企业版专属） |

### `/search` — 实时网络搜索

**主要参数**：

- `query`：搜索查询字符串
- `search_depth`：搜索深度，可选 `basic`（1 credit）/ `advanced`（2 credits）/ `fast`（1 credit，BETA）/ `ultra-fast`（1 credit，BETA）
- `max_results`：最大返回结果数
- `topic`：主题类型，`general` / `news` / `finance`
- `time_range`、`start_date`、`end_date`：时间范围过滤
- `include_answer`：是否包含 LLM 生成答案
- `include_raw_content`：是否返回原始内容
- `include_images`、`include_image_descriptions`：图片相关
- `include_favicon`：是否返回站点图标
- `include_domains`、`exclude_domains`：域名过滤
- `country`：国家/地区过滤
- `auto_parameters`：根据查询意图自动配置搜索参数
- `exact_match`：精确短语匹配（2026 年 2 月新增）
- `include_usage`：在响应中包含 credit 用量信息
- `chunks_per_source`：每个来源返回的内容块数
- `safe_search`：安全过滤（企业版专属）

### `/extract` — 内容抽取

**主要参数**：

- `urls`：目标 URL 列表
- `extract_depth`：`basic`（1 credit / 5 URLs）/ `advanced`（2 credits / 5 URLs）
- `query`：用于结果重排序的查询语句
- `chunks_per_source`：每个来源的内容块数
- `format`：输出格式，`markdown` 或 `text`
- `timeout`：超时时间
- `include_favicon`：是否包含站点图标
- `include_usage`：是否包含用量

### `/map` — 网站结构映射

**主要参数**：

- `url`：起始 URL
- `instructions`：爬取指令（有指令时 2 credits / 10 页，无指令时 1 credit / 10 页）
- `timeout`：超时时间（10–150 秒，默认 150 秒）
- `include_usage`：是否包含用量

### `/crawl` — 网站爬取

**主要参数**：

- `url`：起始 URL
- `extract_depth`：抽取深度
- `instructions`：爬取指令
- `timeout`：超时时间（10–150 秒，默认 150 秒）
- `include_usage`：是否包含用量

> **计费说明**：Crawl 费用 = Mapping 费用 + Extraction 费用之和

### `/research` — 深度研究

**主要参数**：

- `model`：`pro`（最少 15 credits，最多 250 credits / 请求）或 `mini`（最少 4 credits，最多 110 credits / 请求）

### `/usage` — 用量查询

通过 `Authorization` header 传入 API key 即可查询当前 API 用量与计划限额。

---

## 定价

### Credits 体系说明

Tavily 采用 **基于 credit 的定价模型**，不同操作消耗不同数量的 credits：

| 操作 | Credits 消耗 |
|---|---|
| Search（basic / fast / ultra-fast） | 1 credit / 请求 |
| Search（advanced） | 2 credits / 请求 |
| Extract（basic） | 1 credit / 5 个成功 URL |
| Extract（advanced） | 2 credits / 5 个成功 URL |
| Map（无指令） | 1 credit / 10 页 |
| Map（有指令） | 2 credits / 10 页 |
| Crawl | Mapping 费用 + Extraction 费用 |
| Research（model=pro） | 最少 15，最多 250 credits / 请求 |
| Research（model=mini） | 最少 4，最多 110 credits / 请求 |

### 套餐价格

| 套餐 | Credits / 月 | 月费 | 单价 |
|---|---|---|---|
| Researcher（免费） | 1,000 | $0 | — |
| Project | 4,000 | $30 | $0.0075 / credit |
| Bootstrap | 15,000 | $100 | $0.0067 / credit |
| Startup | 38,000 | $220 | $0.0058 / credit |
| Growth | 100,000 | $500 | $0.005 / credit |
| Enterprise | 定制 | 定制 | 定制 |

**按需付费（Pay-as-you-go）**：$0.008 / credit

---

## 速率限制

Tavily 按开发（Development）和生产（Production）环境分别设置限制，超出限制时返回 HTTP 429 并附带 `retry-after` header。

| Endpoint | Development | Production |
|---|---|---|
| `/search`（默认） | 100 RPM | 1,000 RPM |
| `/crawl` | 100 RPM | 100 RPM |
| `/research` | 20 RPM | 20 RPM |
| `/usage` | 10 requests / 10 min | 10 requests / 10 min |

> 企业版支持自定义速率限制。

---

## SDK

| SDK | 安装方式 |
|---|---|
| Python SDK | 官方支持，提供快速入门指南与 SDK 参考文档 |
| JavaScript SDK | 官方支持，提供快速入门指南与 SDK 参考文档 |
| Vercel AI SDK v5 | `@tavily/ai-sdk` 包集成（2025 年 11 月发布） |

**第三方集成**：
- **无代码平台**：Make、n8n（原生集成）
- **LLM 提供商**：OpenAI、Anthropic、Groq（drop-in 集成）
- **MCP**：Databricks MCP Marketplace 合作伙伴
- **企业合作**：IBM WatsonX、JetBrains

---

## 鉴权

通过 `Authorization` header 传入 Bearer token：

```http
Authorization: Bearer tvly-YOUR_API_KEY
```

企业版额外支持通过 API key 管理端点（`/generate-keys`、`/deactivate-keys`、`/key-info`）进行 key 的批量生成与管理。

---

## 结果 / 引用格式

所有搜索响应均返回 JSON 结构，包含以下字段：

| 字段 | 说明 |
|---|---|
| `query` | 原始查询字符串 |
| `answer` | LLM 生成的摘要答案（可选，由 `include_answer` 控制） |
| `results` | 结果数组（见下表） |
| `response_time` | 响应耗时 |
| `usage` | 本次请求消耗的 credits |
| `request_id` | 请求唯一标识 |

**`results` 数组中每条结果包含**：

| 字段 | 说明 |
|---|---|
| `title` | 页面标题 |
| `url` | 页面 URL |
| `content` | 页面内容摘要 |
| `score` | 相关性得分 |
| `raw_content` | 原始内容（由 `include_raw_content` 控制） |
| `favicon` | 站点图标（由 `include_favicon` 控制） |
| `images` | 图片列表（由 `include_images` 控制） |

---

## 结构化输出

Tavily 暂不提供类似 Exa `outputSchema` 的原生 JSON Schema 结构化输出参数。搜索结果以标准 JSON 格式返回，`include_usage` 参数可在响应中附加详细的 credit 消耗明细。

如需结构化数据抽取，可结合 `/extract` endpoint 配合 `query` 重排序参数使用，或在应用层自行处理返回的 `content` / `raw_content` 字段。

---

## 独特功能

- **Credit 计费模型**：免费层无需信用卡，按实际使用量灵活计费
- **内置安全与隐私层**：自动拦截 PII 泄露、提示词注入攻击和恶意来源（`safe_search` 为企业版功能）
- **智能缓存与索引**：保证规模化部署下的低延迟可预测性
- **p50 延迟 180ms**：`/search` endpoint 的中位响应时延
- **99.99% uptime SLA**：企业级可用性保障
- **项目追踪**：通过 `X-Project-ID` header 或 SDK `project_id` 参数跨项目组织用量（2026 年 1 月新增）
- **`auto_parameters` 模式**：根据查询意图自动配置搜索参数
- **`exact_match` 精确匹配**：支持逐字短语搜索（2026 年 2 月新增）
- **意图感知抽取**：`/extract` 的 `query` 参数用于结果重排序，`chunks_per_source` 控制内容块粒度
- **`fast` / `ultra-fast` 搜索深度**：超低延迟搜索选项（BETA，均为 1 credit）
- **企业 API key 管理**：`/generate-keys`、`/deactivate-keys`、`/key-info` 端点
- **Certification Program**：通过认证可获得 API credits 奖励
- **Vercel AI SDK v5 集成**：`@tavily/ai-sdk` 包
- **MCP Marketplace**：与 Databricks 合作上架

---

## 免费额度

- **每月 1,000 credits**，无需绑定信用卡
- 对应约 1,000 次 basic 搜索或 500 次 advanced 搜索

---

## 优势

1. **RAG 原型开发首选**：结果格式简洁，与主流 LLM 框架（LangChain、LlamaIndex 等）集成门槛极低
2. **定价透明可预测**：Credit 体系清晰，免费层无需信用卡，适合快速验证
3. **高可靠性承诺**：99.99% uptime SLA，p50 延迟 180ms，适合生产环境
4. **内置安全层**：自动防护 PII 泄露、提示词注入，减少应用层安全负担
5. **多样化 endpoint**：Search、Extract、Map、Crawl、Research 覆盖完整的数据采集链路
6. **生态集成广泛**：与 IBM WatsonX、Databricks、JetBrains、Vercel 等有官方合作
7. **项目级用量管理**：`X-Project-ID` 支持多项目隔离追踪

---

## 劣势

1. **语义搜索能力有限**：不具备 Exa 的 embeddings-based 神经搜索，对语义相似性查询的支持较弱
2. **无 AI 深度推理**：研究能力不及 Perplexity sonar-deep-research，缺乏 Chain of Thought 推理
3. **Production key 需付费计划**：开发者需升级套餐才能在生产环境使用
4. **Research endpoint 成本不确定**：动态 credit 消耗（pro: 15–250 credits）使单次请求费用难以精确预估
5. **无原生结构化 JSON Schema 输出**：不支持类似 Exa `outputSchema` 的参数化结构化输出
6. **第三方评测排名靠后**：独立 8 题小样本测试（websearchapi.ai）中排名第三；Exa vs Tavily 的 WebWalker 多跳基准测试中 Tavily 71% 低于 Exa 81%（注：后者为 Exa 自家测试数据）

---

## 近期更新

| 时间 | 更新内容 |
|---|---|
| 2026 年 3 月 | 企业版 API key 管理端点上线：`POST /generate-keys`、`POST /deactivate-keys`、`GET /key-info` |
| 2026 年 2 月 | `/search` 新增 `exact_match` 布尔参数，支持精确短语匹配 |
| 2026 年 1 月 | 引入项目追踪功能：`X-Project-ID` header 与 SDK `project_id` 参数 |
| 2025 年 12 月 | `/search` 新增 `fast` 和 `ultra-fast`（BETA）搜索深度选项，均为 1 credit |
| 2025 年 12 月 | 意图感知抽取：`/extract` 新增 `query` 重排序参数与 `chunks_per_source` 参数；同步支持 `/crawl` |
| 2025 年 12 月 | `include_usage` 参数上线，适用于 Search、Extract、Crawl、Map endpoint |
| 2025 年 11 月 | Vercel AI SDK v5 集成发布（`@tavily/ai-sdk` 包） |
| 2025 年 11 月 | Crawl 和 Map endpoint 新增 `timeout` 参数（范围 10–150 秒，默认 150 秒） |

---

## 备注

- **融资情况**：Tavily 已完成 2500 万美元 A 轮融资
- **规模**：每月处理超 1 亿次请求，已爬取数十亿页面
- **企业版功能**：除标准功能外，企业版额外提供 `safe_search` 过滤、API key 批量管理、自定义定价与用量折扣
- **Production key 限制**：生产环境 API key 需有效付费计划或开启 PAYGO，否则只能使用开发环境
- **Crawl 计费**：费用为 Mapping 成本与 Extraction 成本之和，深度爬取前建议先估算总费用
- **Research 动态计费**：pro 模型单次请求 credit 消耗范围为 15–250，mini 模型为 4–110，实际消耗取决于研究深度

---

## 官方文档链接

- **官网**：[https://tavily.com](https://tavily.com)
- **文档首页**：[https://docs.tavily.com](https://docs.tavily.com)
- **定价与 Credits**：[https://docs.tavily.com/documentation/api-credits](https://docs.tavily.com/documentation/api-credits)
- **速率限制**：[https://docs.tavily.com/documentation/rate-limits](https://docs.tavily.com/documentation/rate-limits)
- **API 参考 - 简介**：[https://docs.tavily.com/documentation/api-reference/introduction](https://docs.tavily.com/documentation/api-reference/introduction)
- **API 参考 - Search endpoint**：[https://docs.tavily.com/documentation/api-reference/endpoint/search](https://docs.tavily.com/documentation/api-reference/endpoint/search)
- **更新日志**：[https://docs.tavily.com/changelog](https://docs.tavily.com/changelog)
