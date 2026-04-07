# Claude Code 架构设计文档

## 一、项目概览

| 指标 | 数据 |
|------|------|
| 版本 | v2.1.87 |
| 语言 | TypeScript / TSX |
| 运行时 | Bun |
| UI 框架 | Ink (React for Terminal) |
| 代码规模 | ~1,914 文件, ~206,037 行 |
| 依赖数 | 114 个 |

---

## 二、整体架构分层

```
┌─────────────────────────────────────────────────────┐
│                   Entrypoints Layer                  │
│  cli.tsx | mcp.ts | sdk/ | agentSdkTypes.ts         │
├─────────────────────────────────────────────────────┤
│                    Screen Layer (Ink/React)          │
│  REPL.tsx | Doctor.tsx | ResumeConversation.tsx      │
├─────────────────────────────────────────────────────┤
│              Query Engine (核心调度)                  │
│  QueryEngine.ts → query.ts → tokenBudget/transitions│
├──────────────┬──────────────┬───────────────────────┤
│  Tool System │ Command Sys  │ Skill / Plugin System │
│  40+ Tools   │ ~90 Commands │ ~/.claude/skills/     │
├──────────────┴──────────────┴───────────────────────┤
│                   Services Layer                     │
│  API | MCP | OAuth | Compact | Analytics | LSP ...  │
├─────────────────────────────────────────────────────┤
│              State / Memory / Permissions            │
│  AppState (Zustand) | memdir/ | permissions/        │
├─────────────────────────────────────────────────────┤
│              Infrastructure / Utils                  │
│  Bridge(IDE) | Tasks | Voice | Native-TS | Vendor   │
└─────────────────────────────────────────────────────┘
```

---

## 三、核心子系统详解

### 1. 入口系统 (`src/entrypoints/`)

`cli.tsx` 是主入口，采用**快速路径(Fast Path)** 设计模式：

```
cli.tsx 启动
  ├─ --version        → 零模块加载，直接输出 (极速)
  ├─ --dump-system-prompt → 提取系统提示词 (eval 用)
  ├─ --daemon-worker  → Daemon 子进程
  ├─ claude ps/logs   → 后台会话管理
  ├─ claude bridge    → IDE 远程桥接
  └─ 默认             → 进入 REPL 交互主循环
```

**技术亮点：** 针对不同子命令做"零加载"短路返回，避免加载整个 React 渲染树，CLI 冷启动极快。

---

### 2. 查询引擎 (`QueryEngine.ts` + `query.ts`)

这是整个系统的**心脏**，负责多轮对话的状态机管理。

```
用户输入
  ↓ QueryEngine.submitMessage()
  ├─ 解析引用 (@file, #agent)
  ├─ 加载记忆附件 (CLAUDE.md)
  ↓
  query() 主循环
  ├─ 构建 System Prompt (getSystemPrompt, 914行)
  ├─ 消息标准化 (normalizeMessagesForAPI)
  ├─ 执行 pre-sampling hooks
  ├─ 流式调用 Anthropic API
  ├─ 解析工具调用 / 思考块
  ├─ runTools() → 工具执行
  ├─ 执行 post-sampling hooks
  ├─ Token 预算追踪
  └─ 循环直到 stop_reason === 'end_turn'
  ↓
  持久化转录 → 返回渲染
```

**技术亮点：**

- **Token Budget 管理** (`tokenBudget.ts`)：实时追踪 token 消耗，接近上限时触发自动压缩
- **状态机转换** (`transitions.ts`)：明确的 Terminal/Continue 状态，优雅处理多轮工具调用
- **Prompt Cache 利用**：System Prompt 固定在 API 的 prompt cache 中，后续轮次推理加速

---

### 3. 工具系统 (`Tool.ts` + `tools.ts` + `src/tools/`)

这是最核心的扩展点，采用 **Builder 模式 + 统一接口** 设计。

#### 统一工具接口 (Tool.ts, 793 行)

