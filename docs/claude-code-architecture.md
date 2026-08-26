# Claude Code 源码架构深度解析

> 基于本仓库恢复源码的静态分析，按照“入口层 → 运行时层 → 核心引擎 → 工具与能力 → 基础设施”五层模型，解释 Claude Code 如何把一次自然语言请求转化为可持续、多轮、可审计的 Agent 执行过程。

## 文档定位与源码边界

本仓库不是 Anthropic 官方源码仓库，而是从公开发布物的 source map 恢复出的 TypeScript 源码树。当前 `package.json` 将版本标记为 `999.0.0-restored`；`src/` 下包含约 1,987 个 `.ts/.tsx` 文件、约 47.8 万行代码。它非常适合研究 Claude Code 的模块边界、数据流和工程设计，但不能把“源码中存在某个模块”等同于“当前恢复版本已能完整运行该功能”。

阅读时需要区分三类实现：

1. **完整主链路**：CLI、REPL、QueryEngine、查询循环、工具调度、权限判断、上下文压缩等核心代码相对完整。
2. **条件功能**：大量模块受 `feature(...)`、环境变量、用户类型和远程配置控制，编译或运行时可能被排除。
3. **恢复占位与外部端能力**：少量文件是 shim、空实现或兼容壳；Desktop、Web、IDE 扩展本体也不都在此仓库中，本仓库主要包含它们与本地 Agent 内核之间的协议和适配层。

因此，本文描述的是“源码呈现出的系统设计”，并在涉及恢复限制时明确标注。

## 一、总体架构

```text
┌──────────────────────────────────────────────────────────────────────┐
│ 第一层：入口层                                                       │
│ CLI / 交互式终端 / Headless / SDK / IDE / Web·Desktop·Remote 适配   │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ 标准化输入、会话参数、输出协议
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第二层：运行时层                                                     │
│ REPL / AppState / 输入队列 / Slash Commands / Hooks / 状态与生命周期 │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ 构造一轮 Agent 请求
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第三层：核心引擎                                                     │
│ QueryEngine / queryLoop / 模型流 / Context / Compact / 恢复与重试    │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ tool_use → 校验 → 授权 → 调度
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第四层：工具与能力                                                   │
│ Built-in Tools / MCP / Skill / Plugin / Agent / Command / Workflow   │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ 文件、进程、网络、凭据、持久化
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第五层：基础设施                                                     │
│ Auth / Settings / Storage / Cache / Analytics / Bridge / Transport   │
└──────────────────────────────────────────────────────────────────────┘
```

五层不是五个完全隔离的进程，而是同一 TypeScript 应用中的逻辑分层。最关键的架构原则是：

- **入口可变，内核复用**：交互式 REPL、`--print`、SDK 和远程会话最终复用相同的消息、查询和工具模型。
- **模型只提出动作，运行时决定能否执行**：模型生成 `tool_use`，真正的参数校验、权限判断、并发策略和调用发生在本地工具管线。
- **会话是持续状态机，不是单次 API 请求**：消息、文件快照、权限上下文、Token 使用、压缩边界和任务状态跨多轮保留。
- **上下文是主动管理的资源**：系统通过缓存稳定、微压缩、完整压缩、上下文折叠和溢出恢复控制上下文窗口。
- **扩展统一收敛为工具或命令**：MCP、Skill、Plugin、子 Agent 和工作流最后都会进入工具池、命令池或上下文构造过程。

## 二、一次请求的端到端逻辑

以用户在终端输入“检查项目并修复问题”为例，完整主链路如下：

1. `src/entrypoints/cli.tsx` 完成进程级初始化，然后进入 `src/main.tsx`。
2. `main.tsx` 解析 CLI 参数、设置工作目录、加载设置与权限规则、初始化模型、MCP、插件和会话恢复信息。
3. 交互模式通过 `src/replLauncher.tsx` 渲染 `src/screens/REPL.tsx`；无头模式则进入 `src/cli/print.ts`。
4. 输入先经过 `processUserInput`：解析 Slash Command、`@` 文件引用、附件、Skill、队列消息和输入元数据。
5. 运行时组装系统提示、用户上下文、Git 状态、`CLAUDE.md`/记忆文件、工具定义和 MCP 能力。
6. `QueryEngine.submitMessage()` 把新消息写入会话状态与 transcript，并建立本轮 `ToolUseContext`。
7. `query()`/`queryLoop()` 根据上下文预算决定是否先压缩，再调用模型流式接口。
8. 模型流返回文本、thinking、`tool_use` 或错误事件；文本可以立即呈现给 CLI/SDK。
9. 工具调用进入 `runTools()`：只读且并发安全的调用可批量并发，带副作用的调用按顺序执行。
10. 每个工具依次经过“工具查找 → Zod 参数校验 → 工具自定义校验 → Hook → 权限判断 → 调用 → 结果标准化”。
11. `tool_result` 被作为用户侧协议消息追加回对话，`queryLoop()` 再次调用模型；这个闭环持续到模型不再请求工具。
12. Stop Hook、预算检查、压缩边界、持久化和统计收尾完成后，本轮结束；下一次输入继续复用同一会话状态。

这条链路说明 Claude Code 的核心不是一个“大提示词”，而是一套围绕模型构建的事件循环、能力调度器和状态管理系统。

---

# 第一层：入口层

入口层负责把不同宿主的输入统一成会话请求，并把内核事件转换成宿主可消费的输出。它解决的是“从哪里进入、用什么协议交互”，而不是“Agent 如何推理”。

## 1.1 CLI 进程入口与启动编排

### 核心文件

