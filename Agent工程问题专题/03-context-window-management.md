# 问题 3：窗口快爆了时如何处理上下文

## 问题定义

长对话、多轮工具调用、大体积文件读取、Shell 输出和网页内容都会让上下文窗口快速膨胀。  
真正的问题不是“会不会爆”，而是：

- 什么时候开始处理？
- 先处理什么？
- 哪些东西可以删？
- 哪些东西绝对不能丢？

## 给小白的直白解释

你可以把 context window 想成一个很贵、很有限的工作台。

- 真正的任务目标是工作台中央的零件。
- 工具输出、大段日志、旧文件内容是堆在边上的包装纸和废料。

一个好的系统，不会等工作台完全塞满才去清理。  
它会先把废料挪走，再压缩旧记录，最后才做整体总结。

## 为什么这是 agent 工程问题

如果上下文管理做得差，会出现三类坏结果：

1. `真实任务被噪声挤掉`
   工具输出比用户意图更占空间。
2. `压缩太晚`
   直接触发 `prompt_too_long`。
3. `压缩太粗暴`
   历史虽然变短了，但任务已经漂移了。

## 本项目答案

Claude Code 采用的是“多层渐进式压缩”，不是一上来就做全局摘要。

主入口在 [src/query.ts](/Users/niko/claude-code-public/src/query.ts#L365)。

处理顺序如下：

1. `Tool Result Budget`
   超大的工具输出先裁掉或替换。
2. `Snip`
   删除一部分中间消息。
3. `Microcompact`
   清掉旧工具结果的内容，但保留工具调用结构。
4. `Context Collapse`
   把旧上下文提交到 collapse 存储。
5. `Autocompact`
   让模型生成正式摘要，替换历史。

这个顺序很关键。  
它体现了一个原则：先删噪声，再压语义。

## 关键代码

```ts
messagesForQuery = await applyToolResultBudget(...)

if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
}

const microcompactResult = await deps.microcompact(...)
messagesForQuery = microcompactResult.messages

if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
  messagesForQuery = collapseResult.messages
}

const { compactionResult } = await deps.autocompact(...)
```

## 自动压缩何时触发

自动压缩的阈值计算在 [src/services/compact/autoCompact.ts](/Users/niko/claude-code-public/src/services/compact/autoCompact.ts#L67)。

核心思路：

1. 先拿到模型的 context window。
2. 预留一块 token 给“压缩摘要的输出”。
3. 再预留一块 buffer，避免到极限才压。

代码大意如下：

```ts
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000

export function getEffectiveContextWindowSize(model: string): number {
  return contextWindow - reservedTokensForSummary
}

export function getAutoCompactThreshold(model: string): number {
  return getEffectiveContextWindowSize(model) - AUTOCOMPACT_BUFFER_TOKENS
}
```

这意味着系统不是等到“已经装不下”才压，而是提前留出缓冲区。

## 为什么要分层，而不是直接 summarize

因为不同信息的价值不同：

- 大工具输出通常价值低、体积大。
- 最近工作上下文通常价值高。
- 用户目标和当前任务最重要。

如果一上来就把所有历史做成摘要，会有两个问题：

1. 成本高
2. 摘要质量太依赖模型，容易丢细节

分层压缩的好处是：

- 能本地处理的先本地处理
- 能不调用模型就不调用模型
- 只有真正必要时才用 LLM 做语义压缩

## 这个项目里还需要特别注意的两个视角

为了让这个话题真正能构成学习材料，还要额外看到两个层面：

1. `阈值工程`
   不是只有一个“爆了就压”的开关，而是 warning threshold、auto compact threshold、buffer tokens 的组合。
2. `恢复工程`
   压缩完并不是结束，还要补回关键文件、skill、plan 等工作集。

这也是为什么“上下文管理”不能只看 `compact.ts`，而要看完整的 query loop。

## 设计取舍

这种多层策略的代价是复杂：

- 状态多
- 触发条件多
- 需要处理不同压缩层之间的配合

但它换来的收益也很直接：

- 更少打到真正的 `prompt_too_long`
- 更少无意义摘要
- 更少把当前任务压坏

## 可复用方案

如果你做自己的 agent 系统，可以直接采用这个优先级：

1. `先处理大块工具输出`
2. `再处理旧工具结果`
3. `再处理远历史`
4. `最后才做语义总结`

同时要加两个保护：

- 提前阈值，而不是顶到边缘
- 压缩失败熔断，避免无限重试

## 延伸阅读

建议一起看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
