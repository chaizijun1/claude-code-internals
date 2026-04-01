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
- [第二十一部分：实战工作流食谱](#第二十一部分实战工作流食谱)
- [第二十二部分：故障排除与常见陷阱](#第二十二部分故障排除与常见陷阱)
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

### 14.1 安装与首次运行

```bash
# 全局安装
npm install -g @anthropic-ai/claude-code

# 或使用 npx 直接运行（无需安装）
npx @anthropic-ai/claude-code

# 在项目目录中启动
cd /path/to/your/project
claude

# 带初始提示启动
claude "帮我分析这个项目的结构"

# 无头模式（脚本集成用）
claude -p "列出所有 TODO 注释" > todos.txt
```

### 14.2 理解交互界面

```
┌─────────────────────────────────────────────────────┐
│ Claude Code v2.1.88                                 │
│                                                     │
│ 💬 上一条 Claude 回复...                              │
│                                                     │
│ 🔧 Tool: FileRead (src/index.ts)                    │  ← 工具调用实时显示
│ ✅ Tool complete                                     │
│                                                     │
│ 💬 Claude 的最新回复...                               │
│                                                     │
│ > 你的输入光标在这里 _                                │  ← 输入区域
│                                                     │
│ [Tokens: 12,450 / 200,000]  [Cost: $0.03]          │  ← 状态栏
└─────────────────────────────────────────────────────┘

交互快捷键：
├── Enter        → 发送消息（多行：Shift+Enter）
├── Ctrl+C       → 中断当前操作
├── Ctrl+D       → 退出 Claude Code
├── Esc          → 取消当前输入
├── Up/Down      → 浏览历史输入
└── Tab          → 自动补全斜杠命令
```

### 14.3 创建你的第一个 CLAUDE.md

**这是提升效率的第一步也是最重要一步**。源码分析显示，CLAUDE.md 的内容会被注入到**每一轮**的系统提示中，直接影响模型行为。

```markdown
# 项目说明

## 技术栈
- Next.js 14 + TypeScript + App Router
- Prisma + PostgreSQL
- Tailwind CSS + shadcn/ui

## 开发命令
- `pnpm dev` — 启动开发服务器 (port 3000)
- `pnpm test` — 运行 Vitest 测试
- `pnpm lint` — ESLint 检查
- `pnpm build` — 生产构建

## 目录结构
- src/app/ — 页面路由 (App Router)
- src/components/ — React 组件 (使用 shadcn/ui)
- src/lib/ — 工具函数和共享逻辑
- src/server/ — 服务端逻辑 (tRPC routers)
- prisma/ — 数据库 Schema 和迁移

## 编码规范
- 使用函数组件 + Hooks，禁止 class 组件
- 状态管理用 Zustand，不用 Redux
- 所有 API 通过 tRPC，不要写 REST 路由
- CSS 用 Tailwind，不要写自定义 CSS 文件

## 重要约束
- 不要修改 src/lib/legacy/ — 旧代码，计划下个季度移除
- 测试必须通过才能提交
- 所有数据库变更必须创建迁移文件
- 环境变量在 .env.example 中有模板
```

**为什么有效**：
- 模型不需要花 Token 去探索技术栈 → 节省 30%+ Token
- 编码规范直接注入 → 减少返工
- 约束条件前置 → 避免误操作

### 14.4 权限规则配置

首次使用最重要的配置 — 合理的权限规则让交互更流畅：

```json
{
  "permissions": {
    "rules": {
      "allow": [
        "FileRead",
        "FileEdit",
        "FileWrite",
        "Glob",
        "Grep",
        "Bash(prefix:npm test)",
        "Bash(prefix:npm run)",
        "Bash(prefix:pnpm )",
        "Bash(prefix:git status)",
        "Bash(prefix:git diff)",
        "Bash(prefix:git log)",
        "Bash(prefix:git branch)",
        "Bash(prefix:ls)",
        "Bash(prefix:cat)",
        "Bash(prefix:pwd)",
        "Bash(prefix:echo)",
        "Bash(prefix:node -e)",
        "Bash(prefix:npx tsc --noEmit)",
        "Bash(prefix:npx prisma)"
      ],
      "deny": [
        "Bash(prefix:rm -rf /)",
        "Bash(prefix:sudo)",
        "Bash(prefix:git push --force)",
        "Bash(prefix:DROP )",
        "Bash(prefix:TRUNCATE )"
      ]
    }
  }
}
```

**配置原则**：
| 类别 | 策略 | 示例 |
|------|------|------|
| 只读操作 | 全部 allow | FileRead, Glob, Grep, git status |
| 项目常用命令 | 按前缀 allow | npm test, npm run, pnpm |
| 文件修改 | allow（有 Read 保护） | FileEdit, FileWrite |
| 危险操作 | deny | rm -rf /, sudo, force push |
| 其他 | 默认 ask | 未匹配的命令需确认 |

### 14.5 常用斜杠命令完整速查

| 命令 | 功能 | 关键用法 |
|------|------|---------|
| `/help` | 显示帮助 | 列出所有可用命令 |
| `/compact` | 压缩上下文 | 对话过长时使用，保留关键信息 |
| `/clear` | 清除历史 | 完全重新开始，清空所有上下文 |
| `/status` | 会话状态 | 查看 Token 使用量、模型信息 |
| `/cost` | 费用统计 | 本次会话累计 API 费用 |
| `/model` | 切换模型 | 如 `/model sonnet` 切换到 Sonnet |
| `/commit` | Git 提交 | 自动生成 commit message 并提交 |
| `/review` | 代码审查 | 审查当前分支的更改 |
| `/mcp` | MCP 管理 | 添加/删除/列出 MCP 服务器 |
| `/config` | 配置管理 | 查看/修改 settings.json |
| `/resume` | 恢复会话 | 恢复之前中断的对话 |
| `/tasks` | 任务列表 | 查看 Claude 的任务进度 |
| `/permissions` | 权限管理 | 查看/修改权限规则 |
| `/hooks` | Hook 管理 | 查看/测试配置的 Hook |
| `/doctor` | 诊断检查 | 检查环境配置问题 |
| `/fast` | 快速模式 | 切换 Fast 模式（同模型，更快输出）|

### 14.6 六大基础使用模式

#### 模式 1：代码库探索

```
你: 帮我了解这个项目的结构，重点关注 API 路由和数据模型

# 背后的工具调用链：
# Glob("**/*.ts") → 发现文件结构
# Grep("router|route|endpoint") → 定位 API 定义
# Read(关键文件) → 深入理解
# 生成结构化概览
```

**高效技巧**：给出具体方向比开放式探索快 3 倍。

#### 模式 2：Bug 定位与修复

```
你: 用户在提交表单时报错 "Cannot read property 'email' of undefined"，
    错误发生在 src/components/UserForm.tsx 第 42 行

# 背后的执行链：
# Read(UserForm.tsx) → 定位错误行
# Grep("email") → 追踪数据流
# Read(相关 API) → 理解完整链路
# Edit(修复代码) → 应用修复
# Bash("npm test") → 验证修复
```

**高效技巧**：提供错误信息和文件位置能将修复速度提升 5 倍。

#### 模式 3：功能开发

```
你: 添加用户头像上传功能。要求：
    - 支持 JPG/PNG，最大 5MB
    - 上传到 S3
    - 数据库存储 URL
    - 前端使用 shadcn Upload 组件
```

**高效技巧**：明确技术选型让模型直接编码，而不是先问你用什么。

#### 模式 4：代码重构

```
你: 将 src/utils/api.ts 中的 fetch 调用统一改为使用 axios，
    保持接口签名不变，添加统一的错误处理和重试逻辑
```

#### 模式 5：测试编写

```
你: 为 src/server/routers/user.ts 编写单元测试，
    使用 Vitest + MSW 模拟 HTTP 请求，
    覆盖正常路径和错误路径
```

#### 模式 6：代码审查

```
你: /review

# 或更具体的审查
你: 审查 src/auth/ 目录最近的修改，
    重点关注安全漏洞和性能问题
```

### 14.7 Shell 集成技巧

```bash
# 在 Claude Code 中运行交互式命令（! 前缀）
你: ! gcloud auth login
你: ! docker-compose up -d
你: ! npx prisma studio

# 管道输出给 Claude（无头模式）
git diff | claude -p "审查这些更改"
cat error.log | claude -p "分析这些错误日志"
curl api.example.com/status | claude -p "解释这个 API 响应"

# 将 Claude 输出写入文件
claude -p "生成 .github/workflows/ci.yml 的内容" > .github/workflows/ci.yml

# 多项目切换
你: 请将工作目录切换到 ../backend-service
```

### 14.8 理解权限提示

当 Claude Code 需要执行有风险的操作时，会出现权限提示：

```
Claude wants to run: rm -rf dist/ && npm run build

  [Y] Allow once       — 本次允许
  [A] Always allow      — 记住并始终允许此模式
  [N] Deny             — 拒绝本次
  [D] Always deny       — 记住并始终拒绝

推荐策略：
├── 构建/测试命令 → Always allow
├── 文件删除 → Allow once（确认后单次允许）
├── 不熟悉的命令 → Deny（先了解再决定）
└── 危险命令 → Always deny（如 force push）
```

---

## 第十五部分：高效使用技巧（进阶篇）

### 15.1 CLAUDE.md 大师课

#### 层级策略：全局 → 项目 → 规则文件

```
~/.claude/CLAUDE.md                    # 全局偏好（所有项目通用）
├── "我是高级全栈开发者，10 年经验"
├── "偏好函数式编程和不可变数据"
├── "使用中文回复，技术术语保留英文"
├── "代码注释用英文"
└── "不要解释基础概念，直接给方案"

项目根目录/CLAUDE.md                    # 项目配置
├── 技术栈、开发命令、目录结构
├── @./docs/architecture.md            # 引用架构文档
├── @./docs/api-spec.md               # 引用 API 规范
└── @./CONTRIBUTING.md                 # 引用贡献指南

.claude/rules/security.md              # 安全规则
├── "所有用户输入必须通过 zod 验证"
├── "SQL 查询只能通过 Prisma ORM"
├── "不使用 eval()、innerHTML、dangerouslySetInnerHTML"
└── "API 路由必须验证 JWT token"

.claude/rules/style.md                 # 风格规则
├── "React 组件使用 PascalCase"
├── "工具函数使用 camelCase"
├── "常量使用 UPPER_SNAKE_CASE"
├── "文件名使用 kebab-case"
└── "每个文件不超过 200 行"

.claude/rules/testing.md               # 测试规则
├── "新功能必须有单元测试"
├── "测试文件放在 __tests__/ 目录"
├── "使用 describe/it 结构"
├── "Mock 外部服务，不 mock 内部模块"
└── "测试覆盖率不低于 80%"
```

#### 不同项目类型的 CLAUDE.md 模板

**前端 React 项目**：
```markdown
# Frontend Project

## Stack: React 18 + TypeScript + Vite + Zustand + React Query

## Commands
- `pnpm dev` — Dev server on :5173
- `pnpm test` — Vitest
- `pnpm storybook` — Component showcase

## Conventions
- Prefer server components where possible
- Use React Query for all API calls, no raw fetch
- All forms use react-hook-form + zod
- Components: src/components/{feature}/{ComponentName}.tsx
- Hooks: src/hooks/use{HookName}.ts
- Never use `any` type — prefer `unknown` + type guard

## Key APIs
- Auth: /api/auth/* (NextAuth)
- Data: /api/trpc/* (tRPC)
```

**后端 Python 项目**：
```markdown
# Backend Service

## Stack: Python 3.12 + FastAPI + SQLAlchemy 2.0 + Alembic + pytest

## Commands
- `uv run uvicorn app.main:app --reload` — Dev server
- `uv run pytest` — Tests
- `uv run alembic upgrade head` — Run migrations
- `uv run mypy .` — Type check

## Conventions
- Use async/await everywhere
- Type hints on all functions
- Pydantic models for request/response
- One router file per resource in app/routers/
- Business logic in app/services/, not in routers
- Database models in app/models/

## Important
- Never use raw SQL strings — always use SQLAlchemy
- All endpoints need OpenAPI docs (docstring + response_model)
- Environment variables via app/config.py (pydantic-settings)
```

**基础设施 / DevOps 项目**：
```markdown
# Infrastructure

## Stack: Terraform + AWS + Docker + GitHub Actions

## Commands
- `terraform plan` — Preview changes
- `terraform apply` — Apply changes (DANGEROUS)
- `docker compose up -d` — Local stack

## Rules
- NEVER run terraform apply without showing me the plan first
- NEVER modify production tfvars files
- All resources must have proper tags
- Use modules from modules/ directory
- State is in S3 — do not modify backend config
```

#### @include 高级用法

```markdown
# CLAUDE.md

## 核心文档（自动注入到每轮对话）
@./docs/architecture.md
@./docs/api-spec.md
@./CONVENTIONS.md

## 条件引用
@./docs/database-schema.sql
@./openapi.yaml

## 注意事项
# @include 的文件内容算入 40,000 字符上限
# 优先引用精简的文档，不要引用整个 README
# 如果文件不存在，会被静默忽略
```

### 15.2 Prompt 工程大全（15 个实战模式）

从 System Prompt 源码分析得出的高效 Prompt 模式：

#### 模式 1：精确定位

```
❌ "帮我改改这个文件"
❌ "登录功能有 bug"
✅ "修改 src/api/users.ts:42 的 getUser 函数，
    将数据库查询从 findFirst 改为 findUnique，
    因为 id 字段是唯一的"
```

#### 模式 2：上下文预加载

```
你: 先阅读以下文件，理解数据流后再回答：
    - src/models/user.ts
    - src/server/routers/user.ts  
    - src/components/UserProfile.tsx
    然后告诉我，如果我想添加 "用户地址" 字段，
    需要修改哪些文件，按什么顺序？
```

**原理**：源码中 Read 工具比 Bash(cat) 高效得多，且会进入文件缓存。

#### 模式 3：分步确认

```
你: 我需要将数据库从 MySQL 迁移到 PostgreSQL。
    请按以下步骤执行，每一步完成后等我确认：
    1. 分析所有 MySQL 特有语法的使用
    2. 列出需要修改的文件清单
    3. 修改 ORM 配置
    4. 修改数据类型映射
    5. 修改原生 SQL 查询
    6. 更新测试
```

#### 模式 4：角色设定

```
你: 你现在是一个安全审计专家。请对 src/auth/ 目录
    进行全面安全审查，按 OWASP Top 10 逐项检查，
    并给出严重程度评级（Critical/High/Medium/Low）
```

#### 模式 5：对比分析

```
你: 比较 src/services/old-payment.ts 和 
    src/services/new-payment.ts 的实现差异，
    新版本是否完全覆盖了旧版本的功能？
    有没有遗漏的边界情况？
```

#### 模式 6：逆向工程

```
你: 阅读 src/lib/encryption.ts，解释加密流程，
    画出数据流图，标注每一步的输入输出类型
```

#### 模式 7：约束驱动

```
你: 实现一个缓存模块，约束条件：
    - 零外部依赖
    - 支持 TTL
    - 支持 LRU 淘汰
    - 线程安全
    - 单文件，不超过 100 行
```

#### 模式 8：测试驱动（TDD）

```
你: 用 TDD 方式实现邮件验证函数：
    1. 先写测试用例（正常、边界、异常）
    2. 运行测试确认全部失败
    3. 实现最小代码让测试通过
    4. 重构
```

#### 模式 9：批量操作

```
你: 在整个项目中：
    1. 将所有 console.log 替换为 logger.debug
    2. 将所有 console.error 替换为 logger.error
    3. 在每个文件顶部添加 import { logger } from '@/lib/logger'
    4. 确保没有遗漏，运行测试验证
```

#### 模式 10：文档生成

```
你: 为 src/server/routers/ 下的所有 API 端点生成 
    Markdown 文档，格式如下：
    ## [端点名称]
    - 方法: GET/POST/...
    - 路径: /api/...
    - 参数: {...}
    - 响应: {...}
    - 示例: curl ...
```

#### 模式 11：渐进式重构

```
你: 将 src/utils/helpers.ts 重构为更小的模块，
    但要保证向后兼容：
    - 重构后的代码放在 src/utils/ 子目录
    - 原文件改为 re-export
    - 运行测试确保不破坏任何东西
```

#### 模式 12：错误处理增强

```
你: 审查 src/api/ 下所有路由，找出缺少错误处理的地方：
    - 没有 try-catch 的异步操作
    - 没有验证输入的端点
    - 没有正确返回错误码的地方
    修复所有问题，统一使用项目的 AppError 类
```

#### 模式 13：性能分析

```
你: 分析 src/pages/dashboard.tsx 的性能问题：
    - 找出不必要的重渲染
    - 检查是否有 N+1 查询
    - 检查是否有内存泄漏风险
    - 建议 React.memo / useMemo / useCallback 的使用
    给出具体代码修改，不要只说建议
```

#### 模式 14：迁移脚本

```
你: 编写一个数据迁移脚本 scripts/migrate-user-roles.ts：
    - 从 user.role (字符串) 迁移到 user_roles 表（多对多）
    - 支持回滚
    - 有进度输出
    - 处理中断恢复
    - dry-run 模式
```

#### 模式 15：并行 Agent 委托

```
你: 并行执行以下独立任务：
    1. [Explore Agent] 分析项目中所有第三方 API 集成点
    2. [General Agent] 为 UserService 写单元测试
    3. [Explore Agent] 列出所有环境变量和它们的用途

# 源码显示 Claude Code 会自动：
# - 为每个任务创建独立子代理
# - 根据任务类型选择最佳 Agent（Explore 用于只读搜索）
# - 并行执行，结果汇总
```

### 15.3 Agent 并行工作详解

#### 理解 Agent 类型选择

```
Agent 类型及适用场景（源码分析）：

general-purpose     → 需要读写操作的复杂任务
                      拥有所有工具，最通用

Explore             → 代码库搜索和分析
                      只有只读工具，速度快
                      适合 "查找"、"分析"、"了解"

Plan                → 设计实现方案
                      只读 + 不能编辑/写入
                      适合 "如何实现"、"设计方案"

claude-code-guide   → Claude Code 自身使用帮助
                      Glob, Grep, Read, WebFetch, WebSearch
```

#### 高效并行模式

```
你: 我需要发布一个新版本，请并行完成：
    1. 运行所有测试并报告结果
    2. 检查是否有未提交的文件
    3. 审查自上次发布以来的所有 commit
    4. 检查 package.json 版本号是否已更新

# Claude Code 会并行启动 4 个 Agent
# 所有结果在各自完成后汇总
```

#### Worktree 隔离实验

```
你: 请在隔离环境中尝试升级 React 到 v19，
    测试所有组件是否兼容，不要影响当前代码

# Claude Code 会：
# 1. git worktree add → 创建独立仓库副本
# 2. 在 worktree 中执行所有修改和测试
# 3. 成功 → 返回 worktree 路径和分支名供你 merge
# 4. 失败 → 自动清理，报告哪些组件不兼容
```

### 15.4 MCP 服务器实战配置

#### 开发者常用 MCP 服务器组合

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "postgresql://localhost:5432/mydb" }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/docs"]
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": { "BRAVE_API_KEY": "${BRAVE_API_KEY}" }
    },
    "puppeteer": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-puppeteer"]
    }
  }
}
```

#### MCP 服务器使用场景

| MCP 服务器 | 解锁能力 | 使用示例 |
|-----------|---------|---------|
| **GitHub** | PR/Issue 管理 | "创建 PR"、"查看 Issue #42" |
| **PostgreSQL** | 直接查询数据库 | "查看最近注册的用户"、"分析表结构" |
| **Filesystem** | 访问项目外文件 | "读取 ~/Documents/spec.pdf" |
| **Memory** | 跨会话持久记忆 | "记住这个架构决策" |
| **Brave Search** | 搜索最新信息 | "搜索 React 19 的 breaking changes" |
| **Puppeteer** | 浏览器自动化 | "打开页面截图"、"测试表单提交" |
| **Slack** | 发送通知 | "将测试结果发到 #dev 频道" |
| **Linear/Jira** | 任务管理 | "更新 Linear ticket 状态" |

### 15.5 Hook 自动化工作流库

#### Hook 1：自动格式化（保存即格式化）

```json
{
  "event": "PostToolUse",
  "matcher": "FileEdit|FileWrite",
  "handler": {
    "type": "script",
    "command": "npx prettier --write \"$TOOL_INPUT_FILE_PATH\" 2>/dev/null; echo '{\"continue\":true}'"
  }
}
```

#### Hook 2：自动 Lint 检查

```json
{
  "event": "PostToolUse",
  "matcher": "FileEdit",
  "handler": {
    "type": "script",
    "command": "npx eslint \"$TOOL_INPUT_FILE_PATH\" --fix 2>&1 | tail -5; echo '{\"continue\":true}'"
  }
}
```

#### Hook 3：敏感文件保护

```json
{
  "event": "PreToolUse",
  "matcher": "FileEdit|FileWrite",
  "handler": {
    "type": "javascript",
    "code": "module.exports = async ({input}) => { const p = input.file_path || ''; if (p.includes('.env') || p.includes('credentials') || p.includes('secret')) return {hookSpecificOutput: {permissionDecision: 'deny'}, reason: 'Protected file'}; return {continue: true}; }"
  }
}
```

#### Hook 4：Commit Message 规范检查

```json
{
  "event": "PreToolUse",
  "matcher": "Bash",
  "handler": {
    "type": "javascript",
    "code": "module.exports = async ({input}) => { const cmd = input.command || ''; if (cmd.startsWith('git commit') && !cmd.match(/\\b(feat|fix|docs|style|refactor|test|chore)\\b/)) return {hookSpecificOutput: {permissionDecision: 'deny'}, reason: 'Commit message must follow conventional format'}; return {continue: true}; }"
  }
}
```

#### Hook 5：TypeScript 类型检查门禁

```json
{
  "event": "PostToolUse",
  "matcher": "FileEdit",
  "handler": {
    "type": "script",
    "command": "if echo \"$TOOL_INPUT_FILE_PATH\" | grep -qE '\\.(ts|tsx)$'; then npx tsc --noEmit 2>&1 | tail -10; fi; echo '{\"continue\":true}'"
  }
}
```

#### Hook 6：自动运行相关测试

```json
{
  "event": "PostToolUse",
  "matcher": "FileEdit",
  "handler": {
    "type": "script",
    "command": "TEST_FILE=$(echo \"$TOOL_INPUT_FILE_PATH\" | sed 's/\\.ts$/.test.ts/' | sed 's/\\.tsx$/.test.tsx/'); if [ -f \"$TEST_FILE\" ]; then npx vitest run \"$TEST_FILE\" 2>&1 | tail -10; fi; echo '{\"continue\":true}'"
  }
}
```

### 15.6 会话管理进阶

#### 会话恢复与延续

```bash
# 列出最近会话
claude --resume

