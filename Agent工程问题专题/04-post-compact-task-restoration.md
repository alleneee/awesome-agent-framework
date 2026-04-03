# 问题 4：压缩后如何补回真正的任务状态

## 问题定义

上下文压缩不是终点。  
真正困难的地方在于：压缩之后，系统怎样确保模型还能继续做“刚才那件事”。

换句话说，compact 不是“怎么变短”，而是“变短以后怎么还能接着干活”。

## 给小白的直白解释

可以把 compact 想成写阶段总结。

如果你只写一句：

> “前面聊了很多技术问题。”

那下一轮基本等于重新开工。  
但如果你写的是：

- 当前任务是什么
- 哪些事已经做完
- 哪些文件刚刚看过
- 哪些后台任务还在跑
- 下一步准备做什么

那系统就能继续工作。

所以一个好的 compact，必须保“工作状态”，不是只保“聊天内容”。

## 为什么这是 agent 工程问题

如果 compact 只生成一段摘要文本，常见后果是：

- plan 丢失
- plan mode 丢失
- 最近读过的关键文件丢失
- 已激活的 skill 丢失
- 已启动但未取回结果的后台 agent 丢失

最后模型虽然“还记得大概聊过什么”，但已经不知道下一步该接哪。

## 本项目答案

这个项目在 compact 完成后，不是简单替换成“一个 summary”，而是重建一组结构化 post-compact messages。

核心函数在 [src/services/compact/compact.ts](/Users/niko/claude-code-public/src/services/compact/compact.ts#L330)：

```ts
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,
    ...result.summaryMessages,
    ...(result.messagesToKeep ?? []),
    ...result.attachments,
    ...result.hookResults,
  ]
}
```

注意这里不是只有 `summaryMessages`，还有：

- `messagesToKeep`
- `attachments`
- `hookResults`

也就是说，compact 后的上下文是“摘要 + 保留段 + 工作附件”的组合。

## 压缩后补回了哪些关键状态

在 [src/services/compact/compact.ts](/Users/niko/claude-code-public/src/services/compact/compact.ts#L531) 开始，系统会并行和顺序地补回很多东西。

### 1. 最近读过的文件

通过 `createPostCompactFileAttachments()` 把最近工作集重新挂回去。  
这样模型不用马上重新读一遍关键文件。

### 2. plan 文件

通过 `createPlanAttachmentIfNeeded()` 注入 `plan_file_reference`：

- 路径
- 内容

代码在 [src/services/compact/compact.ts](/Users/niko/claude-code-public/src/services/compact/compact.ts#L1470)。

### 3. plan mode

如果当前还在 plan mode，会重新挂一个 `plan_mode` 附件。  
否则 compact 之后模型可能忘了它当前还处于“先规划，不直接实施”的模式。

代码在 [src/services/compact/compact.ts](/Users/niko/claude-code-public/src/services/compact/compact.ts#L1542)。

### 4. invoked skills

如果当前会话已经调用过 skill，会重新注入 skill 内容，防止 compact 后技能约束消失。

代码在 [src/services/compact/compact.ts](/Users/niko/claude-code-public/src/services/compact/compact.ts#L1494)。

### 5. 异步 agent 状态

如果后台 agent 还在运行，或者已经结束但结果还没被取回，会补一个 `task_status` 附件，提醒主线程“别重复开一个”。

代码在 [src/services/compact/compact.ts](/Users/niko/claude-code-public/src/services/compact/compact.ts#L1568)。

## 关键代码

```ts
const [fileAttachments, asyncAgentAttachments] = await Promise.all([
  createPostCompactFileAttachments(
    preCompactReadFileState,
    context,
    POST_COMPACT_MAX_FILES_TO_RESTORE,
  ),
  createAsyncAgentAttachmentsIfNeeded(context),
])

const planAttachment = createPlanAttachmentIfNeeded(context.agentId)
const planModeAttachment = await createPlanModeAttachmentIfNeeded(context)
const skillAttachment = createSkillAttachmentIfNeeded(context.agentId)
```

## 为什么这比“只做摘要”强

因为它把 compact 分成两部分：

1. `理解历史`
   交给 summary。
2. `继续工作`
   交给附件和保留段。

这是非常重要的工程拆分。  
很多系统的问题就在于，把这两件事混成了一件事。

## 这里还要特别强调一个工程点

后压缩恢复不是无上限的，它本身也有预算设计，例如：

- 最多恢复多少文件
- 每个文件最大 token
- skills 的总预算

这说明这个项目不是无上限地补状态，而是有“恢复预算”的。  
否则压缩完再补太多东西，马上又会逼近阈值。

## 设计取舍

这套设计的优点：

- compact 后任务不容易漂移
- 关键工作集能快速恢复
- 已运行任务不容易重复生成

代价：

- compact 结果结构复杂
- 需要维护附件预算
- 需要区分“摘要应该承载什么”和“附件应该承载什么”

## 可复用方案

如果你设计自己的 compact 恢复层，建议明确区分三类东西：

1. `摘要`
   让模型知道前面发生了什么。
2. `保留段`
   让模型看到仍然需要原样保留的最近消息。
3. `恢复附件`
   让模型拿回当前计划、工作集、后台任务状态。

一句话总结：

一个好的 compact 不是“把上下文缩小”，而是“把上下文重新组织成更适合继续工作的形态”。

## 延伸阅读

继续看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