- `src/entrypoints/cli.tsx`：发布形态的 CLI 入口，安装全局错误处理并调用主程序。
- `src/main.tsx`：参数解析和总启动编排中心。
- `src/dev-entry.ts`：恢复仓库的开发入口；检查缺失导入，并在条件满足时转发到原始 CLI 启动链路。
- `src/setup.ts`、`src/entrypoints/init.ts`：进程环境、初始化顺序及公共启动准备。

### 主要职责

`main.tsx` 是入口层的“交通枢纽”，其职责远多于普通 CLI 参数解析：

- 解析交互模式、`--print`、输入/输出格式、恢复/继续会话、模型、预算和权限模式。
- 接收 `--allowed-tools`、`--disallowed-tools`、`--tools` 和 `--mcp-config`，形成初始能力边界。
- 处理自定义 system prompt、追加 prompt、设置文件和 setting sources。
- 执行迁移、启动遥测、缓存预热、会话恢复、远程连接、SSH、assistant/bridge 等分支。
- 根据运行方式选择 REPL、Headless、远程会话或子命令处理器。

启动阶段特别强调首轮延迟：源码中多处把 MCP、插件检查和上下文预取设计为延迟或并行任务，避免全部初始化阻塞 REPL 首次渲染和第一轮 TTFT。

## 1.2 交互式终端入口

### 核心文件

- `src/replLauncher.tsx`：延迟导入并挂载 REPL。
- `src/screens/REPL.tsx`：交互式终端的顶层控制器。
- `src/components/`：消息列表、输入框、权限对话框、工具进度、任务面板等 UI。
- `src/ink/`：终端渲染、布局、文本换行、输入输出和底层 Ink 兼容代码。

### 模块逻辑

REPL 同时承担展示和交互编排：

- 接收用户输入，处理粘贴、图片、历史搜索、Vim 模式和快捷键。
- 展示模型流、thinking、工具调用进度、权限请求和错误。
- 维护输入队列，使用户可以在当前轮执行期间追加或中断请求。
- 把 MCP 状态、插件状态、任务/Agent 状态和远程 Bridge 状态映射到 UI。
- 在每轮开始时构造有效 system prompt 与 `ToolUseContext`，然后驱动查询循环。

这里的 React/Ink 组件不是核心逻辑本身。真正的查询、工具和权限能力被放在独立模块中，因此无头模式可以复用它们。

## 1.3 Headless 与结构化 I/O

### 核心文件

- `src/cli/print.ts`：`--print`/SDK 风格的非交互执行器。
- `src/cli/structuredIO.ts`：结构化输入输出控制。
- `src/cli/remoteIO.ts`：远程输入输出适配。
- `src/cli/ndjsonSafeStringify.ts`：安全生成 NDJSON。
- `src/cli/transports/`：WebSocket、SSE、Hybrid 和批量事件上传传输层。

### 模块逻辑

Headless 模式把 REPL 中的视觉交互替换为事件协议：

- 输入可以是普通文本，也可以是流式 JSON。
- 输出可包含初始化消息、用户回放、增量模型事件、工具进度、权限请求、压缩边界和最终结果。
- `QueryEngine` 持有跨轮状态，SDK 调用方可以连续提交消息，而不是每次重建整个会话。
- 权限无法通过终端对话框处理时，可交给外部 permission prompt tool 或控制通道。

这种设计让 CLI 同时可以作为人类工具、脚本工具和更大 Agent 平台的本地执行内核。

## 1.4 SDK 入口

### 核心文件

- `src/entrypoints/agentSdkTypes.ts`：SDK 暴露的查询、会话、工具和远程控制接口形状。
- `src/entrypoints/sdk/coreTypes.ts`、`coreTypes.generated.ts`：消息、结果和会话核心类型。
- `src/entrypoints/sdk/controlSchemas.ts`：控制请求的 Zod schema。
- `src/entrypoints/sdk/toolTypes.ts`：工具类型定义。

### 模块逻辑

SDK 层把内核封装为可编程接口：

- 创建/恢复/分叉会话并连续发送 prompt。
- 构造 SDK 自定义工具或进程内 MCP Server。
- 读取、列举、重命名和标记会话。
- 订阅计划任务与远程控制事件。
- 使用统一的 SDK 消息类型接收模型流、工具事件和结果。

需要注意：恢复仓库中部分 SDK 文件更接近发布类型入口或占位实现；分析 SDK 时应结合 `cli/print.ts` 和 `QueryEngine.ts` 观察真实执行路径。

## 1.5 IDE、Desktop、Web 与 Remote 入口

### IDE 集成

- `src/hooks/useIDEIntegration.tsx`：REPL 与 IDE 状态的 React 集成。
- `src/hooks/useIdeSelection.ts`、`useIdeConnectionStatus.ts`：IDE 选择和连接状态。
- `src/services/mcp/vscodeSdkMcp.ts`：通过 MCP/SDK 接入 VS Code 能力。
- `src/services/mcp/client.ts`：允许 IDE 专用 RPC 和工具发现。

IDE 扩展本体不等同于本仓库；本仓库实现的是本地 Agent 如何发现 IDE、调用 IDE 暴露的 MCP/RPC 能力、发送 diff 或诊断请求。

### Web/Desktop/Remote 适配

- `src/remote/RemoteSessionManager.ts`：远程会话生命周期、消息与权限响应。
- `src/remote/SessionsWebSocket.ts`：远程 WebSocket 会话通道。
- `src/remote/sdkMessageAdapter.ts`：SDK 消息与本地消息之间的转换。
- `src/server/directConnectManager.ts`：Direct Connect 会话管理。
- `src/bridge/`：本地进程与远端客户端之间的完整 Bridge 协议实现。

Web 或 Desktop 前端本身可能在其他代码库中；这里的重点是“远端客户端如何把输入送入相同的 QueryEngine，并接收相同的模型和工具事件”。