# 恢复指定会话
claude --resume [session-id]

# 技巧：给会话起名
你: 请把当前会话命名为 "用户系统重构"
# 之后可以按名称搜索恢复
```

#### 上下文管理策略

```
长任务的上下文管理时间线：

阶段 1 (0-30% context)：自由探索和编码
        ↓
阶段 2 (30-60%)：开始有策略地使用上下文
        ├── 用 Grep 代替 Read 全文件
        ├── 给出精确行号范围
        └── 用简洁的指令
        ↓
阶段 3 (60-80%)：主动管理
        ├── /compact 压缩非关键历史
        ├── 将大任务拆分到新会话
        └── 使用 Agent 委托独立子任务
        ↓
阶段 4 (80%+)：Auto-Compact 触发
        └── 系统自动压缩，可能丢失早期细节
        
最佳实践：在阶段 2-3 主动 /compact
```

### 15.7 Git 工作流集成

#### 智能 Commit

```
你: /commit

# Claude Code 会：
# 1. git status 查看变更
# 2. git diff 分析所有修改
# 3. git log 参考历史风格
# 4. 生成规范的 commit message
# 5. 执行 git commit

# 技巧：直接给提交指令更高效
你: 将刚才的所有修改提交，commit message 要用 conventional commits 格式
```

#### Code Review 工作流

```
你: /review