```typescript
interface Tool {
  // 标识
  name, aliases, searchHint

  // 执行
  call(args, context, canUseTool, parentMessage, onProgress)

  // LLM 提示
  description(), prompt()
  inputSchema (Zod), outputSchema (Zod)

  // 元数据 / 安全
  isEnabled(), isReadOnly(), isDestructive()
  isConcurrencySafe()
  checkPermissions()

  // 渲染
  renderToolUseMessage()         // 工具调用时显示
  renderToolResultMessage()      // 结果显示
  renderToolUseProgressMessage() // 进度显示

  // 工具搜索优化
  shouldDefer?, alwaysLoad?
}
```

#### 工具注册池 (tools.ts, 390 行)

```
getAllBaseTools()  →  40+ 内置工具全集
      ↓
getTools(permissionContext)  →  按 deny rules 过滤
      ↓
assembleToolPool()  →  合并 MCP 外部工具，去重
      ↓
最终工具池 → 注入 System Prompt
```

#### 40+ 内置工具分类

| 类别 | 工具 |
|------|------|
| 文件 I/O | FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool |
| 执行 | BashTool (500+ 行), PowerShellTool, REPLTool |
| Agent | AgentTool (233k), SendMessageTool, TeamCreateTool |
| 任务 | TaskCreate/Get/Update/List/Output/Stop |
| 搜索 | WebSearchTool, WebFetchTool, ToolSearchTool |
| 规划 | EnterPlanModeTool, ExitPlanModeTool, VerifyPlanExecutionTool |
| MCP | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool |
| 其他 | SkillTool, NotebookEditTool, SleepTool, ScheduleCronTool |

**技术亮点：**

- **buildTool() 工厂函数**：所有工具通过统一工厂创建，自动填充安全默认值，类型完全安全
- **工具延迟加载 (Tool Deferral)**：不是所有工具都塞进 prompt，通过 `shouldDefer` 标记按需加载，`ToolSearchTool` 让模型主动搜索需要的工具，大幅节省 token
- **流式进度回调**：工具执行过程中通过 `onProgress()` 实时推送进度，BashTool 可以流式显示命令输出

---

### 4. 权限系统 (四层防护)

```
Layer 1: Tool 级别     → Tool.checkPermissions() 内部逻辑
Layer 2: 规则匹配      → settings.json 中的 glob 模式
          bash: ["git *", "npm *"]
          file: ["/src/**"]
Layer 3: 用户交互      → React 权限对话框 (PermissionRequest 组件)
Layer 4: Auto-Mode 分类器 → 安全分类器检查命令模式
```

**技术亮点：**

- **声明式权限规则**：通过 glob pattern 配置，不需要硬编码
- **分层降级**：先走规则匹配，匹配不到才弹对话框，减少用户干扰
- 12 个专用权限对话框组件，每种操作类型有定制化的 UI

---

### 5. MCP 协议集成 (`src/services/mcp/`, 300k+ 行)

MCP (Model Context Protocol) 是让 Claude Code 能调用外部工具服务的关键。

```
MCP Server Config (stdio/SSE/HTTP/WebSocket/SDK)
  ↓ useManageMCPConnections() (React Hook)
  ↓ MCPClient 建立连接
  ↓ 服务端广播 tools + resources
  ↓ assembleToolPool() 合并入工具池
  ↓ MCPTool.call() → client.callTool(server, tool, input)
```

**技术亮点：**

- **多传输层支持**：stdio / SSE / HTTP / WebSocket / SDK 五种方式
- **OAuth 集成** (`auth.ts`, 89k)：MCP 服务可以走 OAuth 认证流程
- **无缝合并**：MCP 外部工具和内置工具共用同一个 Tool 接口，对 LLM 透明

---

### 6. 多 Agent 系统 (`AgentTool/` + `coordinator/` + `tasks/`)

这是系统最复杂的部分之一，支持 Agent 的生成、隔离、通信。