## 1.6 入口层的输入输出契约

| 入口 | 输入 | 输出 | 主要适配点 |
|---|---|---|---|
| 交互式 CLI | 键盘、粘贴、附件、Slash Command | Ink 终端 UI | 人机交互、权限弹窗、实时进度 |
| Headless | 文本或 stream-json | text/json/stream-json | 自动化和脚本执行 |
| SDK | API 调用与 AsyncIterable | SDKMessage 流 | 嵌入其他应用 |
| IDE | IDE MCP/RPC、选择区、诊断 | diff、状态、工具结果 | 编辑器上下文和 UI 协同 |
| Remote/Bridge | WebSocket/API 消息 | 远程事件流 | 跨设备会话与远程审批 |

---

# 第二层：运行时层

运行时层负责“一次会话怎样活着”：它接收入口层事件，维护 UI/会话状态，处理命令与队列，并在适当时机启动或中断核心引擎。

## 2.1 REPL 运行时控制器

`src/screens/REPL.tsx` 是交互模式的运行时壳。它将大量专用 Hook 组合起来，而不是把所有逻辑写成单一循环：

- `useQueueProcessor`：消费排队的用户消息。
- `useCommandQueue`：协调命令执行队列。
- `useCancelRequest`：中断当前请求。
- `useMergedTools`、`useMergedCommands`、`useMergedClients`：动态合并工具、命令和 MCP 客户端。
- `useCanUseTool`：生成当前会话的权限决策函数。
- `useSettingsChange`、`useSkillsChange`：监听配置和 Skill 变化。
- `useRemoteSession`、`useReplBridge`：接入远程会话。
- `useTasksV2`、`useTaskListWatcher`、`useSwarmInitialization`：任务和多 Agent 状态。

这种 Hook 化设计将“终端 UI 状态”和“Agent 内核状态”解耦，但 REPL 仍是交互模式中决定何时提交、排队、打断和重绘的中心。

## 2.2 AppState 与 Store

### 核心文件

- `src/state/AppStateStore.ts`：完整 `AppState` 类型和默认值。
- `src/state/store.ts`：轻量订阅式 Store。
- `src/state/AppState.tsx`：React Provider 和选择器 Hook。
- `src/state/selectors.ts`：派生状态。
- `src/state/onChangeAppState.ts`：状态变化的副作用与外部元数据映射。

### 状态内容

`AppState` 不只是 UI state，它承载：

- 工具权限上下文和附加工作目录。
- MCP 客户端、工具与资源连接状态。
- 文件历史、内容替换和回滚相关状态。
- 当前模型、thinking、fast mode、预算和用量。
- 后台任务、子 Agent、团队和 teammate view。
- 输入队列、通知、页脚、插件、IDE 与远程状态。

Store 允许核心逻辑通过 `getAppState`/`setAppState` 工作，而不直接依赖 React。`QueryEngineConfig` 也使用这两个函数注入状态访问，从而支持 Headless/SDK。

## 2.3 输入预处理与命令路由

### 核心文件

- `src/utils/processUserInput/`：用户输入标准化主链路。
- `src/commands.ts`：命令注册、过滤和统一获取。
- `src/commands/`：各 Slash Command 实现。
- `src/hooks/useMergedCommands.ts`：内置命令、插件命令、Skill 命令的运行时合并。

### 处理顺序

输入不会直接发送给模型，而会先被拆解：

1. 识别本地 Slash Command；本地命令可能直接返回结果而不触发模型。
2. 解析 `@file`、图片、粘贴文本和其他附件。
3. 加载命中的 Skill、插件命令和动态上下文。
4. 注入队列消息、Hook 消息和系统提醒。
5. 根据命令结果调整模型、工具白名单、权限模式或消息历史。
6. 生成标准 `Message[]`，再交给 QueryEngine。

这解释了为什么 Slash Command 可以改变后续会话行为：它处理的不是普通 prompt 字符串，而是运行时配置和消息状态。

## 2.4 Hook 生命周期

Hook 系统分为两类，不应混淆：

- `src/hooks/` 中的 React Hook：管理 REPL 运行时、输入、通知、IDE、队列和 UI 状态。
- 工具/Agent 生命周期 Hook：在模型调用、工具调用、压缩、停止等关键节点执行用户或插件配置的钩子。

典型生命周期包括：

- Session/Startup：会话开始与初始化。
- UserPromptSubmit：用户输入提交前后。
- PreToolUse/PostToolUse/PostToolUseFailure：工具授权、执行和失败阶段。
- PostSampling：模型采样完成后。
- Stop/SubagentStop：主 Agent 或子 Agent 结束前。
- PreCompact：压缩之前允许注入额外指令。

Hook 可以返回提示、阻断执行或提供额外上下文。源码在 `queryLoop()` 中防止错误响应与阻塞 Hook 形成无限重试，说明 Hook 被视为状态机的一部分，而非单纯日志回调。

## 2.5 中断、队列与生命周期一致性

运行时需要同时处理用户继续输入、工具仍在运行和远程控制消息：

- `AbortController` 是统一中断信号。
- 当前执行工具 ID 被保存在上下文中，便于 UI 和中断清理。
- 若流式响应已经产生 `tool_use` 但执行被中断，系统会补齐合成 `tool_result`，保持 API 消息配对不变量。
- 用户的新消息可以触发 interrupt，然后作为下一轮输入继续。
- Transcript 在调用模型前先记录用户消息，避免进程在首个模型响应前退出导致会话无法恢复。

这套一致性机制保证“可恢复会话”的数据结构不会因中断停留在非法状态。

---

# 第三层：核心引擎