# 或更精确的审查
你: 审查当前分支相对于 main 的所有更改，关注：
    1. 安全漏洞
    2. 性能问题
    3. 代码风格一致性
    4. 测试覆盖率
    给出具体的改进建议和代码示例

# 审查 PR
你: 审查 GitHub PR #123 的代码
```

#### 分支管理

```
你: 从 main 创建新分支 feature/user-avatar，
    然后实现用户头像功能

你: 当前分支的修改和 main 有冲突吗？如果有，帮我解决

你: 将当前分支的 3 个 commit squash 为一个，
    生成清晰的 commit message
```

### 15.8 多文件重构模式

```
你: 将项目从 CommonJS (require/module.exports) 
    迁移到 ESM (import/export)：
    1. 先分析有多少文件需要修改
    2. 列出修改清单让我确认
    3. 按依赖顺序（叶子文件优先）逐个修改
    4. 每修改一批文件后运行测试
    5. 更新 package.json 和 tsconfig.json

# Claude Code 会利用 Glob 找到所有文件
# 用 Grep 分析 require/module.exports 用法
# 按拓扑排序修改
# 用 Bash 运行测试验证每一步
```

---

## 第十六部分：高效使用技巧（精通篇）

### 16.1 深度理解权限系统

从源码中发现的**权限决策链**：

```
工具调用 → Allow Rules → Deny Rules → Ask Rules 
       → Permission Mode → YOLO Classifier → User Prompt

