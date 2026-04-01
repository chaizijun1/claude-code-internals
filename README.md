# Claude Code Internals

### Architecture Deep Dive + Power User Guide for Anthropic's AI Coding Agent

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Whitepaper](https://img.shields.io/badge/Whitepaper-English-green.svg)](docs/en/Claude_Code_Whitepaper_EN.md)
[![Whitepaper](https://img.shields.io/badge/Whitepaper-Chinese-red.svg)](Claude_Code_Whitepaper.md)
[![PDF EN](https://img.shields.io/badge/PDF-English-orange.svg)](Claude_Code_Whitepaper_EN.pdf)
[![PDF CN](https://img.shields.io/badge/PDF-Chinese-orange.svg)](Claude_Code_Whitepaper.pdf)

[English](README.md) | [中文](README_CN.md)

---

## Why This Repo?

Claude Code is far more powerful than most users realize. This whitepaper reveals **exactly how it works internally** and — more importantly — **how to use that knowledge to 10x your productivity**.

Based on a deep analysis of **513,000 lines of TypeScript source code** (v2.1.88), this is both:

- **An architecture reference** — understand the agent loop, tool system, permission model, and 108 hidden feature gates
- **A practical power-user guide** — 15 prompt patterns, 10 workflow recipes, 6 hook automations, custom agents, cost optimization, and more

---

## Quick Start: What Can I Learn?

### If you just want to use Claude Code better, start here:

| Chapter | What You'll Learn | Audience |
|---------|-------------------|----------|
| [Ch.14 — Beginner Tips](docs/en/Claude_Code_Whitepaper_EN.md) | First setup, CLAUDE.md, permission rules, 6 usage patterns, slash commands | New users |
| [Ch.15 — Intermediate Tips](docs/en/Claude_Code_Whitepaper_EN.md) | **15 prompt patterns**, CLAUDE.md masterclass, 6 hook recipes, MCP servers, agent parallel work, git workflows | Regular users |
| [Ch.16 — Expert Tips](docs/en/Claude_Code_Whitepaper_EN.md) | Token budget mastery, cost optimization, CI/CD integration, custom agents, SDK mode, multi-project management | Power users |
| [Ch.21 — Workflow Recipes](docs/en/Claude_Code_Whitepaper_EN.md) | **10 complete real-world recipes**: full-stack dev, bug investigation, legacy migration, security audit, perf optimization, and more | All levels |
| [Ch.22 — Troubleshooting](docs/en/Claude_Code_Whitepaper_EN.md) | Top 10 issues, permission/performance debugging, CLAUDE.md pitfalls, hook debugging, best practices checklist | When stuck |

### If you want to understand the internals:

| Chapter | Topic |
|---------|-------|
| Ch.1-2 | Overview & project structure (1,884 files, 38 services) |
| Ch.3 | **Core agent loop** — the `query.ts` heart of the system (1,729 lines) |
| Ch.4 | Tool system — all 45+ tools, registry, execution pipeline |
| Ch.5 | Permission & security model — 5 modes, YOLO classifier, sandbox |
| Ch.6 | Hook event system — 15+ event types, 3 handler types |
| Ch.7 | MCP integration — 6 transport protocols, OAuth, tool bridging |
| Ch.8 | Configuration — 4-level hierarchy, CLAUDE.md parsing |
| Ch.9 | Context management — token budgets, auto-compaction, prompt caching |
| Ch.10 | Agent system — sub-agents, coordinator mode, worktree isolation |
| Ch.11 | System prompt construction — dynamic assembly, caching |
| Ch.12 | API communication — retry logic, streaming, error handling |
| Ch.13 | Build system — esbuild, feature gate DCE, 108 eliminated modules |
| Ch.17 | **Hidden features** — KAIROS, Coordinator Mode, 9 unreleased tools |
| Ch.18-20 | Design patterns, Claude API differences, future roadmap |

---

## Highlights

### Usage Tips Highlights

- **15 Prompt Engineering Patterns** — precision targeting, context preloading, TDD, batch operations, role-setting, constraint-driven, and 9 more
- **10 Real-World Workflow Recipes** — step-by-step guides for full-stack features, bug investigation, legacy migration, security audits, database changes, perf optimization, doc generation, test suites, dependency upgrades
- **6 Hook Automation Recipes** — auto-format, auto-lint, sensitive file protection, commit message enforcement, TypeScript type gate, auto-test
- **3 Custom Agent Templates** — security auditor, database expert, documentation writer
- **Cost Optimization Formula** — derived from source code, with task-type cost estimates
- **CI/CD Integration** — GitHub Actions YAML for automated Claude Code reviews
- **CLAUDE.md Masterclass** — templates for React, Python, and DevOps projects
- **Troubleshooting Guide** — top 10 issues with root cause analysis and fixes

### Architecture Highlights

- **Claude Code is not a chatbot wrapper** — it is a full agent framework with 45+ tools, sub-agent spawning, and a multi-layer permission system
- **108 feature gates** control access to internal capabilities like KAIROS (autonomous mode), Coordinator (multi-agent), and 9 unreleased tools
- **The core agent loop** (`query.ts`, 1,729 lines) has 7 recovery branches for handling token limits, compaction, and error states
- **The permission model** cascades through rules → classifier → hooks → user prompt, with 8 decision reason types

---

## Scale of Analysis

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
| Prompt patterns documented | **15** |
| Workflow recipes | **10** |
| Hook recipes | **6** |
| Custom agent templates | **3** |

---

## Read the Whitepaper

| Format | English | Chinese |
|--------|---------|---------|
| Markdown | [Claude_Code_Whitepaper_EN.md](docs/en/Claude_Code_Whitepaper_EN.md) | [Claude_Code_Whitepaper.md](Claude_Code_Whitepaper.md) |
| PDF | [Claude_Code_Whitepaper_EN.pdf](Claude_Code_Whitepaper_EN.pdf) | [Claude_Code_Whitepaper.pdf](Claude_Code_Whitepaper.pdf) |

**Reading paths:**
- **"I want to use Claude Code better"** → Start at Chapter 14, then 21, then 15-16
- **"I want to understand the internals"** → Start at Chapter 1, read through 13
- **"I'm having issues"** → Jump to Chapter 22

---

## Repository Structure

```
claude-code-internals/
├── README.md                           # This file (English)
├── README_CN.md                        # Chinese README
├── LICENSE                             # MIT License
├── .gitignore
│
├── Claude_Code_Whitepaper.md           # Full whitepaper — Chinese (3,477 lines)
├── Claude_Code_Whitepaper.pdf          # PDF — Chinese
├── Claude_Code_Whitepaper_EN.pdf       # PDF — English
│
└── docs/
    └── en/
        └── Claude_Code_Whitepaper_EN.md  # Full whitepaper — English (3,501 lines)
```

---

## Source Repositories

This analysis is based on publicly available source code:

- **[sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code)** — Extracted source code of Claude Code
- **[ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap)** — Sourcemap reconstruction of Claude Code

All credit for making the source code accessible goes to the maintainers of these repositories.

---

## License

[MIT License](LICENSE). The whitepaper content is original work. The analyzed source code belongs to Anthropic.

---

## Contributing

Found an error? Have insights to add? Want to analyze a newer version?

1. Fork → Branch → PR
2. Issues welcome

---

<p align="center">
  <b>If this helped you master Claude Code, give it a ⭐ so others can find it too.</b>
</p>
