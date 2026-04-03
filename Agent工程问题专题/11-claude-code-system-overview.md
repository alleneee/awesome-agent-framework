# Claude Code 系统总览

这篇文档把原来模块化拆开的 Claude Code 系统说明合并成一篇，目标是让你在一个地方建立完整认知。

如果前面的 `01` 到 `10` 更像“问题驱动”的学习文档，这一篇更像“系统结构总览”。

## 这篇文档讲什么

它回答 8 类系统级问题：

1. Agent 生命周期怎么管理
2. 多 Agent 怎么协作
3. 工具系统怎么组织
4. 上下文怎么压缩
5. query loop 怎么运行
6. 错误怎么恢复
7. 权限怎么控制
8. 状态怎么持久化和恢复

## 一张总图

```text
User Input
  |
  v
QueryEngine.submitMessage()
  |
  +--> processUserInput()         -- 处理 slash 命令、附件、上下文
  |
  +--> query() / queryLoop()      -- 主循环
  |      |
  |      +--> queryModelWithStreaming()   -- 调用模型
  |      +--> StreamingToolExecutor        -- 流式期间提前执行工具
  |      +--> runTools()                   -- 工具编排
  |      +--> autoCompact / reactiveCompact
  |      +--> stop / continue / recover
  |
  +--> transcript persistence     -- 写会话日志
  |
  +--> task system                -- 后台 agent / shell / teammate
```

---

## 1. Agent 生命周期

### 核心问题

Agent 怎么被创建、启动、运行、终止和回收。

### 本项目做法

Claude Code 用统一的 `Task` / `TaskState` 体系管理 agent 生命周期。