决策原因类型（PermissionDecisionReason）：
├── classifier   — Auto 模式分类器决策
├── hook         — Hook 返回的决策
├── rule         — 权限规则匹配
├── mode         — 权限模式决定
├── safetyCheck  — 安全检查
├── sandboxOverride — 沙箱覆盖
├── workingDir   — 工作目录检查
└── asyncAgent   — 异步代理
```

**精确配置策略**：

```json
{
  "permissions": {
    "rules": {
      "allow": [
        "Bash(prefix:npm )",
        "Bash(prefix:pnpm )",
        "Bash(prefix:yarn )",
        "Bash(prefix:node )",
        "Bash(prefix:npx )",
        "Bash(prefix:git status)",
        "Bash(prefix:git diff)",
        "Bash(prefix:git log)",
        "Bash(prefix:git branch)",
        "Bash(prefix:git stash)",
        "Bash(prefix:docker compose)",
        "Bash(prefix:docker ps)",
        "Bash(prefix:curl )",
        "Bash(prefix:cat )",
        "Bash(prefix:ls )",
        "Bash(prefix:find )",
        "Bash(prefix:wc )",
        "Bash(prefix:head )",
        "Bash(prefix:tail )",
        "Bash(prefix:grep )",
        "Bash(prefix:echo )",
        "Bash(prefix:pwd)",
        "Bash(prefix:which )",
        "Bash(prefix:env | grep)",
        "FileRead",
        "FileEdit",
        "FileWrite",
        "Glob",
        "Grep",
        "Agent",
        "NotebookEdit",
        "WebFetch",
        "WebSearch"
      ],
      "deny": [
        "Bash(prefix:rm -rf /)",
        "Bash(prefix:sudo )",
        "Bash(prefix:chmod 777)",
        "Bash(prefix:git push --force)",
        "Bash(prefix:git reset --hard)",
        "Bash(prefix:DROP )",
        "Bash(prefix:TRUNCATE )",
        "Bash(prefix:shutdown)",
        "Bash(prefix:reboot)",
        "Bash(prefix:mkfs)"
      ]
    }
  }
}
```

### 16.2 Token 预算精细管理

#### 成本估算公式（源码推导）

```
单次对话成本 ≈ 系统提示 Token × 轮次数 × 输入价格
            + 输出 Token 总量 × 输出价格
            + 工具调用结果 Token × 输入价格

