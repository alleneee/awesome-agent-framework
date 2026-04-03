# 问题 5：计划如何持久化并参与恢复

## 问题定义

复杂 agent 不能每一轮都靠模型临时“想出计划”。  
真正可靠的系统，需要一个可持久化、可恢复、可附加到上下文里的计划实体。

## 给小白的直白解释

如果 agent 没有 plan file，那么它的计划往往只存在于三种脆弱地方：

- 某一轮 assistant 输出
- 某一小段对话历史
- 模型自己“记得”

这三种地方都不可靠。  
而 plan file 的作用，就是把“当前计划”从易失上下文中拿出来，变成外部状态。

## 为什么这是 agent 工程问题

没有外部 plan，会出现这些问题：

- compact 后计划变模糊
- resume 后计划漂移
- 子 agent 无法可靠继承主计划
- plan mode 很难稳定维持

## 本项目答案

本项目把 plan 设计成会话级文件。

关键路径：

- 通过 session slug 确定 plan 文件名
- 主会话和子 agent 可以有各自 plan 文件
- resume 时尝试恢复 plan 文件
- compact 时把 plan 重新注入上下文

关键代码位置：

- [src/utils/plans.ts](/Users/niko/claude-code-public/src/utils/plans.ts#L119)
- [src/utils/plans.ts](/Users/niko/claude-code-public/src/utils/plans.ts#L135)
- [src/utils/plans.ts](/Users/niko/claude-code-public/src/utils/plans.ts#L164)
- [src/services/compact/compact.ts](/Users/niko/claude-code-public/src/services/compact/compact.ts#L1470)

## 关键代码

```ts
export function getPlanFilePath(agentId?: AgentId): string {
  const planSlug = getPlanSlug(getSessionId())
  if (!agentId) {
    return join(getPlansDirectory(), `${planSlug}.md`)
  }
  return join(getPlansDirectory(), `${planSlug}-agent-${agentId}.md`)
}
```

```ts
export function getPlan(agentId?: AgentId): string | null {
  const filePath = getPlanFilePath(agentId)
  try {
    return getFsImplementation().readFileSync(filePath, { encoding: 'utf-8' })
  } catch (error) {
    if (isENOENT(error)) return null
    return null
  }
}
```

## resume 时 plan 怎么恢复

这个点很重要，也是我之前那版回答里讲得不够细的地方。

在 [src/utils/plans.ts](/Users/niko/claude-code-public/src/utils/plans.ts#L164) 的 `copyPlanForResume()` 中，系统会：

1. 从 log 提取 plan slug
2. 尝试找到现有 plan 文件
3. 如果 plan 文件不存在
4. 优先从 file snapshot 恢复
5. 不行再从消息历史恢复
6. 然后重写回 plan 文件

这说明 plan 不只是“当前有个文件”，而是参与了真正的恢复流程。

## compact 时 plan 怎么重新回到上下文

在 [src/services/compact/compact.ts](/Users/niko/claude-code-public/src/services/compact/compact.ts#L1470)，会调用：

```ts
export function createPlanAttachmentIfNeeded(agentId?: AgentId): AttachmentMessage | null {
  const planContent = getPlan(agentId)
  if (!planContent) {
    return null
  }
  const planFilePath = getPlanFilePath(agentId)
  return createAttachmentMessage({
    type: 'plan_file_reference',
    planFilePath,
    planContent,
  })
}
```

也就是说：

- compact 把旧历史压掉了
- 但 plan 作为外部文件再次被挂回上下文

这就是“任务不漂移”的关键做法之一。

## 为什么 plan 要独立成文件，而不是普通消息

因为 plan 和普通聊天消息的角色不同：

- 聊天消息描述“发生了什么”
- plan 文件描述“接下来准备怎么做”

这两个东西的生命周期不同。  
plan 应该更稳定、更容易恢复、更容易单独更新。

## 设计取舍

plan file 的好处：

- 可持久化
- 可恢复
- 可压缩后再注入
- 可被主会话和子 agent 分开管理

代价：

- 需要额外的文件管理逻辑
- 要处理 slug、fork、resume、冲突
- 要决定 plan 与普通消息的同步边界

## 可复用方案

如果你做自己的 agent 框架，plan 最少应该支持：

1. 稳定路径
2. 主会话/子 agent 隔离
3. compact 后再注入
4. resume 时恢复
5. 可由 UI 或工具单独读取

一句话总结：

plan 不是“聊天的一部分”，而是“任务状态的一部分”。

## 延伸阅读

可以继续看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