核心类型在 [src/Task.ts](/Users/niko/claude-code-public/src/Task.ts#L6)：

```ts
export type TaskType =
  | 'local_bash'
  | 'local_agent'
  | 'remote_agent'
  | 'in_process_teammate'
  | 'local_workflow'
  | 'monitor_mcp'
  | 'dream'
```

状态机在同文件：

```ts
export type TaskStatus =
  | 'pending'
  | 'running'
  | 'completed'
  | 'failed'
  | 'killed'
```

### 为什么重要

如果没有统一状态机：

- 无法稳定终止
- 无法可靠通知
- 无法统一回收

### 对应问题文档

- [06-background-agent-state.md](./06-background-agent-state.md)
- [07-subagent-resume.md](./07-subagent-resume.md)

---

## 2. 多 Agent 协作

### 核心问题

多个 agent 怎么分工、通信、回传结果。

### 本项目做法

Claude Code 主要支持两种协作模式：

1. `Coordinator 模式`
   中心化调度，多 worker 分工。
2. `Fork 模式`
   去中心化分叉，继承父上下文。

通信方式主要有两类：

- mailbox
- pending message queue

`SendMessageTool` 在 [src/tools/SendMessageTool/SendMessageTool.ts](/Users/niko/claude-code-public/src/tools/SendMessageTool/SendMessageTool.ts#L149) 使用 `writeToMailbox()` 路由消息。

### 为什么重要

多 agent 协作如果只是共享上下文，很容易：

- 相互污染
- 难以恢复
- 顺序混乱

### 对应问题文档

- [08-multi-agent-messaging.md](./08-multi-agent-messaging.md)

---

## 3. 工具系统

### 核心问题

工具如何注册、发现、并发执行、做权限检查。

### 本项目做法

虽然本目录的 10 篇问题文档没有单独拆“工具系统”，但在总图里它很关键。

主循环里，模型产生 `tool_use` 后，会进入工具编排逻辑。  
工具执行不是简单串行，而是区分并发安全和非并发安全。

在流式场景下，还会由 `StreamingToolExecutor` 提前开始执行已经完整的工具调用，减少等待时间。

### 为什么重要

工具系统是 agent 从“会聊天”变成“能做事”的核心层。

### 可以顺带关注的源码

- [src/Tool.ts](/Users/niko/claude-code-public/src/Tool.ts)
- [src/tools.ts](/Users/niko/claude-code-public/src/tools.ts)
- [src/services/tools/StreamingToolExecutor.ts](/Users/niko/claude-code-public/src/services/tools/StreamingToolExecutor.ts)

---

## 4. 上下文管理与压缩

### 核心问题

长对话怎么活下去，而不把真正任务压坏。

### 本项目做法

Claude Code 使用渐进式压缩管线：

1. Tool Result Budget
2. Snip
3. Microcompact
4. Context Collapse
5. Autocompact

主入口在 [src/query.ts](/Users/niko/claude-code-public/src/query.ts#L365)。  
阈值逻辑在 [src/services/compact/autoCompact.ts](/Users/niko/claude-code-public/src/services/compact/autoCompact.ts#L67)。

压缩后不会只留 summary，而是会补回：

- plan
- skills
- 最近文件
- 异步 agent 状态

### 为什么重要

上下文管理不是“为了省 token”，而是“为了让任务能继续”。

### 对应问题文档

- [03-context-window-management.md](./03-context-window-management.md)
- [04-post-compact-task-restoration.md](./04-post-compact-task-restoration.md)

---

## 5. Query Loop 与流式处理

### 核心问题

主循环如何把：

- 模型调用
- 工具执行
- 错误恢复
- 压缩
- 继续或停止

串成一个可持续运行的状态机。

### 本项目做法

核心循环在 [src/query.ts](/Users/niko/claude-code-public/src/query.ts)。

它大体上做四件事：

1. 调模型
2. 执行工具
3. 检查 compact / recovery
4. 决定下一轮是否继续

`QueryEngine.submitMessage()` 是对话级入口，位于 [src/QueryEngine.ts](/Users/niko/claude-code-public/src/QueryEngine.ts#L209)。

### 为什么重要

query loop 才是整个 agent 的“心跳”。

没有稳定的 loop，就没有：

- 长任务
- 工具链
- 流式交互
- 恢复逻辑

### 对应问题文档

- [02-session-resume.md](./02-session-resume.md)
- [09-failure-recovery-and-loop-bounds.md](./09-failure-recovery-and-loop-bounds.md)
- [10-message-invariants.md](./10-message-invariants.md)

---

## 6. 错误恢复

### 核心问题

错误发生后，是重试、降级、压缩、续写还是停止。

### 本项目做法

Claude Code 的错误恢复是多层的：

- API 层：`withRetry`
- query loop 层：`prompt_too_long`、`max_output_tokens`、`maxTurns`
- compact 层：熔断和反应式压缩

特别值得注意的是：

- `max_output_tokens` 会注入一条“直接继续，不要 recap”的恢复消息
- `prompt_too_long` 会避免把 stop hooks 继续叠加成死循环

### 为什么重要

恢复策略决定系统在真实环境里是否可用。  
一个只会“失败一次就崩”或者“无限重试”的 agent 都不适合生产。

### 对应问题文档

- [09-failure-recovery-and-loop-bounds.md](./09-failure-recovery-and-loop-bounds.md)

---

## 7. 安全与权限

### 核心问题

工具调用和子 agent 调用如何被限制，避免 agent 做出越权行为。

### 本项目做法

本项目有完整的权限模式与规则体系，虽然当前 `problems/` 目录还没有单开一篇，但从总览角度必须知道它存在。

典型设计包括：

- 多层 permission mode
- 规则来源优先级
- 子 agent 的 bubble 权限模式
- Hook 层拦截

### 为什么重要

没有权限系统的 agent，等于没有边界。

### 可进一步查看的源码

- [src/types/permissions.ts](/Users/niko/claude-code-public/src/types/permissions.ts)
- [src/utils/permissions/](/Users/niko/claude-code-public/src/utils/permissions)

---

## 8. 状态持久化与恢复

### 核心问题

运行时状态、任务状态、对话 transcript、agent metadata 怎么持久化，怎么恢复。

### 本项目做法

Claude Code 用了多层状态：

1. `AppState`
   运行时内存状态中心。
2. `transcript`
   对话和 agent sidechain 的 JSONL 持久化。
3. `task output files`
   后台任务输出。
4. `plan files`
   会话级计划状态。
5. `memory files`
   长期规则和偏好。

### 为什么重要

这套状态分层让系统具备：

- resume 能力
- compact 后继续能力
- 后台 agent 恢复能力
- UI 可见性

### 对应问题文档

- [01-long-term-memory.md](./01-long-term-memory.md)
- [02-session-resume.md](./02-session-resume.md)
- [05-plan-persistence.md](./05-plan-persistence.md)
- [06-background-agent-state.md](./06-background-agent-state.md)
- [07-subagent-resume.md](./07-subagent-resume.md)

---

## 你应该从这篇文档记住什么

如果你把 Claude Code 看成一个“能干活的 agent 操作系统”，它至少由下面几层组成：

1. `对话层`
   QueryEngine 和 query loop
2. `工具层`
   tool schema、权限、执行器
3. `任务层`
   background agent、shell、teammate
4. `上下文层`
   memory、compact、collapse、plan
5. `恢复层`
   transcript、resume、retry、fallback

前面的 `01` 到 `10` 文档是从问题视角拆开看的。  
这一篇则是把这些问题重新拼回一张系统图。