优化杠杆（按效果排序）：
1. 减少系统提示大小 → CLAUDE.md 精简
2. 减少轮次 → 精确指令 + 一次说清
3. 减少工具调用 → 给出文件路径避免搜索
4. 模型选择 → 简单任务用 Haiku
5. 及时压缩 → /compact 避免冗余
```

#### 不同任务的成本对比

| 任务类型 | 预估轮次 | 预估成本 | 优化技巧 |
|---------|---------|---------|---------|
| 简单 Bug 修复 | 3-5 | $0.02-0.05 | 给出文件路径和行号 |
| 功能开发 | 10-20 | $0.10-0.30 | 分步确认，用 CLAUDE.md 预设 |
| 代码审查 | 5-10 | $0.05-0.15 | 用 /review 内置命令 |
| 全项目重构 | 30-60 | $0.50-2.00 | 拆分子任务，Agent 并行 |
| 文档生成 | 5-10 | $0.05-0.10 | 模板化输出 |

#### 实用省钱技巧

```
技巧 1：模型切换
├── 复杂架构设计 → Opus（最强推理）
├── 日常编码 → Sonnet（性价比最高）
├── 简单修改/格式化 → Haiku（最便宜）
└── 切换命令：/model sonnet

技巧 2：减少无效 Token
├── 明确说 "不要解释，直接改代码"
├── 给文件路径而不是让它搜索
├── 一次性给全上下文，而不是逐步追问
└── 用 Agent 并行而不是串行

技巧 3：Smart Compact
├── 完成一个功能点后立即 /compact
├── 切换话题时 /compact
├── 感觉响应变慢时 /compact
└── 不要等到 Auto-Compact（那时已浪费很多 Token）
```

### 16.3 复杂 Hook 管道构建

#### AI 驱动的代码审查门禁

```json
{
  "event": "PreToolUse",
  "matcher": "FileEdit",
  "handler": {
    "type": "agent",
    "prompt": "审查即将进行的文件编辑。检查：\n1. 是否引入安全漏洞（XSS、SQL注入、路径遍历）\n2. 是否遵循 DRY 原则\n3. 是否有明显的逻辑错误\n4. 是否有未处理的边界情况\n\n如果发现 Critical/High 问题，block 并说明原因。\n否则 approve。",
    "timeout": 30,
    "model": "claude-haiku-4-5-20251001"
  }
}
```

#### 全自动 CI 管道（本地版）

```json
{
  "hooks": [
    {
      "event": "PostToolUse",
      "matcher": "FileEdit|FileWrite",
      "handler": {
        "type": "script",
        "command": "bash -c 'F=\"$TOOL_INPUT_FILE_PATH\"; npx prettier --write \"$F\" 2>/dev/null; if echo \"$F\" | grep -qE \"\\.(ts|tsx)$\"; then npx eslint --fix \"$F\" 2>&1 | tail -3; fi; echo \"{\\\"continue\\\":true}\"'"
      }
    },
    {
      "event": "PostToolUse",
      "matcher": "FileEdit",
      "handler": {
        "type": "script",
        "command": "bash -c 'F=\"$TOOL_INPUT_FILE_PATH\"; TF=$(echo \"$F\" | sed \"s/\\.ts$/.test.ts/;s/\\.tsx$/.test.tsx/\"); if [ -f \"$TF\" ]; then npx vitest run \"$TF\" --reporter=verbose 2>&1 | tail -15; fi; echo \"{\\\"continue\\\":true}\"'"
      }
    },
    {
      "event": "PostToolUse",
      "matcher": "Bash",
      "handler": {
        "type": "javascript",
        "code": "module.exports = async ({input}) => { const c = input.command||''; if (c.includes('git commit')) { const r = require('child_process').execSync('npx tsc --noEmit 2>&1 || true').toString(); if (r.includes('error TS')) return {continue:true, systemMessage:'TypeScript errors found, fix before committing: '+r.slice(0,500)}; } return {continue:true}; }"
      }
    }
  ]
}
```

### 16.4 SDK / 无头模式集成

#### 在脚本中调用 Claude Code

```typescript
// script.ts — 批量代码分析
import Anthropic from "@anthropic-ai/sdk";

// Claude Code SDK 模式
const client = new Anthropic();

// 也可以直接用 CLI 的 -p 模式
import { execSync } from "child_process";

// 分析多个文件的安全性
const files = ["src/auth.ts", "src/api.ts", "src/db.ts"];
for (const file of files) {
  const result = execSync(
    `claude -p "分析 ${file} 的安全隐患，JSON 格式输出" --output-format json`,
    { encoding: "utf-8" }
  );
  console.log(`${file}: ${result}`);
}
```

#### CI/CD 集成

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      
      - name: Run Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          DIFF=$(git diff origin/main...HEAD)
          claude -p "审查以下 diff，找出 bug 和安全问题：\n$DIFF" \
            --output-format json > review.json
      
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            const review = require('./review.json');
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: review.result
            });
```

