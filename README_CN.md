# Claude Code Internals

### 深入剖析 Anthropic AI 编程智能体的架构、内部实现与设计模式

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![白皮书](https://img.shields.io/badge/白皮书-中文版-red.svg)](Claude_Code_Whitepaper.md)
[![Whitepaper](https://img.shields.io/badge/Whitepaper-English-green.svg)](docs/en/Claude_Code_Whitepaper_EN.md)
[![PDF 英文](https://img.shields.io/badge/PDF-英文版-orange.svg)](Claude_Code_Whitepaper_EN.pdf)
[![PDF 中文](https://img.shields.io/badge/PDF-中文版-orange.svg)](Claude_Code_Whitepaper.pdf)

[English](README.md) | **中文**

---

## 简介

本仓库包含一份**全面的技术白皮书**，深入分析 **Claude Code v2.1.88** 的架构与内部实现 -- Anthropic 官方推出的 AI 编程 CLI 智能体。

本分析完全基于公开可用的源代码仓库，涵盖从高层架构和智能体循环到具体工具实现、权限模型、上下文管理策略以及隐藏的 Feature Gate 等方方面面。

无论你是研究智能体设计模式的 AI 工程师、希望解锁高级使用技巧的 Claude Code 深度用户，还是对生产级 AI 智能体构建方式感兴趣的研究者，这份白皮书都提供了目前最全面的技术解析。

**分析规模一览：**

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

---

## 目录（白皮书章节）

白皮书共 **20 个章节**，全面覆盖 Claude Code 的内部实现：

| # | 章节 | 描述 |
|---|------|------|
| 1 | **概述与背景** | Claude Code 是什么，技术规模与技术栈 |
| 2 | **项目结构全景** | 完整目录布局、模块组织与依赖关系 |
| 3 | **核心架构深度解析** | 主智能体循环、查询引擎、流式执行与状态管理 |
| 4 | **工具系统** | 全部 45+ 工具实现 -- Bash、文件 I/O、搜索、Web、MCP 桥接等 |
| 5 | **权限与安全模型** | 多层权限系统、信任区域、允许/拒绝规则 |
| 6 | **Hook 事件系统** | 用户可注入的生命周期钩子、执行模型 |
| 7 | **MCP 协议集成** | Model Context Protocol 客户端、6 种传输类型、OAuth、工具桥接 |
| 8 | **配置体系** | 层级配置（全局/项目/会话）、CLAUDE.md 解析、设置解析 |
| 9 | **上下文管理与压缩** | Token 预算、自动上下文压缩、对话裁剪 |
| 10 | **Agent 智能体系统** | 子智能体生成、多智能体协调、任务委托模式 |
| 11 | **System Prompt 构造机制** | 动态 Prompt 组装、分段缓存、能力注入 |
| 12 | **API 通信与重试机制** | Anthropic Messages API 客户端、重试逻辑、错误处理、流式传输 |
| 13 | **构建系统与编译优化** | esbuild 流水线、自定义转换、打包优化 |
| 14 | **高效使用技巧（入门篇）** | 快速上手、基础工作流、核心命令 |
| 15 | **高效使用技巧（进阶篇）** | 高级模式、自定义 Hook、MCP 服务器、多文件操作 |
| 16 | **高效使用技巧（精通篇）** | 高级用户技巧、无头模式、CI/CD 集成、编排 |
| 17 | **内部隐藏功能与 Feature Gate** | 内部功能标志、108 个 Gate 模块、未公开功能 |
| 18 | **架构设计模式总结** | 代码库中使用的关键设计模式汇总 |
| 19 | **与标准 Claude API 的本质差异** | Claude Code 与原始 API 访问的根本区别 |
| 20 | **未来路线图与展望** | 即将推出的功能、架构演进、生态趋势 |

另附**附录**，包含参考表格、术语表及补充数据。

---

## 核心发现

- **Claude Code 不是一个聊天机器人套壳** -- 它是一个成熟的智能体框架，拥有 45+ 工具、子智能体生成能力和精密的权限系统。
- **核心智能体循环**（`query.ts`，约 1,729 行）协调流式工具执行，同时进行实时权限检查和上下文预算管理。
- **108 个 Feature Gate** 控制内部功能的访问，其中许多未公开文档化，外部用户无法访问。
- **MCP 集成**支持 6 种传输协议（stdio、SSE、HTTP 流、Docker、npx 和自定义），并具备完整的 OAuth 2.1 认证。
- **上下文压缩**在接近 Token 限制时自动压缩对话，通过基于 LLM 的摘要保留关键信息。
- **权限模型**在 4 个信任区域中运行，具有细粒度的允许/拒绝规则，防止未经授权的破坏性操作。
- **System Prompt 是动态组装的**，来自多个分段并使用 Prompt 缓存提高效率，总计数千 Token 的行为指令。
- **React/Ink 终端 UI** 在终端中渲染完整的交互式界面，具备状态管理和组件生命周期钩子。

---

## 仓库结构

```
claudeopen/
├── README.md                           # 英文 README
├── README_CN.md                        # 本文件（中文）
├── LICENSE                             # MIT 许可证
├── .gitignore
│
├── Claude_Code_Whitepaper.md           # 完整白皮书（中文）
├── Claude_Code_Whitepaper.pdf          # PDF 版本（中文）
├── Claude_Code_Whitepaper_EN.pdf       # PDF 版本（英文）
│
└── docs/
    └── en/
        └── Claude_Code_Whitepaper_EN.md  # 完整白皮书（英文）
```

---

## 阅读指南

**选择你偏好的格式和语言：**

| 格式 | 英文 | 中文 |
|------|------|------|
| Markdown | [Claude_Code_Whitepaper_EN.md](docs/en/Claude_Code_Whitepaper_EN.md) | [Claude_Code_Whitepaper.md](Claude_Code_Whitepaper.md) |
| PDF | [Claude_Code_Whitepaper_EN.pdf](Claude_Code_Whitepaper_EN.pdf) | [Claude_Code_Whitepaper.pdf](Claude_Code_Whitepaper.pdf) |

白皮书设计为可顺序阅读（第 1-13 章为架构分析，第 14-16 章为使用技巧，第 17-20 章为高级主题），但每章都是独立的，可以单独阅读。

---

## 源代码仓库

本分析基于以下公开可用的源代码仓库：

- **[sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code)** -- Claude Code 提取的源代码
- **[ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap)** -- Claude Code 的 Source Map 重建

感谢以上仓库的维护者使源代码可以公开获取。本项目仅提供分析和评论。

---

## 许可证

本项目采用 [MIT 许可证](LICENSE)。

本仓库中的白皮书及所有分析内容为原创作品。所分析的源代码归 Anthropic 所有，受其自身许可条款约束。

---

## 参与贡献

欢迎贡献！如果你发现错误、想要添加新版本的分析或有见解想要分享：

1. Fork 本仓库
2. 创建功能分支
3. 提交 Pull Request

如果这个项目帮助你更好地理解了 Claude Code，请考虑给一个 Star -- 这有助于更多人发现这个资源。

---

<p align="center">
  <b>如果你觉得这份分析有用，请给个 Star 帮助更多人发现它。</b>
</p>
