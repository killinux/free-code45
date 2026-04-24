# LLM 使用全景分析

本文档整理了代码库中所有使用 LLM 的地方，按功能分类标注关键文件、函数和逻辑。

---

## 1. API 调用层 — 实际发请求的地方

### 主调用入口

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/services/api/claude.ts` | `queryModelWithStreaming()` | 主流式调用，使用 `anthropic.messages.create()` with streaming |
| `src/services/api/claude.ts` | `queryModelWithoutStreaming()` | 非流式回退方案，单次 request/response |
| `src/services/api/claude.ts` | `executeNonStreamingRequest()` | 统一处理流式与非流式执行 |

### SDK 客户端创建

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/services/api/client.ts` | `getAnthropicClient()` | 根据 provider 类型创建对应 SDK 客户端实例 |

### Token 计数 API 调用

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/services/tokenEstimation.ts` | `countMessagesTokensWithAPI()` | 使用 `anthropic.beta.messages.countTokens()` 精确计数 |
| `src/services/tokenEstimation.ts` | `countTokensViaHaikuFallback()` | 用 Haiku 模型做 token 估算的回退方案 |
| `src/services/tokenEstimation.ts` | `countTokensWithBedrock()` | Bedrock 专用 token 计数 |

---

## 2. 模型配置与选择

### 模型选择逻辑

文件：`src/utils/model/model.ts`

| 函数 | 说明 |
|------|------|
| `getMainLoopModel()` | 主模型选择，优先级：`/model` 命令 > `--model` flag > `ANTHROPIC_MODEL` 环境变量 > 用户设置 > 内置默认 |
| `getDefaultMainLoopModel()` | 根据订阅类型返回默认模型：Max/Team Premium → Opus 4.6，Codex → GPT-5.3，其他 → Sonnet 4.6 |
| `getDefaultOpusModel()` | 默认 `claude-opus-4-6` |
| `getDefaultSonnetModel()` | 第一方 `claude-sonnet-4-6`，第三方 `claude-sonnet-4-5` |
| `getDefaultHaikuModel()` | 默认 `claude-haiku-4-5` |

### 模型 ID 格式

- **Anthropic 第一方**: `claude-{family}-{version}-{date}`（如 `claude-opus-4-6-20250514`）
- **AWS Bedrock**: AWS ARN 或 inference profile 名称
- **Google Vertex**: GCP 资源路径
- **Azure Foundry**: 自定义部署 ID
- **OpenAI**: `gpt-5.4`, `gpt-5.3-codex`

### 模型成本定义

文件：`src/utils/modelCost.ts`

| 常量 | 单价 (per Mtok) |
|------|----------------|
| `COST_TIER_5_25` | Opus 4.5/4.6: $5 input, $25 output |
| `COST_TIER_30_150` | Opus 4.6 fast mode: $30 input, $150 output |
| `COST_HAIKU_45` | Haiku 4.5: $1 input, $5 output |
| 另有 cache read/write 成本层级 | |

---

## 3. 消息管道 — 查询生命周期

### 核心调度

| 文件 | 关键函数/类 | 说明 |
|------|-----------|------|
| `src/QueryEngine.ts` | `QueryEngine.submitMessage()` | **核心调度器**：管理消息历史、工具调用循环、流式事件分发、权限检查 |
| `src/query.ts` | 查询状态机 | 工具执行、重试、自动压缩（auto-compaction）逻辑 |

### 消息构造

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/utils/messages.ts` | `createUserMessage()` | 将用户输入包装为 API 消息格式（含附件） |
| `src/utils/messages.ts` | `normalizeMessagesForAPI()` | 清洗消息数据，确保符合 API 规范 |

---

## 4. Prompt 构建与上下文注入

### System Prompt 模板

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/constants/prompts.ts` | `getSystemPrompt()` | 完整 system prompt 模板，包含工具描述，使用 feature flag 控制条件段落 |

### Prompt 组装

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/utils/api.ts` | `buildSystemPromptBlocks()` | 带缓存标记（cache_control）的 prompt 块组装 |
| `src/utils/api.ts` | `toolToAPISchema()` | 将 Tool 对象转换为 Anthropic API 工具 schema |
| `src/utils/queryContext.ts` | `fetchSystemPromptParts()` | 拼接 system prompt + user context + system context |

