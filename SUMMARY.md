# 目录

## 第一部分：基础架构

* [第1章 - 项目概述与架构全景](第01章-项目概述与架构全景.md)
* [第2章 - 构建系统与启动流程](第02章-构建系统与启动流程.md)

## 第二部分：终端 UI 层

* [第3章 - 终端UI渲染引擎](第03章-终端UI渲染引擎.md)
* [第4章 - 事件系统与输入处理](第04章-事件系统与输入处理.md)

## 第三部分：核心 AI 引擎

* [第5章 - 核心对话引擎 QueryEngine](第05章-核心对话引擎QueryEngine.md)
* [第6章 - LLM API 调用与流式处理](第06章-LLM-API调用与流式处理.md)
* [第8章 - 上下文管理与压缩策略](第08章-上下文管理与压缩策略.md)

## 第四部分：工具生态

* [第7章 - 工具系统架构与实现](第07章-工具系统架构与实现.md)

## 第五部分：Agent 与任务

* [第9章 - 任务系统与多Agent架构](第09章-任务系统与多Agent架构.md)
* [第10章 - 团队协作与远程通信](第10章-团队协作与远程通信.md)

## 第六部分：扩展与基础设施

* [第11章 - 插件、Skills与Hook系统](第11章-插件Skills与Hook系统.md)
* [第12章 - 状态管理、安全模型与基础设施](第12章-状态管理安全模型与基础设施.md)

## 第七部分：Agent 工程问题专题

* [专题导读](Agent工程问题专题/README.md)
* [问题1 - 长期记忆应该放在哪里](Agent工程问题专题/01-long-term-memory.md)
* [问题2 - 会话中断后如何恢复](Agent工程问题专题/02-session-resume.md)
* [问题3 - 窗口快爆了时如何处理上下文](Agent工程问题专题/03-context-window-management.md)
* [问题4 - 压缩后如何补回真正的任务状态](Agent工程问题专题/04-post-compact-task-restoration.md)
* [问题5 - 计划如何持久化并参与恢复](Agent工程问题专题/05-plan-persistence.md)
* [问题6 - 后台 agent 状态如何管理](Agent工程问题专题/06-background-agent-state.md)
* [问题7 - 子 agent 如何恢复运行](Agent工程问题专题/07-subagent-resume.md)
* [问题8 - 多 agent 之间如何可靠通信](Agent工程问题专题/08-multi-agent-messaging.md)
* [问题9 - 异常、循环过长和输出截断如何收口](Agent工程问题专题/09-failure-recovery-and-loop-bounds.md)
* [问题10 - 消息结构和工具调用不变量如何维护](Agent工程问题专题/10-message-invariants.md)
* [Claude Code 系统总览](Agent工程问题专题/11-claude-code-system-overview.md)

## 附录

* [附录A - 核心提示词工程全解析](附录A-核心提示词工程全解析.md)