核心引擎负责模型调用与 Agent 闭环。这里要区分两个模块：`QueryEngine` 是会话级门面，`queryLoop` 是单轮内部可多次迭代的状态机。

## 3.1 QueryEngine：会话级门面

### 核心文件

- `src/QueryEngine.ts`
- `src/query.ts`
- `src/query/config.ts`
- `src/query/deps.ts`
- `src/query/stopHooks.ts`
- `src/query/tokenBudget.ts`

### 持有的长期状态

每个会话对应一个 `QueryEngine`，长期保存：

- `mutableMessages`：完整会话消息。
- `readFileState`：文件读取缓存/状态，用于工具一致性。
- `totalUsage`：累计 Token 和成本。
- `permissionDenials`：SDK 需要返回的拒绝记录。
- `AbortController`：当前执行的中断控制。
- 已发现 Skill 和已加载嵌套记忆路径。

`submitMessage()` 是一轮请求的入口。它完成 system prompt 获取、输入处理、transcript 写入、工具权限包装、SDK 初始化消息发出，然后调用 `query()` 消费内部 Agent 循环。

### 为什么使用门面类

如果直接从 REPL 调用 `query()`，Headless/SDK 就需要重复管理消息、使用量、恢复和权限。`QueryEngine` 把这些会话职责集中起来，使入口层只负责输入输出协议。

## 3.2 queryLoop：显式 Agent 状态机

`src/query.ts` 中的 `query()` 创建初始 State，`queryLoop()` 通过 `while`/状态转换持续运行。State 包含：

- 当前消息数组和 `ToolUseContext`。
- 自动压缩跟踪状态。
- 最大输出 Token 恢复次数。
- 是否已经尝试 reactive compact。
- 当前 turn count、stop-hook 状态和 transition reason。
- 待完成的工具摘要和输出 Token override。

一轮内部可能经历多次状态迁移：

```text
prepare_context
  → model_stream
  → tool_execution
  → append_tool_results
  → model_stream
  → completed
```

也可能进入恢复分支：

```text
prompt_too_long
  → context_collapse_drain
  → reactive_compact
  → retry

max_output_tokens
  → increase_output_limit
  → continuation_message
  → retry（有次数上限）

model_overloaded
  → fallback_model
  → strip_incompatible_signatures
  → retry
```

显式记录 `transition.reason` 的价值在于：恢复行为可防重入，避免“压缩失败—Hook 阻断—再次压缩”的死循环。

## 3.3 模型请求构造

### 核心模块

- `src/services/api/claude.ts`：模型请求、流式响应和 API 适配。
- `src/utils/queryContext.ts`：system/user/system context 的并行构造。
- `src/constants/prompts.ts`：系统提示的分段来源。
- `src/utils/systemPrompt.ts`：生成最终有效 system prompt。
- `src/context.ts`：Git 状态、日期、`CLAUDE.md` 和记忆文件上下文。

### 上下文组成

最终请求不是单一字符串，大致由以下部分组成：

1. 默认或用户覆盖的 system prompt。
2. 根据工具、模型、权限模式和功能开关生成的动态系统段。
3. 工作目录的 `CLAUDE.md`、记忆文件和附加目录说明。
4. 会话启动时的 Git 分支、状态和最近提交快照。
5. 当前日期、输出风格、计划模式、团队/Agent 等运行时提示。
6. 历史消息、附件、压缩摘要和工具结果。
7. 当前可见工具的 JSON Schema。

为了提高 prompt cache 命中率，工具在合并后按名称稳定排序；部分系统段也有独立缓存。动态但可后置的信息尽量不破坏前缀稳定性。

## 3.4 流式模型事件

模型调用通过 AsyncGenerator/AsyncIterable 输出事件。引擎可以在完整回复结束前：

- 把文本增量交给 UI 或 SDK。
- 收集 thinking 与签名块。
- 识别完整 `tool_use`。
- 在满足条件时提前启动 `StreamingToolExecutor`。
- 记录请求 ID、使用量、缓存命中和错误类型。

流式设计减少了“等模型完整生成后才开始工具”的串行等待。与此同时，引擎必须保证一个 assistant 消息里的工具块、后续结果和中断补偿保持协议一致。

## 3.5 Context 协调器

上下文管理分为“发现、组装、投影、预算”四类：

- **发现**：扫描 `CLAUDE.md`、memory、Skill、插件、Git 和 IDE 信息。
- **组装**：按 system/user/system context 和消息历史形成请求。
- **投影**：对于 UI 保留完整历史，但 SDK/headless 可以投影已 snip 的视图。
- **预算**：估算消息、工具结果、图片和输出预留所占 Token。

`src/context.ts` 会缓存会话初始 Git 状态与用户上下文，避免每次模型迭代重复执行昂贵 I/O。`CLAUDE.md` 的内容也会进入缓存，供权限分类器等模块复用，减少依赖环。

## 3.6 Compact：多级上下文压缩

### 核心文件

- `src/services/compact/autoCompact.ts`
- `src/services/compact/compact.ts`
- `src/services/compact/microCompact.ts`
- `src/services/compact/sessionMemoryCompact.ts`
- `src/services/compact/apiMicrocompact.ts`
- `src/services/compact/postCompactCleanup.ts`
- `src/services/compact/prompt.ts`

### 级别一：Microcompact

Microcompact 不总结整个对话，而是优先清理旧的、体积较大的工具结果和图片内容：

- 估算各工具结果 Token。
- 识别允许清理的工具类型。
- 保留工具调用与结果配对结构，只把旧内容替换为清理标记。
- 支持基于缓存和时间的触发策略。

它的优势是成本低、对近期语义影响小，并能保持 prompt 前缀稳定。

### 级别二：Auto/Full Compact