### 上下文注入

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/context.ts` | `prependUserContext()` | 注入 cwd、git 状态等前置上下文 |
| `src/context.ts` | `appendSystemContext()` | 追加系统级上下文 |
| `src/memdir/memdir.ts` | `loadMemoryPrompt()` | 注入记忆系统 prompt（当 `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 设置时） |

### QueryEngine 中的 Prompt 拼接

`QueryEngine.ts` 中最终拼接逻辑：

```
asSystemPrompt([
  ...defaultSystemPrompt (或 customSystemPrompt),
  ...memoryMechanicsPrompt (仅当 CLAUDE_COWORK_MEMORY_PATH_OVERRIDE 设置时),
  ...appendSystemPrompt (调用方指定的追加内容)
])
```

---

## 5. 多 Provider 抽象

### Provider 检测

文件：`src/utils/model/providers.ts`

函数 `getAPIProvider()` 返回值：`'firstParty' | 'bedrock' | 'vertex' | 'foundry' | 'openai'`

### 各 Provider 配置

| Provider | 触发条件 | SDK / 适配 |
|----------|---------|-----------|
| **Anthropic (直连)** | `ANTHROPIC_API_KEY` | `@anthropic-ai/sdk` |
| **AWS Bedrock** | `CLAUDE_CODE_USE_BEDROCK` | 动态导入 `@anthropic-ai/bedrock-sdk`，支持 AWS credential 刷新、region 配置、inference profile |
| **Google Vertex** | `CLAUDE_CODE_USE_VERTEX` | `@anthropic-ai/vertex-sdk` + `google-auth-library`，支持 region 路由、metadata server |
| **Azure Foundry** | `CLAUDE_CODE_USE_FOUNDRY` | `@anthropic-ai/foundry-sdk` + `@azure/identity`，Azure AD token provider |
| **OpenAI** | `OPENAI_API_KEY` | `src/services/api/codex-fetch-adapter.ts`（将 Anthropic SDK 调用转译为 OpenAI API 格式） |

### 客户端创建入口

文件：`src/services/api/client.ts`

`getAnthropicClient()` 根据 `getAPIProvider()` 的返回值，创建对应 provider 的 SDK 客户端实例。

---

## 6. 流式响应处理

### SSE 事件处理

文件：`src/services/api/claude.ts`

| 事件类型 | 说明 |
|---------|------|
| `message_start` | 初始化消息，包含 model、usage 起始数据 |
| `content_block_start` | 标记文本/tool_use/thinking 内容块开始 |
| `content_block_delta` | 增量文本/thinking/tool input 流式数据 |
| `content_block_stop` | 当前内容块结束 |
| `message_delta` | 最终消息元数据（stop_reason、usage 更新） |
| `message_stop` | 流结束信号 |

### 流超时处理

- 流式调用有超时机制
- 超时后自动回退到非流式调用 (`queryModelWithoutStreaming`)

---

## 7. 费用与 Token 追踪

