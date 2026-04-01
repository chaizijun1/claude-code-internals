# Claude Code Internals

### A Deep Dive into the Architecture, Internals, and Design Patterns of Anthropic's AI Coding Agent

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Whitepaper](https://img.shields.io/badge/Whitepaper-English-green.svg)](docs/en/Claude_Code_Whitepaper_EN.md)
[![Whitepaper](https://img.shields.io/badge/Whitepaper-Chinese-red.svg)](Claude_Code_Whitepaper.md)
[![PDF EN](https://img.shields.io/badge/PDF-English-orange.svg)](Claude_Code_Whitepaper_EN.pdf)
[![PDF CN](https://img.shields.io/badge/PDF-Chinese-orange.svg)](Claude_Code_Whitepaper.pdf)

---

## Introduction

This repository contains a **comprehensive whitepaper** analyzing the architecture and internals of **Claude Code v2.1.88** -- Anthropic's official AI-powered CLI coding agent.

The analysis is based entirely on publicly available source code repositories. It covers everything from the high-level architecture and agent loop down to individual tool implementations, permission models, context management strategies, and hidden feature gates.

Whether you are an AI engineer studying agent design patterns, a Claude Code power user looking to unlock advanced workflows, or a researcher interested in how production-grade AI agents are built, this whitepaper provides the most thorough technical breakdown available.

**Scale of what we analyzed:**

| Metric | Value |
|--------|-------|
| TypeScript source files | **1,884** |
| Total lines of code | **~513,000 LOC** |
| Tool implementations | **45+** |
| Slash commands | **80+** |
| Service modules | **38** |
| MCP transport protocols | **6** |
| Hook event types | **15+** |
| Feature gate modules | **108** |

---

## Table of Contents (Whitepaper Chapters)

The whitepaper is organized into **20 chapters** covering the full breadth of Claude Code's internals:

| # | Chapter | Description |
|---|---------|-------------|
| 1 | **Overview & Background** | What Claude Code is, its technical scale, and technology stack |
| 2 | **Project Structure Panorama** | Full directory layout, module organization, and dependency graph |
| 3 | **Core Architecture Deep Dive** | The main agent loop, query engine, streaming execution, and state management |
| 4 | **Tool System** | All 45+ tool implementations -- Bash, File I/O, Search, Web, MCP bridge, and more |
| 5 | **Permission & Security Model** | Multi-layer permission system, trust zones, allow/deny rules |
| 6 | **Hook Event System** | User-injectable hooks at key lifecycle points, execution model |
| 7 | **MCP Protocol Integration** | Model Context Protocol client, 6 transport types, OAuth, tool bridging |
| 8 | **Configuration System** | Hierarchical config (global/project/session), CLAUDE.md parsing, settings resolution |
| 9 | **Context Management & Compaction** | Token budgeting, automatic context compression, conversation pruning |
| 10 | **Agent System** | Sub-agent spawning, multi-agent coordination, task delegation patterns |
| 11 | **System Prompt Construction** | Dynamic prompt assembly, section caching, capability injection |
| 12 | **API Communication & Retry** | Anthropic Messages API client, retry logic, error handling, streaming |
| 13 | **Build System & Compilation** | esbuild pipeline, custom transforms, bundle optimization |
| 14 | **Effective Usage (Beginner)** | Getting started, basic workflows, essential commands |
| 15 | **Effective Usage (Intermediate)** | Advanced patterns, custom hooks, MCP servers, multi-file operations |
| 16 | **Effective Usage (Expert)** | Power-user techniques, headless mode, CI/CD integration, orchestration |
| 17 | **Hidden Features & Feature Gates** | Internal feature flags, 108 gate modules, undocumented capabilities |
| 18 | **Architecture Design Patterns** | Summary of key design patterns used throughout the codebase |
| 19 | **Differences from Standard Claude API** | What makes Claude Code fundamentally different from raw API access |
| 20 | **Future Roadmap & Outlook** | Upcoming features, architectural evolution, ecosystem trends |

Plus an **Appendix** with reference tables, glossary, and supplementary data.

---

## Key Findings

- **Claude Code is not a chatbot wrapper** -- it is a full-fledged agent framework with 45+ tools, sub-agent spawning, and a sophisticated permission system.
- **The core agent loop** (`query.ts`, ~1,729 lines) orchestrates streaming tool execution with real-time permission checks and context budgeting.
- **108 feature gates** control access to internal capabilities, many of which are undocumented and inaccessible to external users.
- **The MCP integration** supports 6 transport protocols (stdio, SSE, HTTP streaming, Docker, npx, and custom) with full OAuth 2.1 authentication.
- **Context compaction** automatically compresses conversations when token limits are approached, preserving critical information through an LLM-based summarization pass.
- **The permission model** operates across 4 trust zones with granular allow/deny rules, preventing destructive operations unless explicitly authorized.
- **System prompts are dynamically assembled** from multiple sections with prompt caching for efficiency, totaling thousands of tokens of behavioral instructions.
- **The React/Ink terminal UI** renders a full interactive interface in the terminal, complete with state management and component lifecycle hooks.

---

## Repository Structure

```
claudeopen/
├── README.md                           # This file (English)
├── README_CN.md                        # Chinese version of this README
├── LICENSE                             # MIT License
├── .gitignore
│
├── Claude_Code_Whitepaper.md           # Full whitepaper (Chinese)
├── Claude_Code_Whitepaper.pdf          # PDF version (Chinese)
├── Claude_Code_Whitepaper_EN.pdf       # PDF version (English)
│
└── docs/
    └── en/
        └── Claude_Code_Whitepaper_EN.md  # Full whitepaper (English)
```

---

## How to Read

**Choose your preferred format and language:**

| Format | English | Chinese |
|--------|---------|---------|
| Markdown | [Claude_Code_Whitepaper_EN.md](docs/en/Claude_Code_Whitepaper_EN.md) | [Claude_Code_Whitepaper.md](Claude_Code_Whitepaper.md) |
| PDF | [Claude_Code_Whitepaper_EN.pdf](Claude_Code_Whitepaper_EN.pdf) | [Claude_Code_Whitepaper.pdf](Claude_Code_Whitepaper.pdf) |

The whitepaper is designed to be read sequentially (Chapters 1-13 for architecture, 14-16 for usage, 17-20 for advanced topics), but each chapter is self-contained and can be read independently.

---

## Source Repositories

This analysis is based on publicly available source code from the following repositories:

- **[sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code)** -- Extracted source code of Claude Code
- **[ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap)** -- Source map reconstruction of Claude Code

All credit for making the source code accessible goes to the maintainers of these repositories. This project provides analysis and commentary only.

---

## License

This project is licensed under the [MIT License](LICENSE).

The whitepaper and all analysis content in this repository are original work. The source code analyzed belongs to Anthropic and is subject to its own licensing terms.

---

## Contributing

Contributions are welcome! If you find errors, want to add analysis of newer versions, or have insights to share:

1. Fork this repository
2. Create a feature branch
3. Submit a pull request

If this project helped you understand Claude Code better, please consider giving it a star -- it helps others discover this resource.

---

<p align="center">
  <b>If you find this analysis useful, please give it a star to help others find it.</b>
</p>