当上下文接近模型窗口阈值时，`shouldAutoCompact()` 根据有效窗口、输出预留、缓冲区和连续失败次数决定是否压缩。完整压缩会：

1. 清理不适合摘要的图片或重复附件。
2. 调用专用 compact prompt 生成结构化对话摘要。
3. 写入 `compact_boundary`，记录压缩元数据。
4. 以“摘要 + 保留的近期消息”替换请求上下文。
5. 恢复最近读过的重要文件、Skill、计划和异步 Agent 状态附件。
6. 执行压缩后的缓存与临时状态清理。

恢复附件很重要：只有摘要可能丢失精确代码，系统会在预算范围内重新附加最近读取的文件内容。

### 级别三：Partial/Session Memory Compact

部分压缩保留较早或较新的某一段原始消息，仅对选定区间摘要。Session Memory Compact 还会：

- 计算应该保留的消息边界。
- 调整边界以避免拆开 `tool_use`/`tool_result`。
- 根据 Session Memory 配置生成可继续执行的压缩结果。

### 级别四：溢出恢复

如果 API 仍返回 prompt too long：

- 先尝试提交已分阶段准备的 context collapse。
- 再尝试 reactive compact。
- 对媒体尺寸错误可执行去图片后的恢复。
- 每种恢复有单次或次数限制；无法恢复时才把真实错误返回用户。

## 3.7 Stop Hook 与完成判定

“模型没有调用工具”并不一定立即结束：

- PostSampling Hook 在模型响应完成后异步执行。
- Stop Hook 可以允许结束、阻止继续或返回阻塞错误并要求模型再处理。
- API 错误不运行普通 Stop Hook，避免把错误当成有效回复反复重试。
- Token Budget 可以在模型看似完成时注入继续提示，直到满足预算策略或边际收益规则。

因此，完成条件由模型输出、Hook、预算、错误状态和主动模式共同决定。

---

# 第四层：工具与能力

工具层把模型输出的抽象动作变成受控的本地或远程操作。它是 Claude Code 从“聊天”变成“编码 Agent”的关键。

## 4.1 Tool 抽象

### 核心文件

- `src/Tool.ts`：Tool、ToolUseContext、ToolResult 等核心接口。
- `src/tools.ts`：内置工具注册、过滤和工具池组装。
- `src/utils/toolPool.ts`：去重、排序、Coordinator 过滤。
- `src/services/tools/toolExecution.ts`：单工具执行管线。
- `src/services/tools/toolOrchestration.ts`：多工具编排。
- `src/services/tools/StreamingToolExecutor.ts`：流式提前执行。

一个 Tool 通常提供：

- `name`、描述和 prompt。
- Zod `inputSchema`。
- `isEnabled()` 动态启用条件。
- `isReadOnly()`/`isConcurrencySafe()` 调度属性。
- `validateInput()` 业务校验。
- `checkPermissions()` 工具级权限预判。
- `call()` 实际执行逻辑和进度回调。
- UI 渲染与结果映射函数。

Tool 定义同时服务于模型 schema、权限系统、执行器和 UI，因此它不是简单函数，而是能力元数据的聚合边界。

## 4.2 工具池组装

`getAllBaseTools()` 建立候选内置工具；`getTools()` 再根据模式和功能开关过滤；`assembleToolPool()` 合并 MCP 工具。

组装顺序包括：

1. 根据 REPL/simple/coordinator 等模式选取基础工具。
2. 应用 deny rules。
3. 调用每个工具的 `isEnabled()`。
4. 合并动态 MCP 工具。
5. 按名称去重，入口传入工具具有优先级。
6. 内置工具与 MCP 工具分别稳定排序，提高 prompt cache 稳定性。
7. Coordinator 模式进一步缩小主协调器可见工具集合。

模型只会看到最终工具池的 schema，但执行时仍会再次查找和授权，不能把“未发送 schema”当作唯一安全边界。

## 4.3 工具执行管线

`runToolUse()` 的关键顺序如下：

```text
查找工具/兼容旧别名
  → 检查 AbortSignal
  → Zod schema 校验
  → validateInput 业务校验
  → PreToolUse Hook
  → canUseTool / 工具权限判断
  → call() 执行并流式报告进度
  → 规范化 ToolResult
  → PostToolUse 或 PostToolUseFailure Hook
  → 生成与 tool_use_id 配对的 tool_result 消息
```

错误不会简单抛出并破坏循环，而会尽量转换成带 `is_error` 的 `tool_result` 返回模型，让模型有机会修正参数或换一种方案。

## 4.4 并发调度

`runTools()` 将模型同一回复中的工具调用分批：

- 连续的并发安全工具进入同一并发批次。
- 非并发安全工具各自形成串行批次。
- 并发上限默认为 10，可由环境变量调整。
- 并发工具产生的 context modifier 先按 tool ID 暂存，再按原工具顺序应用，避免竞争导致状态不确定。

典型只读操作如多个文件读取或搜索可以并行；编辑、Shell 和改变状态的工具需要保守串行。工具自身的 `isConcurrencySafe(input)` 可以根据具体参数动态判断。

## 4.5 内置工具模块

### 文件与搜索

- `FileReadTool`：读取文本、图片等文件内容并维护读取状态。
- `FileEditTool`：以精确替换方式修改文件。
- `FileWriteTool`：创建或整体写入文件。
- `GlobTool`、`GrepTool`：文件发现与内容搜索。
- `NotebookEditTool`：Notebook 单元格级编辑。
- `LSPTool`：语言服务诊断或导航。

这些工具与文件快照、权限路径和 post-compact 文件恢复共同工作，不是彼此孤立的 I/O 包装器。

### 进程与终端

