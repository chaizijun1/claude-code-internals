# Claude Code 深度解析白皮书

## 从入门到精通：架构剖析、工作原理与高效使用指南

**版本**: 基于 Claude Code v2.1.88 源码分析  
**日期**: 2026年4月  
**来源**: 基于公开仓库 `sanbuphy/claude-code-source-code` 与 `ChinaSiro/claude-code-sourcemap` 的深度逆向工程分析

---

## 目录

- [第一部分：概述与背景](#第一部分概述与背景)
- [第二部分：项目结构全景](#第二部分项目结构全景)
- [第三部分：核心架构深度解析](#第三部分核心架构深度解析)
- [第四部分：工具系统](#第四部分工具系统)
- [第五部分：权限与安全模型](#第五部分权限与安全模型)
- [第六部分：Hook 事件系统](#第六部分hook-事件系统)
- [第七部分：MCP 协议集成](#第七部分mcp-协议集成)
- [第八部分：配置体系](#第八部分配置体系)
- [第九部分：上下文管理与压缩](#第九部分上下文管理与压缩)
- [第十部分：Agent 智能体系统](#第十部分agent-智能体系统)
- [第十一部分：System Prompt 构造机制](#第十一部分system-prompt-构造机制)
- [第十二部分：API 通信与重试机制](#第十二部分api-通信与重试机制)
- [第十三部分：构建系统与编译优化](#第十三部分构建系统与编译优化)
- [第十四部分：高效使用技巧（入门篇）](#第十四部分高效使用技巧入门篇)
- [第十五部分：高效使用技巧（进阶篇）](#第十五部分高效使用技巧进阶篇)
- [第十六部分：高效使用技巧（精通篇）](#第十六部分高效使用技巧精通篇)
- [第十七部分：内部隐藏功能与 Feature Gate](#第十七部分内部隐藏功能与-feature-gate)
- [第十八部分：架构设计模式总结](#第十八部分架构设计模式总结)
- [第十九部分：与标准 Claude API 的本质差异](#第十九部分与标准-claude-api-的本质差异)
- [第二十部分：未来路线图与展望](#第二十部分未来路线图与展望)
- [附录](#附录)

---

## 第一部分：概述与背景

### 1.1 什么是 Claude Code

Claude Code 是 Anthropic 官方推出的 AI 编程 CLI（命令行界面）工具。它不仅仅是一个简单的聊天界面——而是一个**完整的智能体框架（Agent Framework）**，具备：

- **文件系统完整访问** — 读取、编辑、创建文件
- **Shell 命令执行** — 运行任意 Bash/PowerShell 命令
- **多智能体协调** — 主代理可以派生子代理并行工作
- **MCP 协议集成** — 通过标准协议扩展工具能力
- **上下文智能管理** — 自动压缩、Token 预算跟踪
- **权限安全体系** — 多层权限模型防止误操作
- **Hook 事件机制** — 用户可在关键节点注入自定义逻辑
- **会话持久化** — 可恢复、可中断的长期会话

### 1.2 技术规模

| 指标 | 数值 |
|------|------|
| TypeScript 源文件数 | **1,884** |
| 总代码行数 | **~513,000 LOC** |
| 工具实现数量 | **45+** |
| 斜杠命令数量 | **80+** |
| 服务模块数 | **38** |
| MCP 传输协议支持 | **6 种** |
| Hook 事件类型 | **15+** |
| Feature Gate 模块 | **108 个** |

### 1.3 技术栈

```
语言:      TypeScript (ES2022 target)
运行时:     Node.js >= 18 / Bun (内部)
UI 框架:    React + Ink (终端 UI 渲染)
构建工具:   esbuild + 自定义转换管道
包管理:     npm (ESM 模块)
API:       Anthropic Messages API
协议:      MCP (Model Context Protocol)
```

---

## 第二部分：项目结构全景

### 2.1 顶层目录结构

```
claude-code/
├── src/                          # 主应用源码 (1,884 文件)
│   ├── entrypoints/              # 入口文件
│   │   └── cli.tsx               # CLI 主入口 (~300 行)
│   ├── main.tsx                  # 主协调器 (~4,683 行)
│   ├── query.ts                  # 核心查询循环 (~1,729 行) ★ 最关键文件
│   ├── QueryEngine.ts            # 查询引擎封装
│   ├── Tool.ts                   # 工具基础接口
│   ├── tools.ts                  # 工具注册表
│   ├── commands.ts               # 命令注册表
│   ├── context.ts                # 上下文组装
│   │
│   ├── tools/                    # 工具实现 (~45 个)
│   │   ├── AgentTool/            # 智能体工具 (17 文件, 400KB)
│   │   ├── BashTool/             # Bash 执行
│   │   ├── FileEditTool/         # 文件编辑 (8 文件)
│   │   ├── FileReadTool/         # 文件读取
│   │   ├── FileWriteTool/        # 文件写入
│   │   ├── GlobTool/             # 文件搜索
│   │   ├── GrepTool/             # 内容搜索
│   │   ├── WebFetchTool/         # 网页获取
│   │   ├── WebSearchTool/        # 网页搜索
│   │   ├── MCPTool/              # MCP 工具桥接
│   │   ├── NotebookEditTool/     # Jupyter 编辑
│   │   ├── SkillTool/            # 技能调用
│   │   └── ...                   # 更多工具
│   │
│   ├── commands/                 # 斜杠命令 (~80 个)
│   │   ├── commit.ts             # /commit
│   │   ├── review.ts             # /review
│   │   ├── mcp/                  # /mcp 管理
│   │   ├── config/               # /config 配置
│   │   ├── tasks/                # /tasks 任务
│   │   └── ...
│   │
│   ├── services/                 # 业务服务 (38 个目录)
│   │   ├── api/                  # API 通信层
│   │   │   ├── claude.ts         # Anthropic API 客户端 (125KB)
│   │   │   ├── withRetry.ts      # 重试逻辑 (28KB)
│   │   │   └── errors.ts         # 错误处理
│   │   ├── mcp/                  # MCP 服务 (25 文件, 900KB)
│   │   │   ├── client.ts         # MCP 客户端 (119KB)
│   │   │   ├── config.ts         # MCP 配置
│   │   │   ├── auth.ts           # OAuth 认证 (88KB)
│   │   │   └── types.ts          # 类型定义
│   │   ├── analytics/            # 遥测分析
│   │   ├── compact/              # 上下文压缩
│   │   ├── tools/                # 工具执行服务
│   │   │   └── StreamingToolExecutor.ts  # 流式工具执行器
│   │   └── ...
│   │
│   ├── utils/                    # 工具函数 (30+ 组)
│   │   ├── permissions/          # 权限系统 (26 文件, 720KB)
│   │   ├── hooks/                # Hook 执行
│   │   ├── settings/             # 配置管理 (50+ 文件)
│   │   ├── config.ts             # 配置层级
│   │   ├── claudemd.ts           # CLAUDE.md 解析
│   │   ├── tokens.ts             # Token 计数
│   │   └── ...
│   │
│   ├── components/               # React/Ink 终端 UI 组件 (40+)
│   ├── state/                    # 应用状态管理
│   ├── types/                    # 类型定义
│   ├── hooks/                    # React Hooks
│   ├── constants/                # 常量与 Prompt 模板
│   │   ├── prompts.ts            # System Prompt 构造
│   │   └── systemPromptSections.ts  # Prompt 缓存
│   ├── skills/                   # 技能系统
│   ├── tasks/                    # 任务管理
│   │
│   ├── coordinator/              # 多智能体协调 (Feature-gated)
│   ├── assistant/                # KAIROS 自主模式
│   ├── bridge/                   # 远程控制
│   ├── daemon/                   # 后台守护进程
│   ├── remote/                   # 远程触发器
│   ├── voice/                    # 语音功能
│   └── vim/                      # Vim 集成
│
├── scripts/                      # 构建脚本
│   ├── build.mjs                 # 主构建流程
│   ├── prepare-src.mjs           # 源码预处理
│   └── transform.mjs             # 代码转换
├── vendor/                       # 原生模块存根
├── stubs/                        # Bun 编译时内部功能存根
├── tools/                        # 外部工具 (ripgrep 等)
├── package.json                  # v2.1.88
└── tsconfig.json                 # ES2022 配置
```

### 2.2 关键文件大小排名

| 排名 | 文件 | 大小 | 功能 |
|------|------|------|------|
| 1 | `main.tsx` | 803 KB | 主协调器、REPL 启动 |
| 2 | `tools/AgentTool/AgentTool.tsx` | 233 KB | 智能体工具核心 |
| 3 | `services/api/claude.ts` | 125 KB | API 通信客户端 |
| 4 | `services/mcp/client.ts` | 119 KB | MCP 客户端 |
| 5 | `services/mcp/auth.ts` | 88 KB | MCP OAuth 认证 |
| 6 | `query.ts` | 68 KB | 核心查询循环 |
| 7 | `utils/permissions/permissions.ts` | 52 KB | 权限引擎 |
| 8 | `utils/permissions/yoloClassifier.ts` | 52 KB | Auto 模式分类器 |
| 9 | `utils/analyzeContext.ts` | 42 KB | 上下文分析 |
| 10 | `services/mcp/config.ts` | 51 KB | MCP 配置管理 |

---

## 第三部分：核心架构深度解析

### 3.1 系统分层架构

```
┌────────────────────────────────────────────────────────────┐
│                     CLI Entry Layer                         │
│              (cli.tsx → main.tsx → REPL)                    │
├────────────────────────────────────────────────────────────┤
│                    Command Dispatch                         │
│           (/commands → processUserInput)                    │
├────────────────────────────────────────────────────────────┤
│                     Query Engine                            │
│            (QueryEngine.ts → query.ts)                      │
│     ┌──────────────────────────────────────────┐           │
│     │     Core Agent Loop (异步生成器)          │           │
│     │  ┌─────────┐  ┌──────────┐  ┌─────────┐ │           │
│     │  │API 调用  │→ │工具执行   │→ │上下文   │ │           │
│     │  │claude.ts │  │Streaming │  │管理     │ │           │
│     │  │         │  │ToolExec  │  │compact  │ │           │
│     │  └─────────┘  └──────────┘  └─────────┘ │           │
│     └──────────────────────────────────────────┘           │
├────────────────────────────────────────────────────────────┤
│                      Tool Layer                             │
│   ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐          │
│   │  Bash  │ │  File  │ │  Web   │ │   MCP    │          │
│   │  Tool  │ │  Tools │ │  Tools │ │  Bridge  │          │
│   └────────┘ └────────┘ └────────┘ └──────────┘          │
├────────────────────────────────────────────────────────────┤
│                   Permission Layer                          │
│     ┌────────────┐ ┌──────────┐ ┌────────────┐           │
│     │  规则引擎   │ │ 分类器   │ │  Hook决策   │           │
│     │  Rules     │ │Classifier│ │  Agent     │           │
│     └────────────┘ └──────────┘ └────────────┘           │
├────────────────────────────────────────────────────────────┤
│                    State & Config                           │
│  ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐   │
│  │ AppState │ │ Settings  │ │ CLAUDE.md│ │ Session  │   │
│  │  Store   │ │ Hierarchy │ │  Memory  │ │ Storage  │   │
│  └──────────┘ └───────────┘ └──────────┘ └──────────┘   │
├────────────────────────────────────────────────────────────┤
│                   External Services                         │
│   ┌──────────────┐ ┌────────────┐ ┌─────────────────┐    │
│   │ Anthropic API│ │ MCP Server │ │ GrowthBook/     │    │
│   │ (Messages)   │ │ (Stdio/SSE)│ │ Feature Flags   │    │
│   └──────────────┘ └────────────┘ └─────────────────┘    │
└────────────────────────────────────────────────────────────┘
```

### 3.2 启动流程详解

#### 阶段一：快速路径检测 (cli.tsx)

```
┌─ CLI 入口 ─────────────────────────────────────────────────┐
│                                                            │
│  1. profileCheckpoint('main_tsx_entry')                    │
│  2. startMdmRawRead()        // 并行启动 MDM 读取          │
│  3. startKeychainPrefetch()  // 并行启动密钥链读取          │
│                                                            │
│  快速路径（零模块加载）:                                     │
│  ├─ --version        → 打印版本号，立即退出                  │
│  ├─ --dump-system-prompt → 输出系统提示词                   │
│  ├─ --chrome-native-host → Chrome 扩展模式                  │
│  ├─ --computer-use-mcp → Computer Use MCP 服务器            │
│  ├─ --daemon-worker  → 守护进程工作线程                      │
│  ├─ remote-control   → 远程控制桥接模式                      │
│  ├─ daemon           → 守护进程模式                          │
│  └─ ps / logs        → 后台会话管理                          │
│                                                            │
│  ↓ 无匹配 → 加载完整初始化                                   │
└────────────────────────────────────────────────────────────┘
```

#### 阶段二：完整初始化 (main.tsx)

```
┌─ Main 初始化 ──────────────────────────────────────────────┐
│                                                            │
│  Lines 1-200:    导入 & 性能打点                            │
│  Lines 200-500:  OAuth 配置 & 安全设置加载                   │
│  Lines 500-1000: GrowthBook 功能标志初始化                   │
│  Lines 1000-2000: Commander.js 命令行解析                   │
│  Lines 2000-3700: 模式判断 (交互/无头/Print)                │
│  Lines 3700+:    REPL 启动 & 交互循环                       │
│                                                            │
│  并行预取（首屏渲染后）:                                     │
│  ├─ initUser()                  // 用户资料                  │
│  ├─ getUserContext()            // CLAUDE.md 文件            │
│  ├─ getRelevantTips()           // 功能提示                  │
│  ├─ countFilesRoundedRg()       // 项目文件统计              │
│  ├─ refreshModelCapabilities()  // 模型能力矩阵              │
│  ├─ settingsChangeDetector.initialize() // 配置监听          │
│  └─ skillChangeDetector.initialize()    // 技能热更新        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 3.3 核心查询循环 (query.ts — 系统心脏)

这是整个 Claude Code 最核心的文件，1,729 行的 `query.ts` 实现了**完整的智能体循环**：

```typescript
// 核心状态定义
type State = {
  messages: Message[]                     // 对话历史
  toolUseContext: ToolUseContext          // 工具执行上下文
  autoCompactTracking?: AutoCompactTrackingState
  maxOutputTokensRecoveryCount: number    // API 恢复循环计数器
  hasAttemptedReactiveCompact: boolean    // 是否已尝试响应式压缩
  pendingToolUseSummary?: Promise<...>    // 待处理的工具摘要
  stopHookActive?: boolean               // 停止 Hook 是否激活
  turnCount: number                      // 当前轮次
  transition?: Continue                   // 恢复路径
}

// 主生成器函数
async function* query(params: QueryParams)
  // yields: StreamEvent | RequestStartEvent | Message | TombstoneMessage
  // returns: Terminal { ... }

async function* queryLoop(params, consumedCommandUuids)
  // 实现: 核心智能体循环
  // 处理: 工具执行、压缩、恢复
```

#### 单轮循环执行流程

```
用户输入
    │
    ▼
┌──────────────────────────────────────────┐
│ 1. 组装系统提示词 + 上下文                 │
│    ├─ 基础系统指令 (角色、能力)            │
│    ├─ 工具定义 + MCP 工具提示              │
│    ├─ 权限规则 (auto 模式)                │
│    ├─ 内存文件 (CLAUDE.md)               │
│    └─ 用户/系统上下文                     │
├──────────────────────────────────────────┤
│ 2. 规范化消息 → API 格式                  │
├──────────────────────────────────────────┤
│ 3. 调用 claude.ts 流式 API               │
│    ├─ text_delta 事件                    │
│    ├─ content_block_start/delta/stop     │
│    └─ 收集为完整消息                      │
├──────────────────────────────────────────┤
│ 4. 检查 stop_reason                      │
│    ├─ "end_turn" → 结束循环              │
│    ├─ "tool_use" → 进入工具执行           │
│    └─ "max_tokens" → 恢复机制            │
├──────────────────────────────────────────┤
│ 5. 工具执行 (StreamingToolExecutor)       │
│    ├─ 提取 tool_use 块                   │
│    ├─ 权限检查 (canUseTool)              │
│    ├─ 并行执行工具                        │
│    ├─ 流式进度更新                        │
│    └─ 收集 tool_result 块                │
├──────────────────────────────────────────┤
│ 6. 上下文管理                            │
│    ├─ 检查 Token 预算                    │
│    ├─ 触发自动压缩?                      │
│    └─ 更新消息数组                        │
├──────────────────────────────────────────┤
│ 7. 继续点判断 (7 个分支)                  │
│    ├─ 正常工具循环继续                    │
│    ├─ Token 预算超限 → 自动继续           │
│    ├─ max_output_tokens → 升级重试        │
│    ├─ 响应式压缩触发                      │
│    ├─ Stop Hook 需要执行                 │
│    ├─ 上下文坍缩触发                      │
│    └─ 任务预算耗尽                        │
└──────────────────────────────────────────┘
    │
    ▼
  回到步骤 1（或结束）
```

### 3.4 三种执行模式

| 模式 | 触发方式 | 特点 |
|------|---------|------|
| **交互式 REPL** | 默认启动 | 完整 TUI、持久会话、可中断 |
| **无头/Print 模式** | `-p` / `--print` 标志 | 单次查询、输出到 stdout |
| **SDK/非交互模式** | `--init-only` | 最小初始化、无用户提示 |

---

## 第四部分：工具系统

### 4.1 工具接口定义 (Tool.ts)

```typescript
// 工具定义类型
type Tool = {
  name: string                              // 工具名称
  description: string                       // 给模型看的描述
  input_schema: ToolInputJSONSchema         // JSON Schema 输入规范
  isEnabled?: () => boolean                 // 是否启用
  isIncludedInSystemPrompt?: () => boolean  // 是否在系统提示中
  validateInput?: (input) => ValidationResult  // 输入验证
  expandedDescription?: string              // 扩展描述
  visibility?: 'public' | 'internal'        // 可见性
  mcpInfo?: { serverName; toolName }        // MCP 信息
}

// 工具执行上下文
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    agentDefinitions: AgentDefinitionsResult
    thinkingConfig: ThinkingConfig
    maxBudgetUsd?: number
    // ... 20+ 字段
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev) => AppState): void
}

// 工具结果类型
type ToolResult<Output> = {
  content: ContentBlockParam[]    // API 格式内容
  successMessage?: string         // 成功消息
  toolUseID?: string             // 工具使用 ID
  contentReplaced?: boolean       // 内容是否被替换
  suppressOutput?: boolean        // 是否隐藏输出
  // ... 15+ 更多字段
}
```

### 4.2 完整工具清单

#### 文件操作类 (6 个)

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| **FileRead** | 读取文件 | 支持图片/PDF/Jupyter，行号范围 |
| **FileWrite** | 创建/覆盖文件 | 必须先读取才能覆盖 |
| **FileEdit** | 精确字符串替换 | `old_string` → `new_string`，支持 `replace_all` |
| **Glob** | 文件模式匹配 | 支持 `**/*.ts` 等 glob 模式 |
| **Grep** | 内容搜索 | 基于 ripgrep，支持正则、上下文行 |
| **NotebookEdit** | Jupyter 编辑 | 编辑 `.ipynb` 单元格 |

#### 执行类 (5 个)

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| **Bash** | Shell 命令执行 | 沙箱支持、超时控制、后台运行 |
| **PowerShell** | Windows 命令 | Windows 平台专用 |
| **REPL** | 交互式 REPL | 内部专用 (ANT-only) |
| **Skill** | 技能调用 | 执行预定义/用户定义技能 |
| **Agent** | 智能体派生 | 创建子代理并行工作 |

#### 网络类 (3 个)

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| **WebFetch** | 获取网页内容 | URL 访问、内容提取 |
| **WebSearch** | 网页搜索 | 搜索引擎集成 |
| **WebBrowser** | 浏览器自动化 | Feature-gated，自动化操作 |

#### MCP 桥接类 (4 个)

| 工具 | 功能 | 关键特性 |
|------|------|---------|
| **MCPTool** | MCP 工具代理 | 桥接 MCP 服务器工具 |
| **ListMcpResources** | 列出 MCP 资源 | 发现可用资源 |
| **ReadMcpResource** | 读取 MCP 资源 | 获取资源内容 |
| **ListPeers** | 列出 UDS 对等体 | 内部通信 |

#### 任务管理类 (6 个)

| 工具 | 功能 |
|------|------|
| **TaskCreate** | 创建任务 |
| **TaskGet** | 获取任务详情 |
| **TaskUpdate** | 更新任务状态 |
| **TaskList** | 列出所有任务 |
| **TaskStop** | 停止任务 |
| **TaskOutput** | 获取任务输出 |

#### 系统类 (5 个)

| 工具 | 功能 |
|------|------|
| **AskUserQuestion** | 向用户提问 |
| **ToolSearch** | 搜索延迟加载的工具 |
| **EnterPlanMode** | 进入规划模式 |
| **ExitPlanMode** | 退出规划模式 |
| **ConfigTool** | 配置管理 |

### 4.3 工具注册机制 (tools.ts)

```typescript
// 动态构建可用工具列表
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    // Feature-gated 条件工具
    ...(SleepTool ? [SleepTool] : []),
    ...(cronTools ? cronTools : []),
    ...(RemoteTriggerTool ? [RemoteTriggerTool] : []),
    // ...
  ]
}

// 按权限规则过滤工具
export function filterToolsByDenyRules<T extends { name: string }>(
  tools: readonly T[],
  permissionContext: ToolPermissionContext
): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

### 4.4 工具执行流水线 (StreamingToolExecutor)

```
Claude API 返回 tool_use 块
         │
         ▼
┌─────────────────────────┐
│ 提取所有 tool_use 块     │
│ (可能有多个并行)          │
└──────────┬──────────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
┌─────────┐  ┌─────────┐
│ 权限检查 │  │ 权限检查 │   (并行)
│ Tool A  │  │ Tool B  │
└────┬────┘  └────┬────┘
     │            │
     ▼            ▼
┌─────────┐  ┌─────────┐
│ 执行     │  │ 执行     │   (并行，有并发控制)
│ Tool.A  │  │ Tool.B  │
│ .call() │  │ .call() │
└────┬────┘  └────┬────┘
     │            │
     ▼            ▼
┌─────────────────────────┐
│ 收集 tool_result 块     │
│ 追加到消息数组           │
└─────────────────────────┘
```

---

## 第五部分：权限与安全模型

### 5.1 权限模式

Claude Code 提供 **5 种外部权限模式** 和 **2 种内部权限模式**：

```
外部模式 (用户可选):
├── default         → 交互式提示 (默认)
├── acceptEdits     → 自动批准文件编辑
├── bypassPermissions → 危险：无提示全部批准
├── dontAsk         → 静默拒绝
└── plan            → 规划模式 (/plan 命令专用)

内部模式 (系统使用):
├── auto            → 分类器决策 (Feature-gated)
└── bubble          → 内部状态
```

### 5.2 权限决策流程

```
工具调用请求
     │
     ▼
┌─────────────────────┐
│ 1. 检查 Allow 规则  │ ← settings.json 中的 alwaysAllowRules
│    匹配? → 允许     │
└──────────┬──────────┘
           │ 不匹配
           ▼
┌─────────────────────┐
│ 2. 检查 Deny 规则   │ ← settings.json 中的 alwaysDenyRules
│    匹配? → 拒绝     │
└──────────┬──────────┘
           │ 不匹配
           ▼
┌─────────────────────┐
│ 3. 检查 Ask 规则    │ ← settings.json 中的 alwaysAskRules
│    匹配? → 提示     │
└──────────┬──────────┘
           │ 不匹配
           ▼
┌─────────────────────┐
│ 4. 检查权限模式      │
│    bypass → 允许     │
│    dontAsk → 拒绝    │
│    auto → 分类器     │
│    default → 提示    │
└──────────┬──────────┘
           │ auto 模式
           ▼
┌─────────────────────┐
│ 5. YOLO 分类器      │
│    safe → 允许       │
│    unsafe → 提示     │
│    unknown → 提示    │
└─────────────────────┘
```

### 5.3 权限规则格式

```json
{
  "permissions": {
    "rules": {
      "allow": [
        "Bash(prefix:npm test)",
        "Bash(prefix:git status)",
        "FileEdit",
        "FileRead",
        "Glob",
        "Grep"
      ],
      "deny": [
        "Bash(prefix:rm -rf /)",
        "Bash(prefix:sudo)"
      ]
    }
  }
}
```

**规则匹配语法**：
- `ToolName` — 匹配整个工具
- `ToolName(prefix:xxx)` — 匹配工具名 + 参数前缀
- `MCP:ServerName` — 匹配 MCP 服务器

### 5.4 YOLO 分类器 (Auto 模式核心)

```typescript
// yoloClassifier.ts — 分析命令安全性
classifyYoloAction(command: string, cwd: string)
  → 'safe' | 'unsafe' | 'unknown'

// 危险模式检测：
// ├─ 工作目录外的文件系统操作
// ├─ 凭证/环境变量暴露
// ├─ 系统管理命令
// ├─ 网络操作
// ├─ rm -rf, git reset --hard, git push --force
// ├─ drop, delete, truncate
// └─ sudo, --no-verify
```

### 5.5 沙箱支持

```typescript
// sandbox-adapter.ts — Bash 沙箱
SandboxManager.isSandboxingEnabled()
SandboxManager.areUnsandboxedCommandsAllowed()
SandboxManager.isAutoAllowBashIfSandboxedEnabled()

// 能力限制：
// ├─ 文件访问白名单/黑名单
// ├─ 网络限制
// └─ 环境变量过滤
```

---

## 第六部分：Hook 事件系统

### 6.1 Hook 事件类型

Claude Code 提供 **15+ 种 Hook 事件**，允许用户在关键节点注入自定义逻辑：

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| `SessionStart` | 会话开始 | 初始化环境、加载配置 |
| `UserPromptSubmit` | 用户输入提交前 | 输入验证、日志记录 |
| `PreToolUse` | 工具执行前 | 权限控制、参数修改 |
| `PostToolUse` | 工具执行后 | 结果处理、日志记录 |
| `PostToolUseFailure` | 工具执行失败 | 错误处理、告警 |
| `PermissionRequest` | 权限对话框前 | 自动决策 |
| `PermissionDenied` | 用户拒绝权限 | 记录拒绝原因 |
| `Notification` | 通知发送 | 自定义通知渠道 |
| `Setup` | 首次运行设置 | 环境准备 |
| `Elicitation` | URL 获取请求 | MCP URL 处理 |
| `CwdChanged` | 工作目录变更 | 自动重新加载配置 |
| `FileChanged` | 监视路径文件变更 | 热更新 |
| `SubagentStart` | 子代理启动 | 跟踪/限制子代理 |
| `WorktreeCreate` | 工作树创建 | 隔离环境管理 |

### 6.2 Hook 配置方式

在 `settings.json` 中配置三种类型的 Hook：

```json
{
  "hooks": [
    {
      "event": "PreToolUse",
      "matcher": "Bash",
      
      "handler": {
        "type": "script",
        "command": "~/.claude/hooks/check-bash.sh"
      }
    },
    {
      "event": "PostToolUse",
      
      "handler": {
        "type": "javascript",
        "code": "module.exports = async (input) => { /* ... */ }"
      }
    },
    {
      "event": "UserPromptSubmit",
      
      "handler": {
        "type": "agent",
        "prompt": "验证用户输入是否安全...",
        "timeout": 30,
        "model": "claude-sonnet-4-6"
      }
    }
  ]
}
```

### 6.3 Hook 响应协议

```typescript
// 同步 Hook 响应
{
  continue?: boolean,           // 是否继续执行
  suppressOutput?: boolean,     // 隐藏输出
  stopReason?: string,          // 停止原因
  decision?: 'approve' | 'block',  // 审批决策
  reason?: string,              // 决策原因
  systemMessage?: string,       // 警告消息
  
  hookSpecificOutput?: {
    hookEventName: string,
    permissionDecision?: 'allow' | 'deny' | 'ask',
    updatedInput?: Record<string, unknown>,  // 修改后的输入
    updatedMCPToolOutput?: unknown,
    watchPaths?: string[],
    additionalContext?: string
  }
}

// 异步 Hook 响应
{
  async: true,
  asyncTimeout?: number    // 超时秒数
}
```

### 6.4 Hook 实战场景

**场景 1：自动审批安全命令**
```json
{
  "event": "PreToolUse",
  "matcher": "Bash",
  "handler": {
    "type": "script",
    "command": "echo '{\"hookSpecificOutput\":{\"permissionDecision\":\"allow\"}}'"
  }
}
```

**场景 2：记录所有工具调用**
```json
{
  "event": "PostToolUse",
  "handler": {
    "type": "javascript",
    "code": "module.exports = async ({tool, input, output}) => { require('fs').appendFileSync('/tmp/claude-log.json', JSON.stringify({tool, input, ts: Date.now()}) + '\\n'); return {continue: true}; }"
  }
}
```

**场景 3：AI 审查代码变更**
```json
{
  "event": "PostToolUse",
  "matcher": "FileEdit",
  "handler": {
    "type": "agent",
    "prompt": "审查此文件编辑是否引入安全漏洞。检查 OWASP Top 10。如果发现问题，block 操作并说明原因。",
    "timeout": 60
  }
}
```

---

## 第七部分：MCP 协议集成

### 7.1 MCP 是什么

MCP（Model Context Protocol）是一个开放标准协议，允许 AI 应用与外部数据源和工具进行交互。Claude Code 内置了完整的 MCP 客户端实现。

### 7.2 支持的传输协议

| 协议 | 类型 | 适用场景 |
|------|------|---------|
| **Stdio** | 进程通信 | 本地工具服务器 |
| **SSE** | HTTP Server-Sent Events | 远程服务器 |
| **HTTP** | 请求/响应 | REST API 集成 |
| **WebSocket** | 双向通信 | 实时交互服务 |
| **SDK** | 进程内 | 内部集成 |
| **ClaudeAI Proxy** | 代理 | Claude.ai 云端 |

### 7.3 MCP 服务器配置

```json
// ~/.claude/mcp.json 或 settings.json 中的 mcpServers
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
    },
    "github": {
      "type": "sse",
      "url": "https://mcp.github.com/sse",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    },
    "database": {
      "type": "http",
      "url": "https://db-mcp.example.com/mcp",
      "oauth": {
        "clientId": "xxx",
        "authServerMetadataUrl": "https://auth.example.com/.well-known/openid-configuration"
      }
    }
  }
}
```

### 7.4 MCP 工具集成流程

```
启动时
  │
  ▼
┌─────────────────────────────────┐
│ 1. 发现：列出所有 MCP 工具       │
│    client.tools/list            │
├─────────────────────────────────┤
│ 2. 集成：包装为 Claude 工具      │
│    MCPTool ← { serverName,      │
│               toolName,          │
│               inputSchema }      │
├─────────────────────────────────┤
│ 3. 注册：添加到主工具注册表       │
│    getAllBaseTools() + mcpTools  │
├─────────────────────────────────┤
│ 4. 执行：模型调用时路由到 MCP    │
│    MCPTool.call(input)          │
│    → client.tools/call          │
│    → 反序列化结果                │
│    → 返回 ToolResult             │
├─────────────────────────────────┤
│ 5. 流式：支持服务端推送响应       │
│    SSE/WebSocket 持续接收       │
└─────────────────────────────────┘
```

### 7.5 MCP 资源预取

```typescript
// 异步预取所有 MCP 资源
prefetchAllMcpResources(mcpClients: MCPServerConnection[]): Promise<void>
  // 并行化 resource/list 调用
  // 缓存结果供后续访问
  // 优雅处理超时
```

---

## 第八部分：配置体系

### 8.1 四级配置层级

```
优先级从高到低：

┌─────────────────────────────────────────┐
│ 1. 受管设置 (最高优先级)                  │
│    /etc/claude-code/settings.json       │
│    macOS MDM 策略                        │
│    远程 API 轮询控制                      │
├─────────────────────────────────────────┤
│ 2. 用户设置                              │
│    ~/.claude/settings.json              │
│    全局默认值                             │
├─────────────────────────────────────────┤
│ 3. 项目设置                              │
│    CLAUDE.md (项目根目录)                 │
│    .claude/CLAUDE.md                    │
│    .claude/rules/*.md                   │
├─────────────────────────────────────────┤
│ 4. 运行时设置 (最低优先级)                │
│    CLI 标志 (--permission-mode 等)       │
│    环境变量 (CLAUDE_CODE_* 前缀)         │
└─────────────────────────────────────────┘
```

### 8.2 settings.json 完整结构

```typescript
type GlobalConfig = {
  // ── 通用 ──
  numStartups: number
  userID?: string
  installMethod?: 'local' | 'native' | 'global' | 'unknown'
  hasCompletedOnboarding?: boolean
  
  // ── 权限 ──
  theme: ThemeSetting
  permissions?: {
    defaultMode: PermissionMode
    rules?: {
      allow?: string[]     // 如 'Bash(prefix:rm)', 'FileEdit'
      deny?: string[]
      ask?: string[]
    }
  }
  
  // ── API & 认证 ──
  primaryApiKey?: string
  customApiKeyResponses?: {
    approved?: string[]
    rejected?: string[]
  }
  
  // ── 环境 ──
  env: { [key: string]: string }
  
  // ── 功能开关 ──
  autoCompactEnabled: boolean
  verbose: boolean
  
  // ── 通知 ──
  preferredNotifChannel: 'system' | 'email' | 'slack'
  
  // ── OAuth ──
  oauthAccount?: {
    accountUuid: string
    emailAddress: string
    organizationUuid?: string
    displayName?: string
  }
  
  // ── MCP 服务器 ──
  mcpServers?: Record<string, McpServerConfig>
  
  // ── Hook 配置 ──
  hooks?: HookConfig[]
  
  // ── 项目特定 ──
  projects?: Record<string, ProjectConfig>
}
```

### 8.3 CLAUDE.md 内存系统

#### 加载层级

```
1. 受管内存       /etc/claude-code/CLAUDE.md
2. 用户内存       ~/.claude/CLAUDE.md
3. 项目内存       CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md
4. 本地内存       CLAUDE.local.md

注：靠近 cwd 的文件优先级更高
最大字符数限制：40,000
```

#### @include 指令

```markdown
# CLAUDE.md

@path              # 相对路径
@./rel/path        # 相对路径
@~/home/path       # Home 目录
@/abs/path         # 绝对路径
```

特性：
- 循环引用检测与防护
- 仅加载文本文件
- 静默忽略缺失文件
- 字符数限制防止过大文件

### 8.4 环境变量

| 变量 | 作用 |
|------|------|
| `CLAUDE_CODE_SIMPLE` | 简化模式 |
| `ANTHROPIC_API_KEY` | API 密钥 |
| `CLAUDE_CODE_MAX_BUDGET` | 最大预算 |
| `CLAUDE_CODE_PERMISSION_MODE` | 权限模式 |
| `CLAUDE_CODE_VERBOSE` | 详细输出 |

---

## 第九部分：上下文管理与压缩

### 9.1 Token 预算体系

```typescript
// 核心类型
type TokenBudget = {
  maxContextTokens: number       // 模型最大上下文
  warningThreshold: number       // 警告阈值 (80%)
  compactThreshold: number       // 压缩阈值
  estimatedCurrentTokens: number // 当前估算 Token 数
}

// 预算检查
calculateTokenWarningState(messages, limits) → {
  isNearLimit: boolean      // 接近上限
  isAtLimit: boolean        // 达到上限
  shouldCompact: boolean    // 应该压缩
  estimatedTokens: number   // 估算 Token 数
}
```

### 9.2 自动压缩机制

```
触发条件：
├── 手动：/compact 命令
├── 自动：上下文 > 80% 限制
└── 响应式：max_output_tokens 恢复后

压缩流程：
┌──────────────────────────────────────┐
│ 1. 计算压缩点 (compactionIndex)      │
│    保留最近的关键消息                  │
├──────────────────────────────────────┤
│ 2. 生成摘要                          │
│    对旧消息进行 LLM 总结              │
├──────────────────────────────────────┤
│ 3. 构建新消息数组                     │
│    [system_prompt,                   │
│     attachment(summary),             │
│     remaining_messages]              │
└──────────────────────────────────────┘

Snip 压缩 (Feature-gated):
├── 替换中间消息为极简摘要
├── 仅保留边界处的工具调用和结果
└── 大幅减少 Token 消耗
```

### 9.3 Prompt 缓存优化

Claude Code 使用**多级 Prompt 缓存**来减少重复 Token 消耗：

```
┌─ 系统提示词结构 ──────────────────────────┐
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ 静态部分 (跨组织可缓存)              │  │
│  │ ├─ 基础角色指令                     │  │
│  │ ├─ 工具定义                         │  │
│  │ ├─ 通用指引                         │  │
│  │ └─ cache_control: ephemeral         │  │
│  ├────── 动态分界符 ──────────────────┤  │
│  │ __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__  │  │
│  ├────────────────────────────────────┤  │
│  │ 动态部分 (每次会话不同)              │  │
│  │ ├─ 用户上下文 (CLAUDE.md)           │  │
│  │ ├─ Git 状态                         │  │
│  │ ├─ 会话特定设置                     │  │
│  │ └─ 功能提示                         │  │
│  └────────────────────────────────────┘  │
│                                          │
│  缓存效果：长对话中减少 ~90% 重复 Token   │
└──────────────────────────────────────────┘
```

### 9.4 Section 缓存机制

```typescript
// 常规 section：计算一次，整个会话复用
systemPromptSection(name, compute): SystemPromptSection

// 不可缓存 section：每轮重新计算
DANGEROUS_uncachedSystemPromptSection(name, compute, reason)

// 缓存在 /clear 和 /compact 时清除
```

---

## 第十部分：Agent 智能体系统

### 10.1 Agent 定义

```typescript
type AgentDefinition = {
  agentType: string          // 代理类型标识
  name: string               // 显示名称
  description: string        // 描述
  whenToUse: string          // 使用场景说明
  
  // 能力配置
  tools?: string[]           // 允许使用的工具 (白名单)
  disallowedTools?: string[] // 禁止使用的工具 (黑名单)
  
  // 人格设定
  getSystemPrompt(ctx?): string  // 系统提示词
  memory?: string                // 记忆类型
  
  // 执行配置
  model?: string                 // 使用的模型
  thinkingConfig?: ThinkingConfig // 思考配置
}
```

### 10.2 内置 Agent 类型

| Agent | 功能 | 工具限制 |
|-------|------|---------|
| **general-purpose** | 通用自主代理 | 所有工具 |
| **Plan** | 架构规划代理 | 只读工具，不能编辑/写入 |
| **Explore** | 代码库探索代理 | 只读搜索工具 |
| **claude-code-guide** | 帮助/教程代理 | Glob, Grep, Read, WebFetch, WebSearch |
| **code-reviewer** | 代码审查代理 | 只读工具 |

### 10.3 Agent 执行模型

```
主代理
  │
  ├── Agent(subagent_type: "Explore")
  │     └── 独立上下文，只读工具
  │         返回研究结果
  │
  ├── Agent(subagent_type: "general-purpose")
  │     └── 完整上下文，所有工具
  │         自主完成复杂任务
  │
  └── Agent(isolation: "worktree")
        └── Git worktree 隔离
            独立仓库副本
            完成后合并或清理
```

### 10.4 Agent 提示词最佳实践

源码中内嵌的提示指南：

> **像给一个刚走进房间的聪明同事做简报** — 他没有看过这次对话，不知道你尝试过什么，不了解为什么这个任务重要。
> 
> - 解释你想要完成什么以及为什么
> - 描述你已经学到或排除了什么
> - 给出足够的上下文让代理可以做判断
> - 如果需要简短回应，明确说明

### 10.5 自定义 Agent

用户可以在 `~/.claude/agents/` 目录下创建自定义 Agent：

```json
// ~/.claude/agents/security-reviewer.json
{
  "agentType": "security-reviewer",
  "name": "Security Reviewer",
  "description": "专门审查代码安全性的代理",
  "whenToUse": "当需要安全审查时",
  "tools": ["FileRead", "Glob", "Grep", "Bash"],
  "disallowedTools": ["FileEdit", "FileWrite"],
  "systemPrompt": "你是一个安全专家，专门审查代码中的安全漏洞..."
}
```

---

## 第十一部分：System Prompt 构造机制

### 11.1 Prompt 构建流程

```typescript
buildEffectiveSystemPrompt({
  mainThreadAgentDefinition?: AgentDefinition
  toolUseContext: ToolUseContext
  customSystemPrompt?: string
  defaultSystemPrompt: string[]
  appendSystemPrompt?: string
  overrideSystemPrompt?: string
})
```

**优先级链**：
1. **Override prompt** — 替换所有（如 /loop 模式）
2. **Coordinator 模式 prompt** — 多智能体协调
3. **Agent 系统 prompt** — 如果设置了 mainThreadAgentDefinition
4. **Custom prompt** — 用户通过 `--system-prompt` 指定
5. **Default prompt** — 从 `prompts.ts` 生成

### 11.2 默认系统 Prompt 各 Section

| Section | 功能 | 内容 |
|---------|------|------|
| `getSimpleIntroSection()` | 角色介绍 | "You are an interactive agent..." |
| `getSimpleSystemSection()` | 系统说明 | 工具、权限、标签、Hook、压缩 |
| `getSimpleDoingTasksSection()` | 任务指导 | 软件工程最佳实践、代码风格、安全 |
| `getActionsSection()` | 操作指导 | 可逆性、影响范围、确认模式 |
| `getUsingYourToolsSection()` | 工具使用 | 文件工具、Bash、搜索工具指引 |
| `getMcpInstructionsSection()` | MCP 指引 | MCP 服务器能力和约束 |
| `getLanguageSection()` | 语言偏好 | 注入用户语言偏好 |
| `getOutputStyleSection()` | 输出格式 | markdown/html 格式设置 |

### 11.3 关键 Prompt 设计理念（从源码提取）

以下是从 `prompts.ts` 中提取的核心设计理念：

**任务执行原则**：
```
- 不要添加未被要求的功能、重构或"改进"
- 不要为不可能的场景添加错误处理
- 不要为一次性操作创建辅助函数/工具
- 三行相似代码优于过早抽象
- 修复 Bug 不需要顺便清理周围代码
```

**操作安全原则**：
```
- 仔细考虑操作的可逆性和影响范围
- 本地、可逆操作可以自由执行
- 难以逆转、影响共享系统或有风险的操作要先确认
- 用户批准一次操作并不意味着在所有上下文中都批准
- 遇到障碍时不要用破坏性操作作为捷径
```

**工具使用原则**：
```
- 读取文件用 Read，不用 cat/head/tail
- 编辑文件用 Edit，不用 sed/awk
- 搜索文件用 Glob，不用 find/ls
- 搜索内容用 Grep，不用 grep/rg
- 创建文件用 Write，不用 echo/cat heredoc
```

---

## 第十二部分：API 通信与重试机制

### 12.1 API 客户端 (claude.ts)

```typescript
// 核心 API 调用函数
export async function callApi(config: ApiConfig): Promise<ApiResponse>

// 职责：
// ├─ 消息序列化
// ├─ 系统提示词拆分 (可缓存部分 + 动态部分)
// ├─ 工具定义生成
// ├─ Anthropic SDK 集成
// ├─ 流式响应处理
// ├─ 错误分类
// └─ 重试协调
```

### 12.2 指数退避重试策略

```
尝试 1: 立即执行
尝试 2: 等待 1s
尝试 3: 等待 2s
尝试 4: 等待 4s
尝试 5: 等待 8s (最多 5 次重试)

可重试错误：
├── 429 (速率限制)
├── 500 (服务器错误)
├── 502, 503, 504 (网关错误)
└── 临时网络错误

不可重试错误：
├── 401 (认证失败)
├── 403 (权限不足)
├── 400 (请求错误)
└── 413 (负载过大)
```

### 12.3 max_output_tokens 恢复机制

当 API 返回 `stop_reason: "max_tokens"` 时：

```
检测到 max_output_tokens
     │
     ▼
┌─────────────────────────┐
│ 递增恢复计数器           │
│ maxOutputTokensRecovery │
│ Count++                  │
├─────────────────────────┤
│ 升级 Token 预算          │
│ 请求更高输出限制          │
├─────────────────────────┤
│ 如果恢复次数过多：        │
│ 尝试响应式压缩            │
│ hasAttemptedReactive     │
│ Compact = true           │
├─────────────────────────┤
│ 重新进入查询循环          │
└─────────────────────────┘
```

---

## 第十三部分：构建系统与编译优化

### 13.1 五阶段构建流程

```
阶段 1: 复制
   src/ → build-src/ (保留原始文件)

阶段 2: 转换
   ├─ feature('X') → false (死代码标记)
   ├─ MACRO.VERSION → '2.1.88' (版本注入)
   ├─ import from 'bun:bundle' → 存根 (Bun 特性)
   └─ 移除纯类型导入

阶段 3: 入口包装
   创建 build-src/entry.ts → 指向 src/entrypoints/cli.tsx

阶段 4: 迭代存根 + 打包
   Round N:
   ├─ 1. esbuild 打包 entry.ts
   ├─ 2. 收集缺失模块
   ├─ 3. 为缺失模块创建存根
   └─ 4. 重复直到成功

阶段 5: 输出
   生成最终 cli.js + cli.js.map
```

### 13.2 Feature Gate 编译时消除

```typescript
// 源码中
if (feature('DAEMON')) {
  require('./daemon/main.js')
}

// 编译后 (公开发布版)
if (false) {
  // 整个分支被 esbuild 消除 (dead code elimination)
}
```

这种机制导致 **108 个模块** 在公开版本中被消除。

### 13.3 被消除的内部功能 (108 个模块)

**内部基础设施 (~70 个)**：
- `daemon/main.js` — 后台监控器
- `coordinator/workerAgent.js` — 多智能体协调
- `assistant/` — KAIROS 自主模式
- `bridge/peerSessions.js` — 远程控制对等管理
- `skillSearch/` — 远程技能发现
- `sessionTranscript/sessionTranscript.js` — 分类器后端

**Feature-gated 工具 (~20 个)**：
- `REPLTool` — 交互式 REPL
- `MonitorTool` — MCP 监控
- `WebBrowserTool` — 浏览器自动化
- `WorkflowTool` — 工作流执行
- `SnipTool` — 上下文剪切
- `SleepTool` — 延迟/调度

---

## 第十四部分：高效使用技巧（入门篇）

### 14.1 基础配置优化

#### 创建 CLAUDE.md 项目文件

在项目根目录创建 `CLAUDE.md`，这是提升 Claude Code 效率的**最重要一步**：

```markdown
# 项目说明

## 技术栈
- Next.js 14 + TypeScript
- Prisma + PostgreSQL
- Tailwind CSS

## 开发命令
- `npm run dev` — 启动开发服务器
- `npm test` — 运行测试
- `npm run lint` — 代码检查
- `npm run build` — 构建项目

## 代码规范
- 使用函数组件 + Hooks
- 状态管理用 Zustand
- API 路由在 /app/api/ 目录
- 组件放在 /components/ 目录

## 数据库
- Schema 在 prisma/schema.prisma
- 迁移命令: npx prisma migrate dev

## 重要注意事项
- 不要修改 /lib/legacy/ 下的文件，那是旧代码
- 测试必须通过才能提交
- 使用中文注释
```

**为什么重要**：源码分析显示，CLAUDE.md 的内容会被注入到**每一轮**的系统提示中，直接影响模型的行为。

#### 设置权限规则

```json
// ~/.claude/settings.json
{
  "permissions": {
    "rules": {
      "allow": [
        "FileRead",
        "FileEdit",
        "Glob",
        "Grep",
        "Bash(prefix:npm test)",
        "Bash(prefix:npm run lint)",
        "Bash(prefix:git status)",
        "Bash(prefix:git diff)",
        "Bash(prefix:git log)"
      ],
      "deny": [
        "Bash(prefix:rm -rf)",
        "Bash(prefix:sudo)",
        "Bash(prefix:git push --force)"
      ]
    }
  }
}
```

### 14.2 常用斜杠命令速查

| 命令 | 功能 | 使用场景 |
|------|------|---------|
| `/help` | 显示帮助信息 | 不确定功能时 |
| `/compact` | 压缩上下文 | 对话过长时 |
| `/clear` | 清除对话历史 | 开始新话题时 |
| `/status` | 显示会话状态 | 检查 Token 使用 |
| `/cost` | 显示费用统计 | 追踪 API 支出 |
| `/model` | 切换模型 | 调整速度/质量 |
| `/commit` | Git 提交 | 自动生成 commit message |
| `/review` | 代码审查 | 审查 PR 或更改 |
| `/mcp` | MCP 管理 | 添加/删除 MCP 服务器 |
| `/config` | 配置管理 | 修改设置 |
| `/resume` | 恢复会话 | 继续之前的工作 |
| `/tasks` | 任务管理 | 查看/管理任务列表 |
| `/permissions` | 权限管理 | 调整权限规则 |
| `/hooks` | Hook 管理 | 查看/配置 Hook |

### 14.3 基础使用模式

#### 模式 1：探索式编程

```
你: 帮我了解这个项目的结构，重点关注 API 路由

Claude Code 会：
├── 使用 Glob 搜索文件结构
├── 使用 Grep 查找 API 定义
├── 使用 Read 阅读关键文件
└── 生成项目结构概览
```

#### 模式 2：Bug 修复

```
你: 用户登录后页面白屏，请排查并修复

Claude Code 会：
├── 使用 Grep 搜索登录相关代码
├── 使用 Read 阅读相关组件
├── 使用 Bash 运行测试
├── 使用 Edit 修复 Bug
└── 使用 Bash 验证修复
```

#### 模式 3：功能开发

```
你: 添加用户头像上传功能，支持裁剪

Claude Code 会：
├── 分析现有代码结构
├── 创建必要的新文件
├── 修改相关组件
├── 编写 API 路由
├── 添加测试
└── 运行测试验证
```

### 14.4 Shell 命令集成技巧

在 Claude Code 提示中使用 `!` 前缀运行交互式命令：

```
你: ! gcloud auth login

# Claude Code 会在当前 Shell 中执行该命令
# 输出直接进入对话上下文
```

---

## 第十五部分：高效使用技巧（进阶篇）

### 15.1 利用 Agent 系统并行工作

从源码分析可知，Agent 工具支持并行派生多个子代理。利用这一特性：

```
你: 请同时帮我做以下工作：
    1. 审查 src/auth/ 目录的安全性
    2. 为 src/api/users.ts 编写单元测试
    3. 检查整个项目中是否有 N+1 查询问题

# Claude Code 会并行启动 3 个子代理
# 每个子代理独立工作，互不干扰
# 结果汇总后一起返回
```

**关键发现**：源码中 `AgentTool.tsx` 显示，子代理可以设置 `isolation: "worktree"` 在 Git worktree 中隔离运行，避免文件冲突。

### 15.2 深度利用 CLAUDE.md

#### 多层级 CLAUDE.md

```
~/.claude/CLAUDE.md                    # 全局偏好
├── "我是高级 TypeScript 开发者"
├── "偏好函数式编程风格"
└── "使用中文回复"

项目根目录/CLAUDE.md                    # 项目级配置
├── "使用 pnpm 而不是 npm"
├── @./docs/architecture.md            # 引用架构文档
└── @./docs/api-spec.md               # 引用 API 规范

项目根目录/.claude/rules/security.md    # 安全规则
├── "所有用户输入必须转义"
└── "不使用 eval() 或 innerHTML"

项目根目录/.claude/rules/testing.md     # 测试规则
├── "所有新功能必须有单元测试"
└── "测试覆盖率不低于 80%"
```

#### @include 实战

```markdown
# CLAUDE.md

## 项目文档
@./README.md
@./docs/CONTRIBUTING.md
@./docs/API.md

## 代码规范
@./.eslintrc.json
@./tsconfig.json
```

### 15.3 MCP 服务器扩展能力

#### 常用 MCP 服务器配置

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "${DATABASE_URL}" }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": { "SLACK_TOKEN": "${SLACK_TOKEN}" }
    }
  }
}
```

### 15.4 Hook 自动化工作流

#### 自动格式化代码

```json
{
  "hooks": [
    {
      "event": "PostToolUse",
      "matcher": "FileEdit|FileWrite",
      "handler": {
        "type": "script",
        "command": "npx prettier --write \"$TOOL_INPUT_FILE_PATH\" 2>/dev/null; echo '{\"continue\":true}'"
      }
    }
  ]
}
```

#### 自动运行测试

```json
{
  "hooks": [
    {
      "event": "PostToolUse",
      "matcher": "FileEdit",
      "handler": {
        "type": "script",
        "command": "if echo \"$TOOL_INPUT_FILE_PATH\" | grep -q '\\.test\\.'; then npm test -- --bail 2>&1 | tail -5; fi; echo '{\"continue\":true}'"
      }
    }
  ]
}
```

### 15.5 会话管理技巧

#### 利用 /resume 恢复工作

```
# 查看之前的会话
你: /resume

# 恢复特定会话
你: /resume [session-id]

# Claude Code 会加载完整的对话历史和上下文
```

#### 控制上下文大小

```
# 当对话过长时
你: /compact

# 更激进的压缩
你: /compact --aggressive

# 开始新话题时清除
你: /clear
```

### 15.6 Prompt 工程技巧

从源码分析 System Prompt 后，总结以下高效 Prompt 模式：

#### 精确指令模式

```
❌ "帮我改改这个文件"
✅ "修改 src/api/users.ts 的 getUser 函数，
    将数据库查询从 findFirst 改为 findUnique，
    因为 ID 是唯一的"
```

#### 上下文预加载模式

```
你: 先读取 src/models/user.ts 和 src/api/users.ts，
    然后告诉我用户数据流是如何从 API 到数据库的

# 利用了源码中 Read 工具比 Bash(cat) 更高效的特性
```

#### 多步任务分解模式

```
你: 我需要实现用户导出功能。请按以下步骤：
    1. 先分析现有的数据模型
    2. 设计 API 接口
    3. 实现后端逻辑
    4. 编写测试
    每完成一步，先告诉我你的方案，等我确认后再继续
```

---

## 第十六部分：高效使用技巧（精通篇）

### 16.1 深度理解权限系统

从源码中发现的**权限决策原因类型**（PermissionDecisionReason）：

```typescript
type PermissionDecisionReason = 
  | { type: 'classifier' }    // Auto 模式分类器决策
  | { type: 'hook' }          // Hook 决策
  | { type: 'rule' }          // 权限规则匹配
  | { type: 'mode' }          // 权限模式决定
  | { type: 'safetyCheck' }   // 安全检查
  | { type: 'sandboxOverride' } // 沙箱覆盖
  | { type: 'workingDir' }    // 工作目录检查
  | { type: 'asyncAgent' }    // 异步代理
```

**实际应用**：理解这些决策类型后，可以精确配置规则来优化审批流程。

### 16.2 Token 预算管理策略

```
策略 1：预算感知编程
├── 大型项目中优先使用 Glob + Grep 而不是全文 Read
├── 给出精确的行号范围：Read(file, offset=100, limit=50)
├── 使用 /compact 在关键节点压缩上下文
└── 复杂任务拆分为多个小会话

策略 2：理解 Auto-Compact 触发
├── 上下文达到模型限制的 80% 时自动触发
├── 压缩会丢失部分早期上下文细节
├── 在重要上下文被压缩前使用 /compact 手动控制
└── 监控 /status 或 /cost 了解当前 Token 使用

策略 3：Prompt Cache 利用
├── 保持 CLAUDE.md 内容稳定（缓存友好）
├── 长对话复用系统提示的缓存部分
└── 避免频繁修改系统配置导致缓存失效
```

### 16.3 构建复杂 Hook 管道

#### Agent Hook — AI 驱动的代码审查

```json
{
  "hooks": [
    {
      "event": "PreToolUse",
      "matcher": "FileEdit",
      "handler": {
        "type": "agent",
        "prompt": "审查即将进行的文件编辑。检查：\n1. 是否引入安全漏洞（XSS、SQL注入等）\n2. 是否遵循项目编码规范\n3. 是否有明显的逻辑错误\n\n如果发现严重问题，返回 {decision: 'block', reason: '...'}。\n否则返回 {decision: 'approve'}。",
        "timeout": 30,
        "model": "claude-haiku-4-5-20251001"
      }
    }
  ]
}
```

#### 多阶段验证管道

```json
{
  "hooks": [
    {
      "event": "PostToolUse",
      "matcher": "FileEdit",
      "handler": {
        "type": "script",
        "command": "bash -c 'npx eslint --fix \"$TOOL_INPUT_FILE_PATH\" 2>&1; npx prettier --write \"$TOOL_INPUT_FILE_PATH\" 2>&1; echo \"{\\\"continue\\\":true}\"'"
      }
    },
    {
      "event": "PostToolUse",
      "matcher": "Bash",
      "handler": {
        "type": "javascript",
        "code": "module.exports = async ({input}) => { if (input.command?.includes('test') && input.exitCode !== 0) { return {continue: true, systemMessage: '⚠ 测试失败，请修复后重试'}; } return {continue: true}; }"
      }
    }
  ]
}
```

### 16.4 利用 Worktree 隔离实验

源码中发现的 `worktree` 隔离模式：

```
你: 请在隔离环境中尝试将 ORM 从 Prisma 迁移到 Drizzle，
    不要影响当前代码

# Claude Code 会：
# 1. 创建 Git worktree (独立仓库副本)
# 2. 在 worktree 中执行所有修改
# 3. 如果成功，返回 worktree 路径和分支名
# 4. 如果失败，自动清理 worktree
```

### 16.5 SDK 模式集成

```typescript
// 在自己的脚本中调用 Claude Code
import { QueryEngine } from '@anthropic-ai/claude-code';

const engine = new QueryEngine({
  cwd: '/path/to/project',
  tools: getAllBaseTools(),
  // ... 配置
});

// 提交查询并获取流式结果
for await (const event of engine.submitMessage('分析项目依赖安全性')) {
  if (event.type === 'message') {
    console.log(event.content);
  }
}
```

### 16.6 性能优化技巧

| 技巧 | 原理 | 效果 |
|------|------|------|
| 预填 CLAUDE.md | 注入系统提示，减少探索开销 | -30% Token |
| 使用具体文件路径 | 避免 Glob 搜索 | 更快响应 |
| 分批处理大任务 | 利用 /compact 控制上下文 | 更稳定 |
| 配置 allow 规则 | 减少权限提示交互 | 更流畅 |
| 使用 Agent 并行 | 子代理并行独立工作 | 更高效 |
| 利用 MCP 工具 | 扩展工具集 | 更强能力 |

### 16.7 调试与诊断

```
# 开启详细模式
在 settings.json 中设置 "verbose": true

# 查看会话状态
/status

# 查看 Token 使用和费用
/cost

# 分析上下文窗口使用
# (源码中的 analyzeContext.ts 提供此功能)

# 查看当前配置
/config

# 诊断问题
/doctor
```

---

## 第十七部分：内部隐藏功能与 Feature Gate

### 17.1 Feature Gate 机制

源码使用 `feature('FEATURE_NAME')` 进行编译时条件编译：

```typescript
// 公开版本中：feature('X') → false
// 内部版本中：feature('X') → true (Bun 编译时求值)
```

### 17.2 已知的 Feature Gate 列表

| Feature Gate | 功能 | 状态 |
|-------------|------|------|
| `DAEMON` | 后台守护进程 | 内部 |
| `BRIDGE_MODE` | 远程控制桥接 | 内部 |
| `BG_SESSIONS` | 后台会话 | 内部 |
| `COORDINATOR_MODE` | 多智能体协调 | 内部 |
| `KAIROS` | 全自主助手模式 | 内部 |
| `PROACTIVE` | 主动代理 | 内部 |
| `TRANSCRIPT_CLASSIFIER` | 分类器 | 内部 |
| `AGENT_TRIGGERS` | 定时触发器 | 部分开放 |
| `AGENT_TRIGGERS_REMOTE` | 远程触发器 | 内部 |
| `DIRECT_CONNECT` | 直接连接 | 内部 |
| `SSH_REMOTE` | SSH 远程 | 内部 |
| `HISTORY_SNIP` | 历史剪切 | 内部 |
| `BREAK_CACHE_COMMAND` | 缓存清除命令 | 内部 |
| `CHICAGO_MCP` | Computer Use MCP | 内部 |
| `DUMP_SYSTEM_PROMPT` | 输出系统提示词 | 内部 |
| `WORKFLOW_SCRIPTS` | 工作流脚本 | 内部 |

### 17.3 内部未公开的工具

| 工具 | 功能 | 限制 |
|------|------|------|
| `REPLTool` | 交互式 REPL | ANT-only |
| `MonitorTool` | MCP 监控 | Feature-gated |
| `WebBrowserTool` | 浏览器自动化 | Feature-gated |
| `WorkflowTool` | 工作流执行 | Feature-gated |
| `SnipTool` | 上下文剪切 | Feature-gated |
| `SleepTool` | 延迟/调度 | Feature-gated |
| `TerminalTool` | 终端控制 | Feature-gated |
| `TeamCreateTool` | 团队创建 | Feature-gated |
| `TeamDeleteTool` | 团队删除 | Feature-gated |

### 17.4 KAIROS 自主模式

从源码中提取的 KAIROS 系统信息：

```
KAIROS (全自主助手模式)：
├── 始终在后台运行的 AI 代理
├── Push 通知支持
├── GitHub PR webhook 集成
├── <tick> 心跳机制 — 定期自主执行
├── Push-to-talk 语音模式
├── 主动工具 (SleepTool, CronTools)
└── 独立的 assistant/ 模块系统
```

### 17.5 Coordinator 多智能体模式

```
Coordinator Mode：
├── 维护每个代理的独立对话上下文
├── 根据最佳适配分派工作
├── 聚合多代理结果
├── workerAgent.js — 工作代理
└── coordinatorMode.js — 协调器核心
```

---

## 第十八部分：架构设计模式总结

### 18.1 异步生成器流式处理

```typescript
// 核心模式：使用 async generator 实现流式处理
async function* query(params): AsyncGenerator<StreamEvent | Message> {
  // yield 个别事件
  // 允许消费者实时响应部分结果
  // 通过 return() 实现资源清理
}

// 消费模式
for await (const event of query(params)) {
  if (event.type === 'tool_call') { /* 更新 UI */ }
  if (event.type === 'message') { /* 显示文本 */ }
}
```

### 18.2 Feature Gate 编译时消除

```typescript
// 编译时条件编译 + 死代码消除
if (feature('FEATURE_NAME')) {
  // Bun: flag=true 时包含，flag=false 时 DCE
  // esbuild: feature() → false, 整个分支被消除
  require('./feature-module.js')
}
```

### 18.3 懒加载打破循环依赖

```typescript
// 避免模块循环引用
const getTeammateUtils = () => 
  require('./utils/teammate.js') as typeof import('./utils/teammate.js')

// 只在实际调用时加载
const utils = getTeammateUtils()
```

### 18.4 Memoization 缓存昂贵操作

```typescript
export const getSystemContext = memoize(async () => {
  // 整个会话期间缓存
  // 在 settings 变更时清除
})
```

### 18.5 辨别联合类型保证类型安全

```typescript
type Message = 
  | { type: 'user'; content: string }
  | { type: 'assistant'; message: { content: ContentBlockParam[] } }
  | { type: 'attachment'; metadata: { name: string } }

// 编译时穷尽检查
if (message.type === 'user') { ... }
else if (message.type === 'assistant') { ... }
// TypeScript 会确保处理所有类型
```

### 18.6 并行预取启动优化

```typescript
// main.tsx 中的并行预取模式
startMdmRawRead()        // 并行 1：MDM 子进程
startKeychainPrefetch()  // 并行 2：密钥链读取

// 与模块加载并行执行 (~135ms)
// 结果在需要时按需等待
```

### 18.7 状态管理双轨制

```
React 状态 (AppStateStore):
├── 交互式 UI 使用
├── 生命周期由 React 管理
└── 适用于 REPL 模式

Bootstrap 状态 (bootstrap/state.ts):
├── 全局可变状态
├── 在 React 渲染前可用
└── 适用于 SDK/无头模式
```

---

## 第十九部分：与标准 Claude API 的本质差异

| 特性 | 标准 Claude API | Claude Code |
|------|----------------|------------|
| **执行方式** | API 调用 | CLI + 交互式 TUI |
| **文件访问** | 无 | 完整文件系统 |
| **代码执行** | 无 | Bash/PowerShell |
| **会话状态** | 无状态 | 持久化会话存储 |
| **工具权限** | API Key 固定 | 交互式提示 + 多模式 |
| **上下文管理** | 固定窗口 | 自动压缩 + 缓存 |
| **智能体系统** | 未内置 | 完整子代理/协调器 |
| **MCP 支持** | 未内置 | 完整 MCP 客户端 + 注册表 |
| **插件/技能** | 不支持 | 完整插件系统 |
| **Git 集成** | 不支持 | 深度 Git 状态 + Hook |
| **CLAUDE.md** | 不支持 | 多层级支持 |
| **安全模型** | 无 | 规则 + 分类器 + 沙箱 |
| **成本控制** | 按调用 | Token 预算 + 自动压缩 |
| **多模型** | 手动选择 | 动态切换 + 降级 |

---

## 第二十部分：未来路线图与展望

### 20.1 已知的开发方向

从源码中的 Feature Gate 和代码注释中推断的未来方向：

```
1. KAIROS 全自主模式
   ├── 始终在后台运行的 AI 助手
   ├── 主动执行任务
   └── 语音交互支持

2. Coordinator 多智能体协调
   ├── 多个专业代理协同
   ├── 任务自动分配
   └── 结果聚合与冲突解决

3. 更强的 MCP 生态
   ├── 官方 MCP 注册表
   ├── OAuth 完整支持
   └── 更多传输协议

4. 上下文管理进化
   ├── Snip 压缩 (更智能的上下文剪切)
   ├── 上下文坍缩 (Context Collapse)
   └── 更大的上下文窗口支持

5. 企业级功能
   ├── MDM 策略管理
   ├── 远程设置推送
   ├── 合规审计日志
   └── SSO 集成
```

### 20.2 模型迭代节奏

源码中的模型启动注释 `@[MODEL LAUNCH]` 显示：
- `FRONTIER_MODEL_NAME = 'Claude Opus 4.6'` — 当前前沿模型
- 每次模型更新需修改多处 Prompt 中的模型名称
- 支持 Bedrock 和 Vertex AI 多平台部署

---

## 附录

### 附录 A：完整工具参数速查

#### FileRead

```json
{
  "file_path": "绝对路径 (必填)",
  "offset": "起始行号 (可选)",
  "limit": "读取行数 (可选, 默认 2000)",
  "pages": "PDF 页码范围 (可选, 如 '1-5')"
}
```

#### FileEdit

```json
{
  "file_path": "绝对路径 (必填)",
  "old_string": "要替换的文本 (必填, 必须唯一)",
  "new_string": "替换后的文本 (必填)",
  "replace_all": "替换所有匹配 (可选, 默认 false)"
}
```

#### Bash

```json
{
  "command": "Shell 命令 (必填)",
  "timeout": "超时毫秒数 (可选, 默认 120000, 最大 600000)",
  "run_in_background": "后台运行 (可选, 默认 false)"
}
```

#### Agent

```json
{
  "prompt": "任务描述 (必填)",
  "description": "3-5 词简短描述 (必填)",
  "subagent_type": "代理类型 (可选)",
  "model": "模型覆盖 (可选, sonnet|opus|haiku)",
  "isolation": "隔离模式 (可选, 'worktree')",
  "run_in_background": "后台运行 (可选)"
}
```

#### Grep

```json
{
  "pattern": "正则表达式 (必填)",
  "path": "搜索目录 (可选, 默认 cwd)",
  "glob": "文件过滤 (可选, 如 '*.ts')",
  "type": "文件类型 (可选, 如 'js', 'py')",
  "output_mode": "输出模式 (可选, content|files_with_matches|count)",
  "context": "上下文行数 (可选)",
  "head_limit": "结果限制 (可选, 默认 250)",
  "multiline": "多行匹配 (可选, 默认 false)"
}
```

#### Glob

```json
{
  "pattern": "Glob 模式 (必填, 如 '**/*.tsx')",
  "path": "搜索目录 (可选, 默认 cwd)"
}
```

### 附录 B：settings.json 完整模板

```json
{
  "theme": "dark",
  "verbose": false,
  "autoCompactEnabled": true,
  
  "permissions": {
    "defaultMode": "default",
    "rules": {
      "allow": [
        "FileRead",
        "FileEdit", 
        "FileWrite",
        "Glob",
        "Grep",
        "Bash(prefix:npm)",
        "Bash(prefix:git status)",
        "Bash(prefix:git diff)",
        "Bash(prefix:git log)",
        "Bash(prefix:ls)",
        "Bash(prefix:pwd)"
      ],
      "deny": [
        "Bash(prefix:rm -rf /)",
        "Bash(prefix:sudo)",
        "Bash(prefix:git push --force)"
      ],
      "ask": []
    }
  },
  
  "env": {
    "NODE_ENV": "development"
  },
  
  "mcpServers": {},
  
  "hooks": [],
  
  "preferredNotifChannel": "system"
}
```

### 附录 C：CLAUDE.md 最佳实践模板

```markdown
# 项目：[项目名称]

## 概述
[一句话描述项目]

## 技术栈
- 语言: [TypeScript/Python/...]
- 框架: [Next.js/FastAPI/...]
- 数据库: [PostgreSQL/MongoDB/...]
- 包管理: [npm/pnpm/poetry/...]

## 目录结构
- src/ — 源代码
- tests/ — 测试
- docs/ — 文档
- scripts/ — 脚本

## 开发命令
- 启动: `npm run dev`
- 测试: `npm test`
- 构建: `npm run build`
- 检查: `npm run lint`

## 编码规范
- [列出关键规范]

## 注意事项
- [列出需要特别注意的事项]

## 引用文档
@./docs/architecture.md
@./docs/api.md
```

### 附录 D：关键源码文件索引

| 功能 | 文件路径 | 行数 |
|------|---------|------|
| CLI 入口 | `src/entrypoints/cli.tsx` | ~300 |
| 主协调器 | `src/main.tsx` | ~4,683 |
| 查询引擎 | `src/QueryEngine.ts` | ~200+ |
| **核心循环** | **`src/query.ts`** | **~1,729** |
| 工具定义 | `src/Tool.ts` | ~200+ |
| 工具注册 | `src/tools.ts` | ~300+ |
| 流式执行 | `src/services/tools/StreamingToolExecutor.ts` | ~200+ |
| 权限引擎 | `src/utils/permissions/permissions.ts` | ~250+ |
| YOLO 分类器 | `src/utils/permissions/yoloClassifier.ts` | 52KB |
| Hook 执行 | `src/utils/hooks/execAgentHook.ts` | ~200+ |
| Hook 类型 | `src/types/hooks.ts` | ~300+ |
| MCP 客户端 | `src/services/mcp/client.ts` | 119KB |
| MCP 类型 | `src/services/mcp/types.ts` | ~200+ |
| API 客户端 | `src/services/api/claude.ts` | 125KB |
| 重试逻辑 | `src/services/api/withRetry.ts` | 28KB |
| 上下文压缩 | `src/services/compact/compact.ts` | ~100+ |
| 自动压缩 | `src/services/compact/autoCompact.ts` | ~100+ |
| 配置加载 | `src/utils/config.ts` | ~250+ |
| CLAUDE.md | `src/utils/claudemd.ts` | ~200+ |
| Prompt 构造 | `src/constants/prompts.ts` | ~300+ |
| Prompt 缓存 | `src/constants/systemPromptSections.ts` | ~60 |
| 系统提示词 | `src/utils/systemPrompt.ts` | ~124 |
| 消息类型 | `src/types/message.ts` | ~100+ |
| 权限类型 | `src/types/permissions.ts` | ~250+ |
| 应用状态 | `src/state/AppStateStore.ts` | ~150+ |
| 命令注册 | `src/commands.ts` | ~200+ |
| Token 计数 | `src/utils/tokens.ts` | ~100+ |
| 上下文分析 | `src/utils/analyzeContext.ts` | 42KB |
| 构建脚本 | `scripts/build.mjs` | ~230 |
| Agent 工具 | `src/tools/AgentTool/AgentTool.tsx` | 233KB |
| Agent 加载 | `src/tools/AgentTool/loadAgentsDir.ts` | 26KB |

### 附录 E：常见问题与解答

**Q1: Claude Code 的上下文窗口有多大？**  
A: 取决于使用的模型。Claude Opus 4.6 支持 1M Token 上下文。系统会自动管理并在接近限制时触发压缩。

**Q2: 如何减少 API 费用？**  
A: 
1. 使用 CLAUDE.md 减少探索性 Token 消耗
2. 使用具体文件路径而不是搜索
3. 及时使用 /compact 压缩上下文
4. 对简单任务使用 Haiku 模型

**Q3: Hook 和 MCP 有什么区别？**  
A: Hook 是在 Claude Code 内部事件节点注入逻辑（如工具调用前后），MCP 是通过标准协议扩展可用的外部工具和数据源。

**Q4: 子代理可以嵌套多少层？**  
A: 源码中未见硬性限制，但每层子代理都会消耗 Token 预算。实践中建议不超过 2-3 层。

**Q5: CLAUDE.md 最多可以包含多少内容？**  
A: 源码中定义 `MAX_MEMORY_CHARACTER_COUNT = 40,000` 字符。超出部分会被截断。

---

## 致谢

本白皮书基于以下公开仓库的源码分析：
- [sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code)
- [ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap)

---

*本文档版权归作者所有。仅用于技术学习和研究目的。*
