# 问题 9：异常、循环过长和输出截断如何收口

## 问题定义

一个 agent 系统只要跑得足够久，就一定会遇到这些问题：

- API 速率限制
- 模型过载
- prompt 太长
- 输出被截断
- 工具调用中断
- 会话在循环里越跑越偏

真正的问题不是“会不会出错”，而是“出错后系统怎么优雅收口”。

## 给小白的直白解释

很多人做 agent 时把注意力都放在“让它完成任务”。  
但工程上更重要的问题往往是：

“它失败时会不会变成一个疯狂重试、不断烧 token、还越来越偏的系统？”

一个成熟系统必须有：

- 重试策略
- 恢复策略
- 停止策略
- 熔断策略

## 为什么这是 agent 工程问题

没有这些机制，agent 常见的坏行为是：

- 相同错误无限重试
- 被截断后不断重复前文
- `prompt_too_long` 后还继续注入更多错误消息
- 一轮又一轮继续，但实际已经没有收益

## 本项目答案

这个项目把“错误恢复”和“循环边界”拆成了几层。

### 一类：API 侧重试与分类恢复

核心思想是：

- 不同错误，不同策略
- 不是所有错误都等价

比如：

- `429/529`
  走指数退避或模型回退
- `401/403`
  尝试刷新认证
- `prompt_too_long`
  走压缩恢复

### 二类：query loop 内部恢复

在 [src/query.ts](/Users/niko/claude-code-public/src/query.ts#L1168) 一带，系统会对：

- `prompt_too_long`
- `max_output_tokens`
- stop hooks
- token budget
- maxTurns

分别处理。

## max_output_tokens 的恢复特别值得学

这是我之前讲得太简略的地方。

在 [src/query.ts](/Users/niko/claude-code-public/src/query.ts#L1223)，如果模型输出被截断，系统不会直接结束，而是构造一条 meta 消息：

```ts
const recoveryMessage = createUserMessage({
  content:
    `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
    `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,
})
```

这条消息非常工程化，它强制模型：

- 不要道歉
- 不要 recap
- 直接续写
- 把剩余工作切小

这类恢复提示词非常实用，因为它直接减少了“被截断后模型先废话一轮”的问题。

## maxTurns 和 token budget 是两种不同的边界

### `maxTurns`

硬边界。  
在 [src/query.ts](/Users/niko/claude-code-public/src/query.ts#L1704)，超过轮次就停。

### `token budget`

软边界。  
在 [src/query/tokenBudget.ts](/Users/niko/claude-code-public/src/query/tokenBudget.ts#L40)，系统会判断：

- 当前输出占预算多少
- 是否已经出现收益递减

如果还值得继续，就注入一条 “继续” 的 meta nudge。  
如果收益递减，就及时停下。

这很像：

- `maxTurns` 是保险丝
- `token budget` 是节流阀

## prompt_too_long 的恢复为什么不能乱来

在 [src/query.ts](/Users/niko/claude-code-public/src/query.ts#L1168)，系统对 `prompt_too_long` 很谨慎。

特别值得注意的是：

- 如果恢复失败，不继续跑 stop hooks
- 否则可能进入“错误 -> hook -> 更多上下文 -> 更长错误”的死循环

这个判断非常重要。  
说明系统不是机械地“所有失败都走同一套后处理”，而是知道哪些路径会形成放大回路。

## 还需要补一个宏观视角

从系统角度看，恢复矩阵至少包括：

- 429/529 的处理
- 指数退避
- 流式失败回退到非流式
- fallback model
- 持久重试模式

另外，还要把 query loop 内的状态机视角一起看，才能知道这些恢复钩子到底挂在哪个阶段。

把这两篇和当前源码结合起来看，可以得到一个完整认知：

- 一部分错误在 API 层收口
- 一部分错误在 query loop 层收口
- 一部分错误通过 compact / continuation / stop 机制收口

## 设计取舍

这种多层恢复的优点：

- 更稳
- 更少无意义重试
- 更少 token 浪费
- 更少死循环

代价：

- 逻辑分散
- 需要明确错误分类
- 状态机会更复杂

## 可复用方案

如果你设计自己的 agent，建议至少加上：

1. `错误分类表`
   哪些错误可重试，哪些不可。
2. `截断续写提示`
   明确要求“直接继续，不要 recap”。
3. `循环硬边界`
   最大轮次。
4. `收益递减停止`
   预算或质量阈值。
5. `熔断器`
   避免同一失败路径无限重复。

一句话总结：

好的 agent 系统不是“永不出错”，而是“知道什么时候该重试，什么时候该压缩，什么时候该停”。

## 延伸阅读

建议一起看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