- `BashTool`、`PowerShellTool`：命令执行、输出流、超时和后台化。
- `TaskOutputTool`、`TaskStopTool`：后台任务输出读取和终止。
- `TerminalCaptureTool`：终端状态捕获。
- `REPLTool`：受条件控制的 REPL 能力。

Shell 工具的权限规则需要解析命令前缀、通配符和危险模式；Windows PowerShell 与 Bash 有各自的安全判断路径。

### Web 与外部信息

- `WebSearchTool`、`WebFetchTool`：搜索和网页获取。
- `WebBrowserTool`：条件启用的浏览器能力。
- `MCPTool`、`ListMcpResourcesTool`、`ReadMcpResourceTool`：外部 MCP 能力入口。

### 规划、询问与任务

- `EnterPlanModeTool`、`ExitPlanModeTool`：计划模式状态转换。
- `AskUserQuestionTool`：请求外部决策。
- `TodoWriteTool`：传统待办记录。
- `TaskCreate/Get/Update/List`：结构化任务系统。
- `EnterWorktreeTool`、`ExitWorktreeTool`：隔离工作树生命周期。

### Agent 与团队

- `AgentTool`：创建子 Agent/任务执行单元。
- `SendMessageTool`：Agent 间通信。
- `TeamCreateTool`、`TeamDeleteTool`：团队生命周期。
- `VerifyPlanExecutionTool`：计划执行验证。

`src/coordinator/` 会限制协调器主 Agent 只保留编排所需能力，而 worker 使用适合执行的工具集合，形成“决策面/执行面”分离。

## 4.6 权限系统

### 核心文件

- `src/hooks/useCanUseTool.tsx`
- `src/hooks/toolPermission/`
- `src/utils/permissions/permissions.ts`
- `src/utils/permissions/permissionSetup.ts`
- `src/utils/permissions/permissionsLoader.ts`
- `src/utils/permissions/PermissionUpdate.ts`
- `src/utils/permissions/shellRuleMatching.ts`

### 决策层次

权限不是一个布尔开关，而是多来源规则合并：

1. 工具是否在最终工具池中。
2. 企业、项目、用户和本地设置中的 allow/ask/deny 规则。
3. CLI 传入的 allowed/disallowed tools。
4. 当前权限模式，如默认、计划、自动或 bypass 类模式。
5. 工具自己的 `checkPermissions()`。
6. 路径是否位于工作目录或额外授权目录。
7. 交互处理器、Coordinator/Worker 处理器或远程权限桥。

规则更新既可以只作用于当前会话，也可以持久化到设置。系统还检测危险权限、过宽 Shell 前缀和被 ask/deny 遮蔽的不可达 allow 规则。

## 4.7 MCP

### 核心文件

- `src/services/mcp/config.ts`：多作用域配置加载、策略过滤、去重和启停。
- `src/services/mcp/client.ts`：连接、工具/资源/命令发现和调用。
- `src/services/mcp/auth.ts`：MCP OAuth。
- `src/services/mcp/elicitationHandler.ts`：表单或 URL elicitation。
- `src/tools/MCPTool/`：把 MCP 工具包装为统一 Tool。

### 逻辑

MCP 配置可来自项目、用户、企业、插件、SDK 或 claude.ai。系统会：

- 规范化 server name 和工具名。
- 展开环境变量并应用 allowlist/denylist 管理策略。
- 按 stdio、HTTP、SSE、WebSocket 或进程内 transport 建立连接。
- 批量发现 tools/resources/prompts。
- 转换 MCP 内容块、图片、blob 和 structured content。
- 将 MCP 工具纳入同一权限和执行管线。
- 处理 OAuth、重连、超时、URL elicitation 和结果压缩。

MCP 并不是绕过权限的旁路；它最终仍是 Tool，并受工具池与 `canUseTool` 约束。

## 4.8 Skill、Plugin、Agent 与 Command

### Skill

Skill 通常是带说明、资源和触发规则的可加载能力。`SkillTool` 负责显式调用，输入预处理也可根据命令或语义加载 Skill 内容。Skill 可来自内置目录、用户/项目目录、插件或远程搜索结果。

### Plugin

Plugin 是更大的扩展包，可贡献：

- Slash Commands。
- Skills。
- Agents。
- Hooks。
- MCP Servers。
- 设置与资源。

运行时加载插件后，把贡献内容分别并入命令池、工具/Skill 上下文、Agent 定义和 Hook 注册表。

### Agent

Agent 定义包含标识、适用条件、system prompt、工具限制、模型和权限信息。`AgentTool` 负责调度；worker 可在独立上下文或进程中运行，并通过任务、消息或团队状态回传结果。

### Command

Command 是用户显式入口；有些完全本地执行，有些把结构化提示追加给模型。它可以改变模型、权限、模式、上下文或会话，而不一定调用外部工具。

---

# 第五层：基础设施

基础设施层向上提供稳定的身份、配置、持久化、缓存、观测和跨端传输。它决定系统是否可恢复、可管理和可诊断。

## 5.1 认证与凭据

### 核心模块

- `src/cli/handlers/auth.ts`：登录、状态和登出命令。
- `src/services/oauth/`：OAuth 流程。
- `src/services/mcp/auth.ts`：MCP Server OAuth。
- `src/utils/config.ts` 及凭据相关工具：API Key、Token 和配置读取。

认证层需要支持 Anthropic 账户/API、第三方模型平台和 MCP OAuth。凭据不会作为普通 prompt 传给模型；请求适配层只获取调用所需的认证材料。远程 Bridge 还使用 JWT、work secret 和可信设备信息建立通道身份。

## 5.2 设置与配置分层

设置可能来自：

- 命令行参数和环境变量。
- 用户级、项目级和本地设置文件。
- 企业托管设置。
- SDK 初始化请求。
- 功能开关与远程实验配置。