### 16.5 自定义 Agent 创建

```json
// ~/.claude/agents/security-auditor.json
{
  "agentType": "security-auditor",
  "name": "Security Auditor",
  "description": "专门进行代码安全审计的代理",
  "whenToUse": "当需要安全审查、漏洞检测或合规检查时",
  "tools": ["FileRead", "Glob", "Grep", "Bash", "WebSearch"],
  "disallowedTools": ["FileEdit", "FileWrite"],
  "systemPrompt": "你是一个资深安全工程师，专注于：\n1. OWASP Top 10 漏洞检测\n2. 依赖安全审计\n3. 认证/授权逻辑审查\n4. 敏感数据泄露防护\n5. 输入验证完整性\n\n输出格式：\n- 严重程度: Critical/High/Medium/Low/Info\n- 位置: 文件路径:行号\n- 描述: 问题说明\n- 修复建议: 具体代码示例"
}

// ~/.claude/agents/db-expert.json
{
  "agentType": "db-expert",
  "name": "Database Expert",
  "description": "数据库设计和查询优化专家",
  "whenToUse": "当需要数据库设计、查询优化或迁移时",
  "tools": ["FileRead", "Glob", "Grep", "Bash"],
  "systemPrompt": "你是 PostgreSQL 数据库专家，擅长：\n1. Schema 设计和范式化\n2. 查询性能优化（EXPLAIN ANALYZE）\n3. 索引策略\n4. 数据迁移脚本\n5. 连接池和并发处理"
}

// ~/.claude/agents/doc-writer.json
{
  "agentType": "doc-writer",
  "name": "Documentation Writer",
  "description": "技术文档和 API 文档撰写专家",
  "whenToUse": "当需要编写文档、README、API doc 时",
  "tools": ["FileRead", "FileWrite", "Glob", "Grep"],
  "systemPrompt": "你是技术文档专家。输出的文档要求：\n1. 结构清晰，使用标题层级\n2. 包含代码示例\n3. API 文档包含请求/响应示例\n4. 面向目标读者（开发者）\n5. Markdown 格式"
}
```

**使用方式**：
```
你: 用 security-auditor 代理审查整个 src/auth/ 目录

你: 用 db-expert 代理优化这个慢查询：
    SELECT * FROM orders JOIN users ON ...
```

### 16.6 性能优化清单

| # | 技巧 | 原理（源码依据） | 预期效果 |
|---|------|-----------------|---------|
| 1 | 写好 CLAUDE.md | 注入系统提示，减少探索 | Token -30% |
| 2 | 给出文件路径 | 跳过 Glob 搜索 | 响应快 2x |
| 3 | 给出行号范围 | Read(offset, limit) 减少读取 | Token -50%/文件 |
| 4 | 一次给全信息 | 减少追问轮次 | 轮次 -60% |
| 5 | 用 Agent 并行 | 子代理独立上下文 | 速度快 3x |
| 6 | 及时 /compact | 减少每轮 Token 消耗 | 长对话续航 2x |
| 7 | 配置 allow 规则 | 减少权限交互等待 | 流畅度 +50% |
| 8 | 用 Grep 替代 Read | 只获取相关行 | Token -80%/搜索 |
| 9 | 简单任务用 Haiku | 模型降级 | 费用 -90% |
| 10 | 批量操作一次下 | 减少来回 | 效率 +200% |

### 16.7 调试与诊断大全

```bash
# ──────── 会话信息 ────────
/status          # Token 使用量、模型、会话 ID
/cost            # 本次会话累计费用
/config          # 当前配置详情

# ──────── 环境诊断 ────────
/doctor          # 检查环境、依赖、配置问题

# ──────── 上下文诊断 ────────
/compact         # 查看上下文使用情况 + 压缩
                 # 源码中 analyzeContext.ts 计算：
                 # - 消息 Token 数
                 # - 系统提示 Token 数
                 # - 剩余预算
                 # - 是否需要压缩

# ──────── 详细模式 ────────
# settings.json 中设置
{ "verbose": true }
# 会显示：
# - 每次 API 调用的 Token 消耗
# - 工具调用详细参数
# - 权限决策原因
# - Hook 执行结果

# ──────── 常见问题快速排查 ────────

问题：响应越来越慢
原因：上下文接近上限
方案：/compact 或 /clear 后重新开始

问题：权限提示太频繁
原因：allow 规则不够
方案：把常用命令加入 allow 规则

问题：Claude 忘记了之前的讨论
原因：Auto-Compact 压缩了早期上下文
方案：关键信息写入 CLAUDE.md 或重新说明

问题：MCP 服务器连接失败
原因：配置错误或进程未启动
方案：/mcp 查看状态，检查命令路径

问题：文件编辑结果不对
原因：old_string 不唯一或有隐藏字符
方案：给更多上下文让 old_string 唯一

问题：子代理没有预期效果
原因：Prompt 太模糊
方案：给出具体文件路径、预期结果、约束条件
```

### 16.8 多项目协同管理

```bash
# 在同一会话中切换项目
你: 切换到 ../api-service 目录

# 使用 addDir 添加额外工作目录
你: /addDir ../shared-types
# 现在 Claude Code 可以同时访问两个项目

# 跨项目操作示例
你: 我修改了 shared-types/src/user.ts 中的 User 类型，
    请检查 api-service 和 web-app 中所有使用 User 类型的地方，
    看哪些需要同步更新

# 微服务架构中的使用
你: 分析 api-gateway、user-service、order-service 三个服务
    之间的 API 调用关系，画出依赖图
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

## 第二十一部分：实战工作流食谱

> 本章提供 10 个完整的实战场景，每个都包含具体 Prompt、预期行为和最佳实践。

### 食谱 1：全栈功能开发（用户通知系统）

```
你: 我需要实现一个用户通知系统，包括：
    - 数据库：notifications 表（id, user_id, type, title, body, read, created_at）
    - API：GET /notifications, POST /notifications/read/:id, GET /notifications/unread-count
    - 前端：通知铃铛图标 + 下拉列表 + 未读数量角标
    - 实时：WebSocket 推送新通知

    请按以下顺序实现：
    1. 数据库迁移
    2. 后端 API
    3. WebSocket 处理
    4. 前端组件
    5. 集成测试
    每步完成后运行测试确认