```
用户调用 AgentTool
  ├─ 加载 Agent 定义 (~/.claude/agents/*.md)
  ├─ 选择隔离模式:
  │   ├─ worktree → Git worktree 隔离副本
  │   ├─ remote   → 远程计算环境 (CCR)
  │   └─ default  → 同目录执行
  ├─ 前台: runAgent() → 新 QueryEngine 循环 → 等待完成
  └─ 后台: LocalAgentTask → 异步执行 → 进度推送
```

**任务类型体系：**

- `local_bash` — Shell 命令
- `local_agent` — 本地后台 Agent
- `remote_agent` — 远程 Agent
- `in_process_teammate` — 进程内多 Agent 协作
- `dream` — 主动式后台任务

**技术亮点：**

- **Prompt Cache Fork** (`FORK_SUBAGENT` feature)：子 Agent 可以复用父 Agent 的 prompt cache，避免重复计算系统提示词的 KV cache
- **Coordinator 模式** (`coordinatorMode.ts`)：多 Agent 协调器，管理 worker 上下文过滤
- **Git Worktree 隔离**：每个 Agent 在独立的 git worktree 中工作，互不干扰

---

### 7. 上下文压缩系统 (`src/services/compact/`)

随着对话变长，token 会超限。压缩系统是保持长对话可用的关键。

```
策略 1: Compact Boundary  → 注入压缩标记，截断旧消息
策略 2: Tool Use Summary  → 多工具序列替换为摘要
策略 3: Snip-Based        → 省略旧消息，保留近期 (HISTORY_SNIP)
策略 4: Reactive Compact  → 接近 token 上限时自动触发
```

**技术亮点：**

- **Microcompact Cache** (`cachedMCConfig.ts`)：缓存压缩配置，避免重复计算
- **80% 阈值触发**：上下文超过 token limit 的 80% 时自动压缩
- **对用户透明**：压缩过程会显示提示但不中断工作流

---

### 8. 记忆系统 (`src/memdir/`)

```
~/.claude/CLAUDE.md (项目级)
  ├─ 项目指令、约定、偏好
  ├─ 自动加载条件:
  │   ├─ token 预算充足 (< 200k context)
  │   ├─ 关键词相关性匹配
  │   └─ 在时效窗口内
  └─ 注入到 System Prompt

Team Memory: ~/.claude/teams/{team}/CLAUDE.md
Session Memory: per-session 临时记忆
```

**技术亮点：**

- **相关性预取** (`startRelevantMemoryPrefetch()`)：异步预加载相关记忆，不阻塞主流程
- **团队记忆同步** (`teamMemorySync`)：多人协作场景下共享项目记忆

---

### 9. IDE 桥接 (`src/bridge/`, 300k+ 行)

```
Claude CLI ←→ Bridge Server ←→ IDE Extension
           WebSocket / SSE
           JSON-RPC 协议
           JWT 认证 + 设备信任
```

**技术亮点：**

- **双向消息协议** (`replBridge.ts`, 100k)：REPL 和 IDE 实时双向同步
- **安全体系**：JWT 签名 + 设备指纹 + 共享密钥三重保护
- **配置热更新** (`pollConfig.ts`)：配置变更自动轮询生效

---

### 10. 构建系统 (`scripts/build.ts`)

```typescript
// 54 个编译时 Feature Flag
KAIROS, VOICE_MODE, COORDINATOR_MODE, ULTRAPLAN,
BRIDGE_MODE, BG_SESSIONS, FORK_SUBAGENT, TORCH, ...

// 构建方式
bun run build           → ./cli (标准构建)
bun run build:dev       → ./cli-dev (开发构建)
bun run build:dev:full  → ./cli-dev (全部实验特性)
bun run compile         → ./dist/cli (编译优化)
```

**技术亮点：**

