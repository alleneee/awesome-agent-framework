# 问题 8：多 agent 之间如何可靠通信

## 问题定义

只要系统里有多个 agent，就会出现协作问题：

- 怎么给另一个 agent 发消息？
- 怎么知道消息有没有被收到？
- 怎么避免不同 agent 相互覆盖上下文？
- 怎么保证通信可恢复、可重放？

## 给小白的直白解释

多 agent 协作最容易掉进一个误区：

“反正都在一个系统里，大家共享状态就行。”

这是非常脆弱的。

更稳的做法像团队协作：

- 每个人有自己的工作上下文
- 需要沟通时发消息
- 消息进入对方收件箱或队列
- 对方在合适的时机消费

## 为什么这是 agent 工程问题

如果多 agent 直接共享同一个上下文对象：

- 容易互相污染
- 很难跨进程
- 很难恢复
- 很难保证消息顺序

## 本项目答案

本项目主要有两类通信方式：

1. `mailbox`
   更适合团队成员、进程内 teammate、跨地址路由。
2. `pendingMessages`
   更适合本地后台 agent 在运行过程中被注入消息。

这两套机制解决的是不同粒度的问题。

## mailbox 路由

在 [src/tools/SendMessageTool/SendMessageTool.ts](/Users/niko/claude-code-public/src/tools/SendMessageTool/SendMessageTool.ts#L149)，普通消息通过 `writeToMailbox()` 写入目标收件箱。

```ts
await writeToMailbox(
  recipientName,
  {
    from: senderName,
    text: content,
    summary,
    timestamp: new Date().toISOString(),
    color: senderColor,
  },
  teamName,
)
```

这说明消息不是直接塞进对方 prompt，而是先进入一个外部消息通道。

## pendingMessages 队列

本地后台 agent 的轻量注入使用 `pendingMessages`。

入队在 [src/tasks/LocalAgentTask/LocalAgentTask.tsx](/Users/niko/claude-code-public/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L162)：

```ts
export function queuePendingMessage(taskId: string, msg: string, setAppState: ...) {
  updateTaskState(taskId, setAppState, task => ({
    ...task,
    pendingMessages: [...task.pendingMessages, msg]
  }))
}
```

排空在 [src/tasks/LocalAgentTask/LocalAgentTask.tsx](/Users/niko/claude-code-public/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L181)：

```ts
export function drainPendingMessages(taskId: string, getAppState: ..., setAppState: ...): string[] {
  const task = getAppState().tasks[taskId]
  if (!isLocalAgentTask(task) || task.pendingMessages.length === 0) {
    return []
  }
  const drained = task.pendingMessages
  updateTaskState(taskId, setAppState, t => ({ ...t, pendingMessages: [] }))
  return drained
}
```

这个模型本质上是生产者-消费者。

## 为什么不是立刻打断对方 prompt

因为 agent 正在一轮工具执行或流式响应中时，直接把消息插进去可能破坏消息顺序或 API 结构。  
所以更稳的做法是：

- 先排队
- 在合适边界 drain
- 作为下一轮可消费输入进入

这在 query loop 这种多阶段状态机里尤其重要。

## 这里还要补一个更大的图景

多 agent 协作不只是“发消息”，还包括“协作模式”：

多 agent 协作不只是“发消息”，还包括“协作模式”。

在这个项目里，主要有两种：

1. `Coordinator 模式`
   中心化调度，多 worker 分工。
2. `Fork 模式`
   去中心化分叉，共享父上下文。

这意味着通信系统也要适配不同协作模式，而不是只有一种简单通道。

## 设计取舍

这套设计的优点：

- 通信更稳
- agent 边界更清晰
- 支持不同粒度的协作

代价：

- 需要维护 mailbox、队列、通知三种概念
- 消息不是“即时共享”，而是“时机驱动消费”

## 可复用方案

如果你自己做多 agent 通信，建议分三层：

1. `控制消息`
   比如 shutdown、plan approval。
2. `协作消息`
   比如“请继续做 X”。
3. `结果通知`
   比如 task 完成、输出就绪。

并且区分：

- 跨 agent / 跨进程：邮箱或消息总线
- 单 agent 中途注入：本地队列

一句话总结：

多 agent 通信最重要的不是“能发消息”，而是“消息能在正确时机、以正确顺序、在正确边界被消费”。

## 延伸阅读

继续看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