预期行为：
├── Claude 会先阅读现有代码理解架构
├── 创建 Prisma migration
├── 实现 tRPC router
├── 添加 WebSocket handler
├── 创建 React 组件
├── 编写测试
└── 运行全部测试确认通过
```

### 食谱 2：Bug 调查与修复（生产环境问题）

```
你: 生产环境报告以下错误，每天大约出现 50 次：
    
    Error: Connection pool exhausted
    at PostgresPool.acquire (src/db/pool.ts:45)
    at UserRepository.findById (src/repos/user.ts:23)
    at AuthMiddleware.verify (src/middleware/auth.ts:67)
    
    请：
    1. 分析可能的根因（不急着修复）
    2. 检查连接池配置和使用模式
    3. 搜索所有数据库连接的使用，找出泄漏点
    4. 提出修复方案让我确认
    5. 确认后实施修复
    6. 添加监控/告警代码防止复发

预期行为：
├── Read pool.ts:45 附近代码
├── Grep 搜索所有数据库连接使用
├── 分析连接释放模式（是否有未关闭连接）
├── 检查事务中的错误处理（是否 finally 中释放）
├── 提出方案（等待确认）
├── 修复代码
└── 添加连接池监控 middleware
```

### 食谱 3：遗留代码现代化

```
你: src/legacy/ 目录包含 2019 年写的 Express.js 代码，
    我需要逐步迁移到现有的 Next.js App Router 架构。
    
    规则：
    - 每次只迁移一个端点
    - 旧代码保持可用直到新代码测试通过
    - 保持向后兼容（相同的请求/响应格式）
    - 使用 Prisma 替代原始 SQL
    
    请先分析 src/legacy/ 中有哪些端点，
    按复杂度排序，从最简单的开始迁移。

预期行为：
├── Glob 和 Grep 分析 legacy 目录
├── 列出所有端点及其复杂度
├── 从最简单的开始：
│   ├── 阅读旧实现
│   ├── 创建新的 App Router 版本
│   ├── 将原始 SQL 转为 Prisma
│   ├── 编写测试
│   ├── 运行测试确认
│   └── 报告完成，继续下一个
└── 每个端点独立完成和验证
```

### 食谱 4：API 设计与实现

```
你: 设计一个博客系统的 RESTful API，要求：
    - 资源：posts, comments, tags, users
    - 认证：JWT
    - 分页：cursor-based
    - 过滤：支持多字段
    - 排序：支持多字段
    
    请先设计 API 文档（OpenAPI 格式），
    等我确认后再实现。

预期行为：
├── 生成 OpenAPI YAML 规范
├── 等待用户确认
├── 实现数据库 Schema
├── 实现各个 endpoint
├── 添加中间件（auth, pagination, validation）
├── 编写集成测试
└── 生成 API 文档
```

### 食谱 5：数据库 Schema 变更

```
你: 需要将用户系统从单一角色改为多角色：
    - 当前：users 表有 role 字符串字段
    - 目标：roles 表 + user_roles 中间表
    - 要求：
      1. 创建迁移脚本
      2. 数据迁移（保留现有角色数据）
      3. 更新所有使用 user.role 的代码
      4. 更新 API 响应格式
      5. 支持回滚
    
    重要：先生成 dry-run 的迁移计划，
    列出所有会受影响的文件

预期行为：
├── 分析所有 user.role 的使用点
├── 生成影响分析报告
├── 创建迁移文件（含数据迁移）
├── 修改 ORM 模型
├── 更新所有引用代码
├── 更新 API 层
├── 编写测试
└── 验证回滚可行
```

### 食谱 6：性能优化

```
你: 仪表盘页面加载需要 8 秒，太慢了。
    页面组件是 src/pages/dashboard/index.tsx
    
    请完整分析并优化：
    1. 检查 API 调用（是否有串行请求应该并行）
    2. 检查数据库查询（N+1、缺少索引）
    3. 检查前端渲染（不必要的重渲染）
    4. 检查数据大小（是否取了不需要的字段）
    5. 建议缓存策略
    
    每个优化给出预期性能提升

预期行为：
├── Read 仪表盘组件代码
├── 追踪所有 API 调用 → 查找串行/瀑布请求
├── 追踪 SQL 查询 → 查找 N+1 和全表扫描
├── 分析 React 渲染 → 查找不必要的 state 变更
├── 提出优化方案（每个带预期收益）
├── 实施优化
├── 添加 performance mark 用于验证
└── 运行测试确认功能不受影响
```

### 食谱 7：安全审计

```
你: 对整个项目进行安全审计，按 OWASP Top 10 检查：
    A01 - 访问控制失效
    A02 - 密码学失败
    A03 - 注入
    A04 - 不安全的设计
    A05 - 安全配置错误
    A06 - 脆弱和过时的组件
    A07 - 身份认证和验证失败
    A08 - 软件和数据完整性失败
    A09 - 安全日志和监控失败
    A10 - 服务端请求伪造
    
    对每个发现给出：严重程度、文件位置、修复方案

推荐搭配：使用 security-auditor 自定义 Agent
你: 用 security-auditor Agent 并行审查以下目录：
    1. src/auth/
    2. src/api/
    3. src/middleware/
```

### 食谱 8：文档批量生成

```
你: 为项目生成以下文档：
    1. README.md — 项目介绍、安装、使用
    2. API.md — 所有 API 端点文档（从代码自动生成）
    3. ARCHITECTURE.md — 架构概览 + 数据流图
    4. CONTRIBUTING.md — 贡献指南
    5. CHANGELOG.md — 从 git log 生成变更日志
    
    每个文档先生成大纲让我确认

预期行为：
├── Glob + Grep 分析项目结构
├── 生成每个文档的大纲（等待确认）
├── 确认后逐个生成
├── API 文档从实际代码提取
├── 架构图用 Mermaid 格式
└── CHANGELOG 从 git log 生成
```

### 食谱 9：测试套件补全

```
你: 项目目前测试覆盖率只有 30%。
    请帮我将核心模块的覆盖率提升到 80%：
    
    优先级：
    1. src/server/routers/ — API 路由（最重要）
    2. src/services/ — 业务逻辑
    3. src/utils/ — 工具函数
    4. src/components/ — 关键交互组件
    
    要求：
    - 每个模块先分析现有测试，找出缺失
    - 测试包含：正常路径、边界情况、错误处理
    - Mock 外部依赖，不 mock 内部模块
    - 每完成一个模块运行一次测试

推荐：使用 Agent 并行编写不同模块的测试
```

### 食谱 10：依赖升级

```
你: 项目依赖需要大版本升级：
    - React 18 → 19
    - Next.js 14 → 15
    - TypeScript 5.3 → 5.5
    
    请：
    1. 先在隔离环境(worktree)中测试升级
    2. 列出所有 breaking changes 和受影响代码
    3. 逐个修复兼容性问题
    4. 运行完整测试套件
    5. 如果全部通过，告诉我如何 merge