- **编译时死代码消除**：`feature('MY_FLAG')` 在构建时求值，未启用的特性整个模块树被移除，零运行时开销
- **分级构建**：标准/开发/全特性三档，外部发布版精确控制暴露的功能
- **Bun 原生 ESM + 字节码**：ESM 格式 + minify + bytecode 编译，启动极快

---

## 四、关键技术亮点总结

### 1. 编译时 Feature Flag 系统

不同于运行时 feature toggle，这里用 Bun bundler 在编译阶段直接剔除未启用特性的代码路径。54 个 flag 精确控制功能暴露，确保外部构建不包含内部实验特性。**零运行时开销**。

### 2. 工具延迟加载 (Tool Deferral)

不把 40+ 工具的 schema 全部塞进 System Prompt，而是标记 `shouldDefer`，让 LLM 通过 `ToolSearchTool` 按需搜索。一个典型的 token 预算优化，可节省数千 token。

### 3. Prompt Cache Fork

子 Agent 复用父 Agent 已渲染的系统提示词 cache key，避免在 API 侧重新计算 KV cache。对多 Agent 场景下的推理延迟有显著优化。

### 4. 流式工具执行 (StreamingToolExecutor)

工具执行不是"黑盒等结果"，而是通过 `onProgress()` 回调实时推送进度。BashTool 可以流式显示命令输出，FileEditTool 可以显示 diff 进度。**用户体验远优于传统等待模式**。

### 5. 四层权限模型

工具自身逻辑 → 声明式规则匹配 → 用户交互对话框 → 安全分类器，层层递进。既保证安全性，又最小化对用户的打扰。

### 6. 智能上下文压缩

四种压缩策略自动管理上下文窗口：边界截断、工具摘要、历史裁剪、响应式触发。80% 阈值自动触发，对用户透明，让长对话不会"失忆"。

### 7. Git Worktree Agent 隔离

每个子 Agent 可以在独立的 git worktree 中工作，修改互不影响。完成后自动清理或保留变更。**真正的并行开发隔离**。

### 8. Ink React 终端渲染

用 React 组件模型构建终端 UI，权限对话框、进度条、diff 显示全部是 React 组件。比传统 ncurses 更易维护和组合，同时支持完整的键盘/鼠标事件处理。

### 9. MCP 协议的多传输层适配

一个统一的 MCP client 抽象支持 stdio / SSE / HTTP / WebSocket / SDK 五种传输方式，外部工具服务无论用什么方式暴露，都能无缝接入工具池。

### 10. QueryEngine 与 UI 解耦

QueryEngine 是纯状态机，不依赖 React。这使得同一套查询引擎既能驱动终端 REPL，也能作为 headless SDK 被程序调用，还能在多 Agent 场景中被克隆。

---

## 五、数据流全景

```
User Input
  ↓
REPL.tsx (Ink React)
  ├─ 解析引用 (@file, #agent)
  ├─ 附加记忆 (CLAUDE.md)
  └─ QueryEngine.submitMessage()
       ↓
     query() 主循环
       ├─ System Prompt 组装 (914 行逻辑)
       ├─ 消息标准化
       ├─ Anthropic API 流式调用 (withRetry)
       ├─ 工具调用检测
       │    ↓
       │  权限检查 → 工具执行 → 进度推送
       │    ↓
       │  结果注入消息队列
       ├─ Token 预算检查 → 可能触发压缩
       ├─ Hook 执行 (pre/post)
       └─ 循环 or 终止
            ↓
       消息渲染 → 转录持久化 → 等待下一轮
```

---

## 六、工具执行流程

```
Tool detected in API response
  ↓ Extract ToolUseBlock
  ├─ Validate input schema (Zod)
  ├─ Permission check (checkPermissions)
  ├─ User dialog if needed
  └─ Call tool.call(input, context, canUseTool, parentMessage, onProgress)

Tool.call() async
  ├─ Execute operation (file read, bash command, etc.)
  ├─ Emit progress events via onProgress()
  ├─ Return ToolResult {data, mcpMeta?, newMessages?}
  └─ Throw on error

StreamingToolExecutor
  ├─ Collect progress updates
  ├─ Persist to transcript
  ├─ Render progress UI
  └─ Map result to ToolResultBlockParam

Query loop
  ├─ Queue tool_result message
  ├─ Continue with next API turn
  └─ Repeat until stop_reason === 'end_turn'
```