加载后并不是简单覆盖：权限规则、MCP Server 和插件等对象有各自的合并、去重和策略过滤逻辑。托管设置可以限制用户配置，例如仅允许企业批准的 MCP Server 或权限规则。

## 5.3 会话、Transcript 与文件历史

### 核心文件

- `src/history.ts`：会话历史与消息恢复。
- `src/utils/log.ts` 及 session storage 相关模块：transcript 写入和刷新。
- `src/services/SessionMemory/`：会话记忆。
- `src/assistant/sessionHistory.ts`、`sessionDiscovery.ts`：持久助手会话发现。
- `src/utils/fileHistory.ts`、`src/hooks/useFileHistorySnapshotInit.ts`：文件快照与回滚。

持久化对象至少包括：

- 用户/助手/工具/系统消息。
- 压缩边界和摘要元数据。
- 文件修改前后的快照与内容替换。
- 会话 ID、父工具 ID、Agent/任务关系。
- 使用量、权限拒绝和部分运行元数据。

用户消息在模型请求前写入 transcript，是为了解决“请求已接受但进程在首个响应前退出”的恢复缺口。文件历史则让 rewind/恢复不只回滚对话，还能恢复代码状态。

## 5.4 Cache 体系

源码中存在多类缓存，目标不同：

- **Prompt cache 稳定性**：工具排序、系统段缓存和稳定前缀。
- **上下文发现缓存**：Git 状态、`CLAUDE.md`、memory、Skill 与插件结果。
- **MCP 连接与发现缓存**：连接、工具、资源和 prompt 列表。
- **文件读取状态**：避免编辑建立在未知或过期内容上。
- **Microcompact cache edits**：记录可清理块并在安全时提交。
- **远程配置缓存**：减少启动阶段网络依赖。

缓存不是统一 Key-Value 服务，而是分布在各边界的专用缓存。发生设置变化、cache breaker、压缩或连接重置时，对应模块显式失效。

## 5.5 Analytics、Telemetry 与诊断

### 主要能力

- 启动阶段和首轮延迟 checkpoint。
- 模型请求、Token、缓存和成本统计。
- 工具调用、权限结果、错误分类和 MCP 类型。
- Query chain ID、depth 和子 Agent 关联。
- 慢 Hook、慢工具阶段和异常日志。

代码通常对工具名、URL 或错误内容做安全化再进入统计字段，区分可记录元数据和潜在代码/路径内容。诊断日志与产品分析事件也分开处理。

## 5.6 Bridge 传输层

### 核心模块

- `src/bridge/bridgeMain.ts`、`remoteBridgeCore.ts`：Bridge 主循环。
- `src/bridge/createSession.ts`、`sessionRunner.ts`：远程会话建立与执行。
- `src/bridge/bridgeApi.ts`：后端 API 客户端。
- `src/bridge/bridgeMessaging.ts`：消息发送。
- `src/bridge/inboundMessages.ts`、`inboundAttachments.ts`：远端输入和附件。
- `src/bridge/bridgePermissionCallbacks.ts`：远程权限审批。
- `src/bridge/replBridge.ts`、`replBridgeTransport.ts`、`replBridgeHandle.ts`：本地 REPL 与 Bridge 的传输接口。
- `src/bridge/jwtUtils.ts`、`trustedDevice.ts`、`workSecret.ts`：通道认证和信任。

### 逻辑

Bridge 将远端客户端视为另一个入口：

1. 本地创建或附着会话，取得 Bridge 标识和认证材料。
2. 建立 API/WebSocket 或轮询通道。
3. 远端 prompt 和附件被清洗、标准化后进入本地输入队列。
4. 本地模型流、工具状态和 UI 状态上传到远端。
5. 需要人工权限时，权限请求可发送到远端并等待回调。
6. Flush gate、容量唤醒、轮询退避和状态指针处理断线与重连。

Bridge 不重新实现 Agent；它只把远端输入输出接到同一个本地运行时，因此文件和 Shell 操作仍发生在本地受控环境。

## 5.7 Feature Gate 与产品变体

`feature('...')`、`USER_TYPE`、环境变量和远程配置共同决定模块是否进入工具池、命令池或启动链路。它们分别解决：

- 编译时裁剪与包体变体。
- 内部/外部用户差异。
- 本地实验或紧急开关。
- 服务端灰度和模型配置。

因此，阅读源码时应沿“注册点 → gate → `isEnabled()` → 权限 → 调用点”完整追踪，不能只看到文件就判定功能可用。

---

# 跨层关键机制

## 3.1 消息是不变量载体

系统的核心数据不是字符串，而是带类型、UUID、父工具 ID 和元数据的 Message：

- user：用户输入、系统注入和 `tool_result`。
- assistant：模型文本、thinking 和 `tool_use`。
- system：初始化、状态、压缩边界和控制消息。
- progress：工具与 Hook 的增量进度。

必须保持 `tool_use.id` 与 `tool_result.tool_use_id` 配对。中断、错误和并发执行都围绕这个不变量做补偿。

## 3.2 ToolUseContext 是跨层执行上下文

`ToolUseContext` 将运行时和工具层连接起来，包含：

- 当前消息、工具、命令、MCP 客户端和模型。
- 权限上下文、工作目录和读取状态。
- AbortController、进度回调和在途工具 ID。
- 文件历史、归属信息和应用状态更新函数。
- Agent ID、query source、query tracking 和预算。

工具不需要直接引用 REPL，但仍能读取执行所需的全部上下文。

## 3.3 主 Agent 与子 Agent 复用同一内核

子 Agent 并不是另一套产品：