预期行为：
├── 创建 git worktree（隔离环境）
├── 在 worktree 中升级依赖
├── npm install → 检查 peer dependency 冲突
├── npm run build → 收集编译错误
├── 逐个修复
├── npm test → 确认全部通过
└── 返回 worktree 路径供用户 merge
```

---

## 第二十二部分：故障排除与常见陷阱

### 22.1 Top 10 常见问题

| # | 问题 | 症状 | 根因 | 解决方案 |
|---|------|------|------|---------|
| 1 | **响应变慢** | 等待时间从秒级变成分钟级 | 上下文接近上限 | `/compact` 或开新会话 |
| 2 | **遗忘上下文** | Claude 忘记之前讨论的内容 | Auto-Compact 压缩了早期消息 | 关键信息写入 CLAUDE.md |
| 3 | **权限提示频繁** | 每个操作都要确认 | Allow 规则不够 | 扩展 settings.json rules |
| 4 | **文件编辑失败** | "old_string not found" | 字符串不唯一或有隐藏字符 | 提供更多上下文让 old_string 唯一 |
| 5 | **MCP 连接失败** | MCP 服务器无响应 | 命令路径错误或依赖缺失 | `/mcp` 检查状态，验证命令 |
| 6 | **子代理无效果** | Agent 返回空结果或不相关 | Prompt 太模糊 | 给出具体路径、目标、约束 |
| 7 | **Token 超限** | "Context window exceeded" | 单次对话内容太多 | 拆分任务，使用多个会话 |
| 8 | **代码风格不一致** | 生成的代码和项目风格不同 | CLAUDE.md 缺少风格指南 | 添加编码规范到 CLAUDE.md |
| 9 | **测试不通过** | 修改后测试失败 | Claude 不知道项目测试模式 | CLAUDE.md 中说明测试框架和模式 |
| 10 | **API 费用过高** | 单次会话花费数美元 | 冗余搜索、大文件读取 | 精确指令 + /compact + 模型切换 |

### 22.2 权限相关故障排除

```
问题：每次 git status 都要确认
┌──────────────────────────────────────┐
│ 诊断：检查 settings.json             │
│ $ cat ~/.claude/settings.json        │
│                                      │
│ 修复：添加 allow 规则                 │
│ "Bash(prefix:git status)"            │
│ "Bash(prefix:git diff)"              │
│ "Bash(prefix:git log)"               │
└──────────────────────────────────────┘

问题：无法编辑 .env 文件（被 Hook 阻止）
┌──────────────────────────────────────┐
│ 诊断：检查 Hook 配置                  │
│ /hooks                               │
│                                      │
│ 修复：调整 Hook 的 matcher            │
│ 或临时禁用该 Hook                     │
└──────────────────────────────────────┘

问题：sandbox 模式下命令被拒绝
┌──────────────────────────────────────┐
│ 诊断：SandboxManager 配置             │
│ 检查沙箱文件访问白名单                 │
│                                      │
│ 修复：将需要的路径加入白名单           │
│ 或在受信环境中禁用沙箱                 │
└──────────────────────────────────────┘
```

### 22.3 性能故障排除

```
问题：Claude Code 启动慢（>5 秒）
可能原因：
├── MCP 服务器启动超时 → 检查 MCP 配置，禁用不需要的服务器
├── 大量 CLAUDE.md 内容 → 精简 @include 文件
├── 网络问题 → 检查 API 连接
└── GrowthBook 功能标志拉取超时 → 通常会自动超时，无需处理

问题：工具执行超时
可能原因：
├── Bash 命令耗时过长 → 设置合理 timeout
├── 大型项目 Glob 过慢 → 使用更精确的 pattern
├── MCP 工具响应慢 → 检查 MCP 服务器状态
└── 网络请求超时 → WebFetch 默认 120s，可能需要调大

问题：内存使用过高
可能原因：
├── 长时间会话消息累积 → /compact 或 /clear
├── 大量子代理并行 → 减少并行数
└── 大文件读取 → 使用 offset/limit 参数
```

### 22.4 CLAUDE.md 陷阱

```
陷阱 1：内容过多
├── 问题：CLAUDE.md + @include 超过 40,000 字符
├── 症状：部分内容被静默截断
└── 修复：精简内容，只保留关键指令

陷阱 2：指令矛盾
├── 问题：全局 CLAUDE.md 和项目 CLAUDE.md 有矛盾指令
├── 症状：行为不确定
└── 修复：项目级覆盖全局级，确保一致性

陷阱 3：过度约束
├── 问题："不要修改任何文件" 类的指令
├── 症状：Claude 无法完成任务
└── 修复：约束应该是具体的，如 "不要修改 src/legacy/"

陷阱 4：@include 循环引用
├── 问题：A includes B, B includes A
├── 症状：源码有循环引用检测，会被打断
└── 修复：检查并消除循环引用

陷阱 5：包含大型文件
├── 问题：@include 了一个 10000 行的文件
├── 症状：挤占其他内容的空间，浪费 Token
└── 修复：只引用精简的文档摘要，不要引用完整文件
```

### 22.5 Hook 调试技巧

```bash
# 测试 Hook 是否正确配置
/hooks

# 调试 Hook 执行
# 在 Hook 脚本中添加日志
echo "{\"continue\":true}" 
# Hook 输出到 stderr 不影响 JSON 解析
echo "DEBUG: input was $TOOL_INPUT_FILE_PATH" >&2

# 常见 Hook 错误
1. JSON 输出格式错误 → Hook 返回必须是合法 JSON
2. 脚本无执行权限 → chmod +x hook-script.sh
3. 路径错误 → 使用绝对路径
4. 超时 → 默认 60s，复杂操作需设置更长 timeout
5. 环境变量缺失 → Hook 在独立进程中运行
```

### 22.6 最佳实践检查清单

```
□ 项目根目录有 CLAUDE.md
□ CLAUDE.md 包含技术栈、命令、目录结构、规范
□ settings.json 配置了合理的 allow/deny 规则
□ 常用命令（test, lint, build）在 allow 列表中
□ 危险命令（rm -rf /, sudo, force push）在 deny 列表中
□ 按项目类型设置了 .claude/rules/ 规则文件
□ MCP 服务器按需配置（不要配置不用的）
□ Hook 经过测试，不影响正常工作流
□ 长任务有 /compact 策略
□ 团队成员共享一致的 CLAUDE.md 和 .claude/ 配置
```

---

## 致谢

本白皮书基于以下公开仓库的源码分析：
- [sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code)
- [ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap)

---

*本文档版权归作者所有。仅用于技术学习和研究目的。*
