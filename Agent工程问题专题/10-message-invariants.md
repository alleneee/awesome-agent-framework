# 问题 10：消息结构和工具调用不变量如何维护

## 问题定义

一个支持工具调用的 agent 系统，不只是管理“文本历史”，还要管理“结构化消息历史”。

一旦消息被裁剪、压缩、恢复、重组，就必须回答：

- `tool_use` 和 `tool_result` 还能对上吗？
- 流式 assistant 的分片还能合并吗？
- 传给模型 API 的消息格式还合法吗？

## 给小白的直白解释

如果只把消息当作普通字符串，你会很自然地想：

“把前面几条消息删掉不就行了？”

但在 agent 系统里，消息经常不是纯文本，而是结构化块：

- text
- thinking
- tool_use
- tool_result

这就像你不能随便删数据库里的某一行外键记录。  
你删掉一条消息，可能会让后面的引用全断掉。

## 为什么这是 agent 工程问题

常见问题包括：

- 保留了 `tool_result`，却没保留对应 `tool_use`
- 流式 assistant 的多个片段被拆开后无法重新合并
- compact 后历史看起来没问题，但一送 API 就报格式错误

## 本项目答案

这个项目在 session memory compact 里，专门有一层“保持 API 不变量”的逻辑。

关键函数在 [src/services/compact/sessionMemoryCompact.ts](/Users/niko/claude-code-public/src/services/compact/sessionMemoryCompact.ts#L232)：

```ts
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[],
  startIndex: number,
): number
```

这个函数的目标不是做语义压缩，而是做结构校正。

## 它到底在保护什么

从代码注释可以看出，它主要保护两类不变量。

### 1. tool_use / tool_result 配对

如果保留范围里有 `tool_result`，那系统必须确保与之匹配的 `tool_use` 也被保留。  
否则传回 API 时就会出现“结果引用了一个不存在的工具调用”。

### 2. 同一个 assistant message 的分片完整性

流式输出时，一个 assistant message 可能被拆成多个片段：

- thinking
- tool_use
- text

它们可能共享同一个 `message.id`，但有不同 `uuid`。  
如果裁剪时只保留后半段，就会破坏后续消息归并。

## 关键代码

```ts
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[],
  startIndex: number,
): number {
  if (startIndex <= 0 || startIndex >= messages.length) {
    return startIndex
  }

  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }

  // 然后回溯需要保留的 tool_use 与同 message.id 片段
}
```

## 这说明什么

上下文裁剪不能只基于：

- “消息数量太多”
- “token 太大”

它还必须基于消息之间的结构关系。

也就是说，context management 除了是 token 工程，还是消息图结构工程。

## 这里还可以再补一个相关点

query loop 在流式中断时，也会做结构补偿：  
如果流式过程中中断，系统会为未完成的 `tool_use` 生成缺失的错误 `tool_result`，以保持消息结构一致。

这和这里的思路是同一类：

- 不要让 API 看到残缺的工具调用链
- 无论是中断、resume、compact，都要维护结构合法性

## 设计取舍

这种结构保护的好处：

- 更少 API 格式错误
- resume 更稳
- compact 后可继续执行

代价：

- 代码更复杂
- 裁剪逻辑不能只按 token 做
- 需要理解消息之间的引用关系

## 可复用方案

如果你的 agent 支持工具调用，建议显式维护至少三条不变量：

1. 每个 `tool_result` 都必须有对应的 `tool_use`
2. 同一 assistant message 的必要分片不能被错误拆断
3. API 看到的消息序列必须满足顺序和角色要求

一句话总结：

在 agent 系统里，消息不是普通文本列表，而是带引用关系的结构化状态。

## 延伸阅读

建议一起看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
