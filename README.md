# Claude Code 源码教学指南

Claude Code v2.1.88 源码从 0 到 1 的深度教学指南。面向有经验的开发者，大量引用源码片段并逐行讲解。

## 项目概况

- **代码规模**: 1908 个 TypeScript 文件，约 38 万行代码
- **技术栈**: TypeScript + React (Ink) + Bun bundler + Zod v4
- **定位**: Anthropic 官方 CLI 工具，基于 Ink (React for CLI) 构建终端 UI

## 学习路径

建议按以下顺序阅读，从宏观架构到核心引擎，再到扩展系统：

### 第一部分：基础架构

| 章节 | 内容 | 核心文件 |
|------|------|----------|
| [第1章 - 项目概述与架构全景](第01章-项目概述与架构全景.md) | 技术选型、架构分层、模块依赖、设计理念 | `package.json`, `build.ts`, `tsconfig.json` |
| [第2章 - 构建系统与启动流程](第02章-构建系统与启动流程.md) | Bun bundler、CLI 入口、启动链路、REPL/print 分叉 | `build.ts`, `entrypoints/cli.tsx`, `main.tsx`, `setup.ts` |

### 第二部分：终端 UI 层

| 章节 | 内容 | 核心文件 |
|------|------|----------|
| [第3章 - 终端UI渲染引擎](第03章-终端UI渲染引擎.md) | 自定义 React reconciler、DOM 模型、Yoga 布局、双缓冲渲染 | `ink/ink.tsx`, `ink/reconciler.ts`, `ink/dom.ts`, `ink/layout/` |
| [第4章 - 事件系统与输入处理](第04章-事件系统与输入处理.md) | W3C 事件模型、键绑定、Vim 状态机、焦点管理 | `ink/events/`, `keybindings/`, `vim/`, `ink/parse-keypress.ts` |

### 第三部分：核心 AI 引擎

| 章节 | 内容 | 核心文件 |
|------|------|----------|
| [第5章 - 核心对话引擎 QueryEngine](第05章-核心对话引擎QueryEngine.md) | QueryEngine 生命周期、queryLoop 循环、Token 预算、Stop Hooks | `QueryEngine.ts`, `query.ts`, `query/` |
| [第6章 - LLM API 调用与流式处理](第06章-LLM-API调用与流式处理.md) | Raw Stream、流式事件、重试框架、成本追踪、Fallback | `services/api/claude.ts`, `services/api/withRetry.ts` |
| [第8章 - 上下文管理与压缩策略](第08章-上下文管理与压缩策略.md) | 五层渐进式压缩、Autocompact、Token 估算 | `services/compact/`, `utils/tokens.ts` |

### 第四部分：工具生态

| 章节 | 内容 | 核心文件 |
|------|------|----------|
| [第7章 - 工具系统架构与实现](第07章-工具系统架构与实现.md) | Tool 接口、权限模型、BashTool/FileEditTool 深入分析 | `Tool.ts`, `tools.ts`, `tools/` |

### 第五部分：Agent 与任务

| 章节 | 内容 | 核心文件 |
|------|------|----------|
| [第9章 - 任务系统与多Agent架构](第09章-任务系统与多Agent架构.md) | Task 类型体系、Agent 生命周期、Fork Subagent | `Task.ts`, `tasks/`, `tools/AgentTool/` |
| [第10章 - 团队协作与远程通信](第10章-团队协作与远程通信.md) | Team 创建、消息路由、Shutdown 协议、WebSocket | `tools/TeamCreateTool/`, `tools/SendMessageTool/`, `remote/` |

### 第六部分：扩展与基础设施

| 章节 | 内容 | 核心文件 |
|------|------|----------|
| [第11章 - 插件、Skills与Hook系统](第11章-插件Skills与Hook系统.md) | 插件架构、Skills 执行、Hook 机制、MCP 集成、Cron | `plugins/`, `skills/`, `hooks/`, `tools/SkillTool/` |
| [第12章 - 状态管理、安全模型与基础设施](第12章-状态管理安全模型与基础设施.md) | Store 设计、Settings 合并、权限体系、持久化、Bridge | `state/`, `types/permissions.ts`, `history.ts`, `bridge/` |

### 第七部分：Agent 工程问题专题

这一部分不是按源码模块划分，而是按 Agent 开发中的关键工程问题来组织，适合已经读完主章节、准备从“做一个可靠 Agent 系统”角度重新梳理一遍的人。

