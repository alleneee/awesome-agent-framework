# 问题 6：后台 agent 状态如何管理

## 问题定义

一个后台 agent 不是普通函数调用。  
它会持续运行、调用工具、产生输出、被查看、被终止、被恢复，甚至被主线程引用结果。

所以系统必须回答：

- 它的状态放哪？
- 它的输出放哪？
- 它完成时怎么通知别人？
- 它什么时候可以被回收？

## 给小白的直白解释

可以把后台 agent 想成一个“长期工单”。

它不是一句“帮我去做这件事”就结束了，而是有一整套状态：

- 已创建
- 运行中
- 输出写到哪
- 是否被 UI 盯着看
- 是否已经通知过主线程
- 是否还能继续接消息

如果这些状态没有统一管理，后台 agent 很快就会变成不可控的异步黑盒。

## 为什么这是 agent 工程问题

如果后台 agent 只是一个 `Promise`：

- 没法追踪阶段性进度
- 没法安全终止
- 没法为 UI 提供稳定状态
- 没法让父 agent 可靠读取结果

## 本项目答案

本项目把后台 agent 建模成 `LocalAgentTaskState`，并统一挂进 task 系统。

定义在 [src/tasks/LocalAgentTask/LocalAgentTask.tsx](/Users/niko/claude-code-public/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L116)。

它除了通用 task 字段外，还包含：

- `prompt`
- `progress`
- `messages`
- `pendingMessages`
- `retrieved`
- `retain`
- `diskLoaded`
- `evictAfter`

这说明后台 agent 在这里不是“抽象任务”，而是一个有 UI 生命周期、有消息、有输出的完整实体。

## 关键代码

```ts
export type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  progress?: AgentProgress
  retrieved: boolean
  messages?: Message[]
  pendingMessages: string[]
  retain: boolean
  diskLoaded: boolean
  evictAfter?: number
}
```

## 这些字段分别解决什么问题

### `progress`

让 UI 和主线程知道这个 agent 当前大概做到了哪里。

### `messages`

允许在查看 agent transcript 时立刻显示它的会话内容。

### `pendingMessages`

允许别的 agent 或主线程在它运行过程中注入消息，而不是必须等它完全结束。

### `retrieved`

表示它的结果是否已经被消费过，避免重复读取。

### `retain` / `diskLoaded` / `evictAfter`

这是偏 UI 和生命周期管理的字段：

- `retain`
  表示 UI 正在持有它，暂时不能回收。
- `diskLoaded`
  表示是否已经从磁盘侧链加载过 transcript。
- `evictAfter`
  表示什么时候可以从状态树里回收。

## 完成后怎么通知主线程

后台 agent 完成时，不是简单改个状态，而是会发一条任务通知。

关键逻辑在 [src/tasks/LocalAgentTask/LocalAgentTask.tsx](/Users/niko/claude-code-public/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L197)。

```ts
const message = `<task-notification>
<task_id>${taskId}</task_id>
<output_file>${outputPath}</output_file>
<status>${status}</status>
<summary>${summary}</summary>
</task-notification>`
```

这意味着主线程不需要直接“等待那个 Promise 返回”，而是可以通过统一的 task-notification 机制消费结果。

## 为什么通知里要有 output file

因为长输出不适合直接塞进通知文本里。  
通知的作用是告诉主线程：

- 任务是谁
- 现在什么状态
- 结果在哪

而不是把结果本体全部带上。

## 这里还需要补一个生命周期视角

理解后台 agent 时，必须同时理解它的生命周期：

- task 先 `pending`
- 再 `running`
- 最后进入 `completed / failed / killed`

理解这一点之后，后台 agent 的很多字段就容易看懂了：  
它们不是随便加的，而是在为不同生命周期阶段服务。

## 设计取舍

这种任务化设计的优点：

- 易于追踪
- 易于恢复
- 易于通知
- 易于 UI 展示

代价：

- 状态模型更复杂
- 需要清理和回收逻辑
- 需要区分“运行态”和“已通知可回收态”

## 可复用方案

如果你自己做后台 agent，建议最少提供：

1. `统一 task state`
2. `独立 output file`
3. `进度字段`
4. `幂等通知机制`
5. `回收时机控制`

不要让后台 agent 只是“后台线程”。  
它应该是一个系统里一等公民的任务实体。

## 延伸阅读

继续看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