- 使用 Agent Definition 构造不同 system prompt、工具集合和模型。
- 通过相同 query/tool pipeline 执行。
- 使用 `parent_tool_use_id`、agent ID、任务和消息通道关联父子关系。
- Coordinator 模式用工具过滤把主 Agent 变成编排器。

复用保证权限、工具错误、MCP 和统计语义在主/子 Agent 间一致。

## 3.4 安全边界是分层防御

Claude Code 不依赖单一确认框：

- 工具是否暴露给模型。
- deny rule 是否直接排除。
- schema 和业务参数是否有效。
- 路径和命令是否满足规则。
- Hook 是否阻断。
- 人工/远程审批是否允许。
- 工具自身是否支持当前环境。

任一层拒绝都应转成明确结果，而不是静默执行。

---

# 源码目录导航

| 路径 | 层级 | 作用 |
|---|---|---|
| `src/main.tsx` | 入口 | 参数解析与启动总编排 |
| `src/entrypoints/` | 入口 | CLI、SDK、MCP 导出入口 |
| `src/cli/` | 入口/运行时 | Headless、结构化 I/O、传输与子命令 |
| `src/screens/REPL.tsx` | 运行时 | 交互式会话控制器 |
| `src/components/`、`src/ink/` | 运行时 | 终端 UI 与渲染基础 |
| `src/state/`、`src/context/` | 运行时 | 应用状态和 React Context |
| `src/hooks/` | 运行时 | 队列、输入、权限、IDE、远程和通知 |
| `src/QueryEngine.ts` | 核心 | 会话级查询门面 |
| `src/query.ts`、`src/query/` | 核心 | Agent 状态机、预算和停止逻辑 |
| `src/services/api/` | 核心/基础 | 模型 API 与流式响应 |
| `src/services/compact/` | 核心 | 上下文压缩和恢复 |
| `src/tools/`、`src/tools.ts` | 工具 | 内置工具及注册表 |
| `src/services/tools/` | 工具 | 校验、授权、执行和并发调度 |
| `src/services/mcp/` | 工具/基础 | MCP 配置、连接、发现、OAuth 和调用 |
| `src/skills/`、`src/plugins/` | 工具 | Skill 与插件扩展 |
| `src/coordinator/`、`src/tasks/` | 工具/运行时 | 多 Agent 编排与任务状态 |
| `src/utils/permissions/` | 工具/基础 | 权限规则、模式与 Shell 匹配 |
| `src/history.ts`、SessionMemory | 基础 | 会话持久化与恢复 |
| `src/bridge/`、`src/remote/` | 基础/入口 | 远程控制和跨端传输 |
| `shims/`、`vendor/` | 兼容 | 本地包 shim 与恢复兼容实现 |

# 设计上的重点与代价

## 优点

- 同一内核支持交互、脚本、SDK 和远程控制。
- 工具调用有 schema、权限、Hook 和结果协议多层约束。
- AsyncGenerator 贯穿模型流与工具流，适合低延迟 UI。
- 上下文管理不是一次性截断，而是多级、有恢复附件的策略。
- MCP/Skill/Plugin/Agent 最终收敛到少数统一抽象。
- Transcript、文件历史和压缩边界让长会话具备恢复能力。

## 复杂度代价

- `main.tsx`、`REPL.tsx`、`query.ts` 和 `cli/print.ts` 体量很大，承担大量分支编排。
- Feature gate 和产品变体让静态追踪变复杂。
- React UI 状态、核心状态和持久化状态之间需要大量同步代码。
- 流式工具提前执行提高性能，也显著增加中断与消息配对复杂度。
- 恢复源码中的 shim/占位文件会干扰“代码存在即完整”的判断。

# 建议的源码阅读顺序

1. `src/entrypoints/cli.tsx` → `src/main.tsx`：理解启动分流。
2. `src/replLauncher.tsx` → `src/screens/REPL.tsx`：理解交互运行时。
3. `src/QueryEngine.ts`：理解一轮消息如何进入核心。
4. `src/query.ts`：理解模型—工具—模型闭环和恢复状态机。
5. `src/Tool.ts` → `src/tools.ts`：理解工具抽象与注册。
6. `src/services/tools/toolOrchestration.ts` → `toolExecution.ts`：理解并发、授权和结果。
7. `src/context.ts` → `src/services/compact/`：理解上下文与压缩。
8. `src/services/mcp/`、`src/skills/`、`src/plugins/`：理解扩展系统。
9. `src/utils/permissions/`：理解安全边界。
10. `src/history.ts`、`src/bridge/`：理解持久化与远程控制。

# 本地运行与验证

```bash
bun install
bun run version
bun run dev
```

环境要求以 `package.json` 为准：Bun ≥ 1.3.5、Node.js ≥ 24。恢复仓库没有完整的一等测试脚本；修改核心模块后应至少执行版本启动检查，并针对受影响的命令、工具或 REPL 路径进行手工验证。

# 总结

Claude Code 可以概括为一个“以消息状态机为核心、以工具协议为执行边界、以上下文管理为续航系统”的本地 Agent Runtime：

- 入口层让人、脚本、SDK、IDE 和远端客户端共享内核。
- 运行时层维持会话、队列、命令、Hook 和 UI 生命周期。
- 核心引擎在模型流、工具调用、压缩、错误恢复之间执行显式状态转换。
- 工具层通过统一抽象把文件、Shell、Web、MCP、Skill、Plugin 和 Agent 纳入同一授权管线。
- 基础设施层提供身份、配置、持久化、缓存、观测与 Bridge，使 Agent 能长期、跨端并可恢复地运行。

真正值得借鉴的不是某一个隐藏命令，而是这些模块之间的契约：模型负责提出下一步，运行时负责维护不变量，权限系统决定边界，工具执行器产生可验证结果，上下文系统再把结果带回下一轮推理。