| 文档 | 内容 | 适合解决的问题 |
|------|------|---------------|
| [专题导读](Agent工程问题专题/README.md) | 学习路线、问题分类、阅读方式 | 从哪里开始、怎么把问题串起来 |
| [Claude Code 系统总览](Agent工程问题专题/11-claude-code-system-overview.md) | 生命周期、协作、工具、上下文、恢复、持久化的整体图景 | 先建立系统级认知 |
| [问题 1~10](Agent工程问题专题/01-long-term-memory.md) | 长期记忆、会话恢复、上下文压缩、计划持久化、后台 Agent、多 Agent 通信、恢复边界等 | 针对单个工程难题深入学习 |

## 英文文件名索引

每章均提供英文命名的同内容副本，方便非中文环境导航：

| English Filename | Chapter |
|------------------|---------|
| [Chapter-01](Chapter-01-Project-Overview-and-Architecture.md) | Project Overview and Architecture |
| [Chapter-02](Chapter-02-Build-System-and-Startup-Flow.md) | Build System and Startup Flow |
| [Chapter-03](Chapter-03-Terminal-UI-Rendering-Engine.md) | Terminal UI Rendering Engine |
| [Chapter-04](Chapter-04-Event-System-and-Input-Handling.md) | Event System and Input Handling |
| [Chapter-05](Chapter-05-Core-Conversation-Engine-QueryEngine.md) | Core Conversation Engine |
| [Chapter-06](Chapter-06-LLM-API-and-Streaming.md) | LLM API and Streaming |
| [Chapter-07](Chapter-07-Tool-System-Architecture.md) | Tool System Architecture |
| [Chapter-08](Chapter-08-Context-Management-and-Compression.md) | Context Management and Compression |
| [Chapter-09](Chapter-09-Task-System-and-Multi-Agent.md) | Task System and Multi-Agent |
| [Chapter-10](Chapter-10-Team-Collaboration-and-Remote-Communication.md) | Team Collaboration and Remote Communication |
| [Chapter-11](Chapter-11-Plugin-Skills-and-Hook-System.md) | Plugin, Skills and Hook System |
| [Chapter-12](Chapter-12-State-Security-and-Infrastructure.md) | State, Security and Infrastructure |

## 代码流程图索引

以下章节包含 Mermaid 格式的代码流程图，帮助可视化理解核心流程：

| 章节 | 流程图 | 描述 |
|------|--------|------|
| 第2章 | 完整启动链路 | CLI 入口 → 快速路径分发 → init() → setup() → REPL |
| 第5章 | submitMessage 生命周期 | 8 阶段异步生成器流程 |
| 第5章 | queryLoop while(true) | 10 步循环体：预处理 → 压缩 → 调用 → 工具执行 → 下一轮 |
| 第6章 | queryModel 五阶段流水线 | 前置检查 → 消息规范化 → 参数构建 → 重试调用 → 流式事件 |
| 第7章 | 工具调用执行流水线 | Schema 解析 → 验证 → 权限 → 执行 → 结果转换 |
| 第8章 | 五层渐进式压缩管道 | L1~L5 逐层压缩 + Reactive Compact 应急 |
| 第9章 | Task 状态机 | pending → running → completed/failed/killed |
| 第9章 | Agent 生命周期 | AgentTool → fork QueryEngine → 通知 → 结果回传 |

> 流程图使用 Mermaid 语法，可在 GitHub、VS Code (Markdown Preview Mermaid) 等环境直接渲染。

## 核心架构图

```
CLI Entry (entrypoints/cli.tsx)
    |
    v
Bootstrap (setup.ts + bootstrap/)
    |
    v
REPL Screen (screens/REPL.tsx)
    |
    |-- Ink Renderer (ink/)            [第3-4章]
    |   |-- Custom React Reconciler
    |   |-- Yoga Layout Engine (TS)
    |   |-- W3C Event System
    |   |-- Vim Mode State Machine
    |
    |-- QueryEngine                    [第5-6章]
    |   |-- query() loop
    |   |-- queryModel() streaming
    |   |-- 5-layer context compression [第8章]
    |
    |-- Tool System (tools/)           [第7章]
    |   |-- 30+ built-in tools
    |   |-- MCP tool integration
    |   |-- Permission model
    |
    |-- Task System (tasks/)           [第9-10章]
    |   |-- Local/Remote/InProcess
    |   |-- Team collaboration
    |   |-- WebSocket protocol
    |
    |-- Extensions                     [第11章]
    |   |-- Plugin system
    |   |-- Skills system
    |   |-- Hook mechanism
    |
    |-- Infrastructure                 [第12章]
    |   |-- 34-line Store
    |   |-- 5-layer Settings
    |   |-- JSONL persistence
    |   |-- Bridge / Remote Control
    |
    v
Anthropic API (services/api/)
```