---

## 七、Agent 生成流程

```
用户调用 Agent tool
  ↓ AgentTool.call(input)
  ├─ 加载 agent 定义 (agents/*.md/json)
  ├─ 检查 model 权限
  ├─ 准备 forked messages (FORK_SUBAGENT)
  ├─ 选择隔离模式:
  │  ├─ worktree → createAgentWorktree()
  │  ├─ remote → checkRemoteAgentEligibility()
  │  └─ default → 同目录
  ├─ 后台模式 (run_in_background: true):
  │  ├─ 创建 LocalAgentTask
  │  ├─ registerAsyncAgent()
  │  └─ 返回 {status: 'async_launched'}
  └─ 前台模式:
     ├─ runAgent() (新 QueryEngine)
     ├─ 等待完成
     └─ 返回 {status: 'completed', ...result}
```

---

## 八、System Prompt 组装层次

```
1. 基础指令 (文件操作、工具使用规范)
2. 工具目录 (name, description, inputSchema)
3. 上下文 (cwd, git info, environment)
4. 记忆 (CLAUDE.md 内容)
5. 特性专属段落 (tool search, thinking 等)
6. 用户自定义覆盖 (appendSystemPrompt, customSystemPrompt)
```

---

## 九、关键目录索引

| 模块 | 路径 | 说明 |
|------|------|------|
| 入口 | `src/entrypoints/cli.tsx` | CLI 主入口 |
| 查询引擎 | `src/QueryEngine.ts` | 会话生命周期 |
| 查询循环 | `src/query.ts` | 主查询逻辑 |
| 工具接口 | `src/Tool.ts` | 统一工具定义 (793行) |
| 工具注册 | `src/tools.ts` | 工具池组装 (390行) |
| 工具实现 | `src/tools/` | 47 个工具目录 |
| 命令注册 | `src/commands.ts` | ~90 命令 (500+行) |
| 屏幕 | `src/screens/REPL.tsx` | 主交互 UI |
| 组件 | `src/components/` | 200+ UI 组件 |
| 状态 | `src/state/AppState.tsx` | 全局状态类型 |
| MCP | `src/services/mcp/` | MCP 协议 (300k+行) |
| Agent | `src/tools/AgentTool/` | 多 Agent 系统 |
| 记忆 | `src/memdir/` | 记忆管理 |
| 桥接 | `src/bridge/` | IDE 集成 (300k+行) |
| 权限 | `src/utils/permissions/` | 权限引擎 |
| 压缩 | `src/services/compact/` | 上下文压缩 |
| Prompt | `src/constants/prompts.ts` | 系统提示词 (914行) |
| 构建 | `scripts/build.ts` | 构建脚本 (200行) |

---

## 十、设计原则总结

1. **模块化 + 统一接口**：所有工具、命令、技能都遵循统一接口，通过注册表管理
2. **编译时优化**：Feature Flag 在构建时消除死代码，而非运行时判断
3. **分层权限**：安全不是一刀切，而是工具级 → 规则级 → 用户级 → 分类器级逐层递进
4. **UI 与逻辑解耦**：QueryEngine 不依赖 React，支持 CLI / SDK / 多 Agent 多种使用方式
5. **协议标准化**：MCP 协议统一外部工具接入，无论传输层如何变化
6. **Token 精打细算**：延迟加载工具、智能压缩、prompt cache 复用，多维度优化 token 使用
7. **隔离与并行**：Git worktree 级别的 Agent 隔离，支持真正的并行开发
8. **渐进式复杂度**：从简单 Bash 到多 Agent 协调，功能按需启用，不增加基础复杂度
