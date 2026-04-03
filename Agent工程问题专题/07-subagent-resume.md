# 问题 7：子 agent 如何恢复运行

## 问题定义

子 agent 一旦被暂停、切后台、进程中断或会话恢复，就要回答一个非常难的问题：

“怎样恢复成刚才那个 agent，而不是假装重新开了一个长得很像的新 agent？”

## 给小白的直白解释

恢复子 agent 不是“重新发一遍任务描述”。

因为真实的子 agent 在运行过程中已经积累了很多隐含状态：

- 它读过哪些消息
- 它看过哪些工具结果
- 它工作在哪个 worktree
- 它继承的是哪个 system prompt
- 它是否是 fork agent
- 它当前的工具替换状态是什么

如果这些状态恢复不全，系统虽然表面上“恢复了”，但实际上已经不是同一个执行上下文。

## 为什么这是 agent 工程问题

如果恢复只靠一段 prompt，会有几个典型问题：

- 重复的 tool_use / tool_result
- worktree 错位
- prompt cache 命中失效
- fork agent 和普通子 agent 行为混乱

## 本项目答案

本项目使用 [src/tools/AgentTool/resumeAgent.ts](/Users/niko/claude-code-public/src/tools/AgentTool/resumeAgent.ts#L42) 的 `resumeAgentBackground()` 来恢复子 agent。

恢复步骤大致是：

1. 读取 transcript
2. 读取 metadata
3. 清理不适合恢复的消息
4. 重建 content replacement state
5. 恢复 worktree
6. 判断 agent 类型
7. 重建 system prompt / tool pool
8. 重新注册 async agent
9. 重新进入 `runAgent()`

## 关键代码

```ts
const [transcript, meta] = await Promise.all([
  getAgentTranscript(asAgentId(agentId)),
  readAgentMetadata(asAgentId(agentId)),
])

const resumedMessages = filterWhitespaceOnlyAssistantMessages(
  filterOrphanedThinkingOnlyMessages(
    filterUnresolvedToolUses(transcript.messages),
  ),
)

const resumedReplacementState = reconstructForSubagentResume(
  toolUseContext.contentReplacementState,
  resumedMessages,
  transcript.contentReplacements,
)
```

## 这段代码意味着什么

恢复不是“照单全收 transcript”，而是要先做清理：

- 去掉纯空白 assistant 消息
- 去掉孤立的 thinking-only 消息
- 去掉未解决的 tool use 残留

因为 transcript 是运行日志，不是可以无脑原样重新喂给模型的 API 输入。

## worktree 恢复为什么重要

这部分在 [src/tools/AgentTool/resumeAgent.ts](/Users/niko/claude-code-public/src/tools/AgentTool/resumeAgent.ts#L80)。

系统会检查原来的 `worktreePath` 还在不在：

- 在，就恢复进去
- 不在，就退回父 cwd

这很重要，因为很多子 agent 的真实工作环境并不是主线程 cwd，而是独立工作树。

## fork agent 恢复为什么更麻烦

fork agent 不是普通子 agent。

在 [src/tools/AgentTool/resumeAgent.ts](/Users/niko/claude-code-public/src/tools/AgentTool/resumeAgent.ts#L116) 后，系统会在恢复 fork agent 时尽量重建父 system prompt。

这样做的原因是：

- fork agent 很依赖 prompt cache 共享
- 恢复时如果前缀不一致，缓存特性就会失效

这也是这个项目协作设计里的一个核心点：  
fork 模式不仅是“继承上下文”，还是“尽量保持字节级相同前缀”。

## agent 类型和工具池为什么也要恢复

恢复不只是恢复消息，还要恢复“这个 agent 是谁”。

代码在 [src/tools/AgentTool/resumeAgent.ts](/Users/niko/claude-code-public/src/tools/AgentTool/resumeAgent.ts#L99) 和 [src/tools/AgentTool/resumeAgent.ts](/Users/niko/claude-code-public/src/tools/AgentTool/resumeAgent.ts#L158)。

系统会：

- 判断是不是 fork agent
- 找到原始 agent definition
- 根据 agent 的 permission mode 和 appState 重建可用工具池

否则恢复后的 agent 就可能和原 agent 拥有不同权限或不同工具，这会直接破坏行为一致性。

## 设计取舍

这套恢复逻辑的优点：

- 恢复出来的更像“原 agent 的继续”
- 不容易引入重复工具调用
- 对 fork / worktree / metadata 更稳

代价：

- 恢复逻辑复杂
- transcript 和 metadata 的格式必须稳定
- 需要额外清洗消息

## 可复用方案

如果你设计子 agent 恢复，建议至少恢复以下 6 类信息：

1. transcript
2. metadata
3. content replacement / tool state
4. worktree / cwd
5. agent type / permissions
6. 父 system prompt 环境

一句话总结：

子 agent 恢复的目标不是“重新执行一个差不多的任务”，而是“继续原来那个执行上下文”。

## 延伸阅读

继续看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
