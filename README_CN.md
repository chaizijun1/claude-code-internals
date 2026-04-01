# Claude Code Internals

### 架构深度解析 + 高效使用指南：Anthropic AI 编程智能体

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![白皮书](https://img.shields.io/badge/白皮书-中文版-red.svg)](Claude_Code_Whitepaper.md)
[![Whitepaper](https://img.shields.io/badge/Whitepaper-English-green.svg)](docs/en/Claude_Code_Whitepaper_EN.md)
[![PDF 英文](https://img.shields.io/badge/PDF-英文版-orange.svg)](Claude_Code_Whitepaper_EN.pdf)
[![PDF 中文](https://img.shields.io/badge/PDF-中文版-orange.svg)](Claude_Code_Whitepaper.pdf)

[English](README.md) | **中文**

---

## 为什么需要这个仓库？

Claude Code 的能力远超大多数用户的认知。这份白皮书揭示了**它内部究竟是如何运作的**，更重要的是——**如何利用这些知识将你的生产力提升 10 倍**。

基于 **513,000 行 TypeScript 源代码**（v2.1.88）的深度分析，这既是：

- **架构参考手册** — 理解智能体循环、工具系统、权限模型和 108 个隐藏的 Feature Gate
- **实用高效使用指南** — 15 个 Prompt 模式、10 个工作流食谱、6 个 Hook 自动化、自定义 Agent、成本优化等

---

## 快速导航：我能学到什么？

### 想用好 Claude Code？从这里开始：

| 章节 | 你将学到 | 适合人群 |
|------|---------|---------|
| [第 14 章 — 入门技巧](Claude_Code_Whitepaper.md) | 首次配置、CLAUDE.md、权限规则、6 种使用模式、斜杠命令 | 新手用户 |
| [第 15 章 — 进阶技巧](Claude_Code_Whitepaper.md) | **15 个 Prompt 模式**、CLAUDE.md 大师课、6 个 Hook 食谱、MCP 配置、Agent 并行、Git 工作流 | 日常用户 |
| [第 16 章 — 精通技巧](Claude_Code_Whitepaper.md) | Token 预算精通、成本优化、CI/CD 集成、自定义 Agent、SDK 模式、多项目管理 | 高级用户 |
| [第 21 章 — 实战食谱](Claude_Code_Whitepaper.md) | **10 个完整实战场景**：全栈开发、Bug 调查、遗留迁移、安全审计、性能优化等 | 所有人 |
| [第 22 章 — 故障排除](Claude_Code_Whitepaper.md) | Top 10 常见问题、权限/性能调试、CLAUDE.md 陷阱、Hook 调试、最佳实践清单 | 遇到问题时 |

### 想理解内部实现？

| 章节 | 主题 |
|------|------|
| 第 1-2 章 | 概述 & 项目结构（1,884 文件、38 个服务） |
| 第 3 章 | **核心智能体循环** — 系统心脏 `query.ts`（1,729 行）|
| 第 4 章 | 工具系统 — 全部 45+ 工具、注册表、执行流水线 |
| 第 5 章 | 权限安全模型 — 5 种模式、YOLO 分类器、沙箱 |
| 第 6 章 | Hook 事件系统 — 15+ 事件类型、3 种处理器 |
| 第 7 章 | MCP 集成 — 6 种传输协议、OAuth、工具桥接 |
| 第 8 章 | 配置体系 — 四级层级、CLAUDE.md 解析 |
| 第 9 章 | 上下文管理 — Token 预算、自动压缩、Prompt 缓存 |
| 第 10 章 | Agent 系统 — 子代理、协调模式、Worktree 隔离 |
| 第 11 章 | System Prompt 构造 — 动态组装、分段缓存 |
| 第 12 章 | API 通信 — 重试逻辑、流式传输、错误处理 |
| 第 13 章 | 构建系统 — esbuild、Feature Gate DCE、108 个被消除模块 |
| 第 17 章 | **隐藏功能** — KAIROS、协调模式、9 个未发布工具 |
| 第 18-20 章 | 设计模式、Claude API 差异、未来路线图 |

---

## 亮点速览

### 使用技巧亮点

- **15 个 Prompt 工程模式** — 精确定位、上下文预加载、TDD、批量操作、角色设定、约束驱动等
- **10 个实战工作流食谱** — 全栈功能开发、Bug 调查、遗留代码迁移、安全审计、数据库变更、性能优化、文档生成、测试套件补全、依赖升级的完整步骤指南
- **6 个 Hook 自动化食谱** — 自动格式化、自动 Lint、敏感文件保护、Commit 规范检查、TypeScript 类型门禁、自动测试
- **3 个自定义 Agent 模板** — 安全审计员、数据库专家、文档撰写员
- **成本优化公式** — 从源码推导，含任务类型成本估算表
- **CI/CD 集成** — GitHub Actions YAML 自动化 Claude Code 代码审查
- **CLAUDE.md 大师课** — 提供 React、Python、DevOps 三种项目模板
- **故障排除指南** — Top 10 常见问题及根因分析和修复方案

### 架构亮点

- **Claude Code 不是聊天套壳** — 是完整的智能体框架，拥有 45+ 工具、子代理派生和多层权限系统
- **108 个 Feature Gate** 控制 KAIROS（自主模式）、Coordinator（多智能体）和 9 个未发布工具的访问
- **核心智能体循环**（`query.ts`，1,729 行）有 7 个恢复分支处理 Token 限制、压缩和错误状态
- **权限模型**经过 规则 → 分类器 → Hook → 用户提示 的级联决策，有 8 种决策原因类型

---

## 分析规模

| 指标 | 数值 |
|------|------|
| TypeScript 源文件数 | **1,884** |
| 总代码行数 | **~513,000 LOC** |
| 工具实现数量 | **45+** |
| 斜杠命令数量 | **80+** |
| 服务模块数 | **38** |
| MCP 传输协议 | **6 种** |
| Hook 事件类型 | **15+** |
| Feature Gate 模块 | **108 个** |
| 文档化的 Prompt 模式 | **15 个** |
| 工作流食谱 | **10 个** |
| Hook 食谱 | **6 个** |
| 自定义 Agent 模板 | **3 个** |

---

## 阅读白皮书

| 格式 | 英文 | 中文 |
|------|------|------|
| Markdown | [Claude_Code_Whitepaper_EN.md](docs/en/Claude_Code_Whitepaper_EN.md) | [Claude_Code_Whitepaper.md](Claude_Code_Whitepaper.md) |
| PDF | [Claude_Code_Whitepaper_EN.pdf](Claude_Code_Whitepaper_EN.pdf) | [Claude_Code_Whitepaper.pdf](Claude_Code_Whitepaper.pdf) |

**阅读路线建议：**
- **"我想用好 Claude Code"** → 从第 14 章开始，然后第 21 章，再看第 15-16 章
- **"我想理解内部实现"** → 从第 1 章开始，按顺序读到第 13 章
- **"我遇到问题了"** → 直接跳到第 22 章

---

## 仓库结构

```
claude-code-internals/
├── README.md                           # 英文 README
├── README_CN.md                        # 本文件（中文）
├── LICENSE                             # MIT 许可证
├── .gitignore
│
├── Claude_Code_Whitepaper.md           # 完整白皮书 — 中文（3,477 行）
├── Claude_Code_Whitepaper.pdf          # PDF — 中文
├── Claude_Code_Whitepaper_EN.pdf       # PDF — 英文
│
└── docs/
    └── en/
        └── Claude_Code_Whitepaper_EN.md  # 完整白皮书 — 英文（3,501 行）
```

---

## 源代码仓库

本分析基于以下公开源代码：

- **[sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code)** — Claude Code 提取的源代码
- **[ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap)** — Claude Code 的 Sourcemap 重建

感谢以上仓库的维护者。

---

## 许可证

[MIT License](LICENSE)。白皮书内容为原创作品。所分析的源代码归 Anthropic 所有。

---

## 参与贡献

发现错误？有想法分享？想分析新版本？

1. Fork → Branch → PR
2. 欢迎提 Issue

---

<p align="center">
  <b>如果这份资料帮你掌握了 Claude Code，请给个 ⭐ 让更多人发现它。</b>
</p>