### 费用计算

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/cost-tracker.ts` | `addToTotalSessionCost()` | 累计会话总 USD 成本 |
| `src/cost-tracker.ts` | `calculateUSDCost()` | 根据 token 数 × 模型单价计算费用 |
| `src/utils/modelCost.ts` | 成本常量 | 各模型的 input/output/cache 单价定义 |

### Token 估算

| 文件 | 说明 |
|------|------|
| `src/services/tokenEstimation.ts` | API 精确计数 + 启发式估算（4 bytes/token, JSON 为 2 bytes/token）|

### Usage 聚合

| 文件 | 关键函数 | 说明 |
|------|---------|------|
| `src/services/api/claude.ts` | `updateUsage()` / `accumulateUsage()` | 从流式事件中聚合 usage 数据 |
| `src/services/api/logging.ts` | `logAPISuccessAndDuration()` | API 调用指标日志 |

### 追踪维度

- `input_tokens` — 输入 token 数
- `output_tokens` — 输出 token 数
- `cache_read` — 缓存命中读取的 token 数
- `cache_write` — 缓存写入的 token 数
- `cache_creation` — 缓存创建的 token 数

---

## 8. 横切关注点

### 重试逻辑

文件：`src/services/api/withRetry.ts`

- `withRetry()` 包装 API 调用，指数退避重试
- 处理：认证错误、429 限流、5xx 服务端错误
- 重试时保留上下文状态

### Prompt Caching

文件：`src/services/api/claude.ts`

- 缓存标记：`{ type: 'ephemeral', ttl: '1h', scope?: 'global' }`
- 1h TTL 通过 GrowthBook feature gate 控制启用
- 监控缓存失效（cache break detection）

### Extended Thinking

文件：`src/utils/thinking.ts`

- 三种模式：`disabled` / `enabled` / `adaptive`
- 最大 budget tokens 根据模型动态设定
- Thinking 块通过 `content_block_delta` 事件流式传输

### Tool Use (函数调用)

| 文件 | 说明 |
|------|------|
| `src/Tool.ts` | 工具定义、`findToolByName()` 路由 |
| `src/tools/` 目录 | 各工具实现 |
| `src/utils/api.ts` | `toolToAPISchema()` 转换工具 schema |

- 支持 `eager_input_streaming`（工具输入流式传输）
- Tool result 通过 `tool_use_id` 与 `tool_result` 配对

### 遥测分析

文件：`src/services/analytics/`

- 上报维度：model、betas enabled、thinking type、fast mode、effort level
- 平台：GrowthBook / Statsig

---

## 9. 关键文件索引

| 文件 | 职责 |
|------|------|
| `src/QueryEngine.ts` | 主查询调度，消息生命周期，SDK 接口 |
| `src/query.ts` | 查询状态机，工具执行，压缩逻辑 |
| `src/services/api/claude.ts` | 流式/非流式 API 执行，事件处理，usage 追踪 |
| `src/services/api/client.ts` | SDK 客户端创建，多 provider 支持 |
| `src/services/api/codex-fetch-adapter.ts` | OpenAI 协议转译适配器 |
| `src/services/api/withRetry.ts` | API 重试策略 |
| `src/services/tokenEstimation.ts` | Token 计数（API + 启发式） |
| `src/utils/model/model.ts` | 模型选择与默认值 |
| `src/utils/model/providers.ts` | Provider 检测逻辑 |
| `src/utils/modelCost.ts` | 模型成本定义 |
| `src/utils/api.ts` | System prompt 组装，工具 schema 构建，缓存控制 |
| `src/utils/queryContext.ts` | System prompt 部件装配 |
| `src/utils/messages.ts` | 消息构造与清洗 |
| `src/utils/thinking.ts` | Extended Thinking 配置 |
| `src/constants/prompts.ts` | System prompt 模板 |
| `src/context.ts` | 上下文注入（cwd、git 等） |
| `src/memdir/memdir.ts` | 记忆系统 prompt 加载 |
| `src/cost-tracker.ts` | USD 成本计算与累计 |
| `src/services/api/logging.ts` | API 调用日志 |
| `src/Tool.ts` | 工具定义与路由 |

---

## 10. 复杂度评估与后续分析建议

以下区域复杂度较高，建议分专题深入分析：

1. **QueryEngine + query.ts 查询状态机** — 包含消息历史管理、自动压缩、工具执行循环、权限检查，是整个 LLM 交互的核心
2. **多 Provider 适配层** — 特别是 OpenAI fetch adapter 做了完整的协议转译（Anthropic → OpenAI API），逻辑复杂
3. **Prompt Caching 策略** — 缓存标记的插入位置、TTL 管理、cache break 检测跨多个文件协作
4. **Token 估算与成本追踪** — 多种估算方式的选择逻辑、按模型分别追踪的聚合机制
