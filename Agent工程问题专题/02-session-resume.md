# 问题 2：会话中断后如何恢复

## 问题定义

agent 在真实世界里一定会被打断：

- 用户按下停止
- 网络断开
- 进程退出
- IDE / CLI 被关闭
- 子 agent 跑到一半挂掉

所以系统必须回答一个问题：中断之后，如何恢复到“足够继续工作”的状态，而不是重新开始。

## 给小白的直白解释

把会话想成一场长时间的施工。

如果你每天收工前不写施工日志，第二天就只能靠记忆猜昨天做到哪了。  
而一个可靠的 agent 系统，必须像工地一样有“施工日志”和“现场记录”。

resume 的本质不是“模型想起来了”，而是“系统把之前的记录重新装回来了”。

## 为什么这是 agent 工程问题

如果恢复依赖内存状态，就会遇到这些问题：

1. 进程一死，全丢。
2. API 还没回时被杀掉，最后一轮用户输入可能没有被保存。
3. 子 agent 的状态和主会话状态容易脱节。
4. 恢复后无法确认“最后一个可靠检查点”在哪。

## 本项目答案

这个项目把 resume 建立在 transcript 持久化上，而不是只依赖 `mutableMessages`。

最关键的一点是：  
在进入 query loop 之前，就先把用户消息写到 transcript。

这样即使出现下面这种情况：

- 用户刚发了一个新请求
- API 还没来得及返回
- 进程就被杀掉

下次恢复时，系统仍然知道“这个请求已经被接收过了”。

关键代码在 [src/QueryEngine.ts](/Users/niko/claude-code-public/src/QueryEngine.ts#L436)。

## 关键代码

```ts
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages)
  if (isBareMode()) {
    void transcriptPromise
  } else {
    await transcriptPromise
    if (
      isEnvTruthy(process.env.CLAUDE_CODE_EAGER_FLUSH) ||
      isEnvTruthy(process.env.CLAUDE_CODE_IS_COWORK)
    ) {
      await flushSessionStorage()
    }
  }
}
```

## 这段代码在做什么

它的行为可以翻译成一句人话：

“用户这条消息一旦被系统接收，就尽快写入可恢复存储，而不是等模型回复完再说。”

这里有两个模式：

1. `bare mode`
   非交互场景，允许 fire-and-forget。
2. `interactive mode`
   等待 transcript 写完，再继续走 query。

这样设计是因为交互式会话更看重 resume 的可靠性。

## 继续恢复靠什么

用户消息被写进去只是第一步。  
后面 assistant、user、compact boundary 也会不断写回 transcript。

关键代码在 [src/QueryEngine.ts](/Users/niko/claude-code-public/src/QueryEngine.ts#L687)：

```ts
if (
  message.type === 'assistant' ||
  message.type === 'user' ||
  (message.type === 'system' && message.subtype === 'compact_boundary')
) {
  messages.push(message)
  if (persistSession) {
    if (message.type === 'assistant') {
      void recordTranscript(messages)
    } else {
      await recordTranscript(messages)
    }
  }
}
```

这说明 transcript 不是只在开头存一次，而是边运行边记录。

## 为什么 compact boundary 也要存

因为 resume 不是简单读一串对话而已。  
会话中途可能已经发生过 compact。

如果 compact boundary 不被持久化，恢复后的系统就不知道：

- 哪些消息已经被摘要替换
- 哪个边界之后才是当前有效历史

这会直接影响上下文重建。

## 设计取舍

这套方案的好处很明显：

- 中断点更细
- 恢复更稳定
- 用户消息不容易丢

代价是：

- 写盘会进入关键路径
- 需要处理并发和 flush
- transcript 本身也会变成需要管理的系统资产

但对一个可 resume 的 agent 系统来说，这个代价通常值得付。

## 工程上应该记住什么

resume 系统至少要保存：

1. 已接收的用户消息
2. 已产生的 assistant 消息
3. 上下文压缩边界
4. 子 agent transcript
5. 和恢复相关的 metadata

## 可复用方案

如果你要设计自己的 resume：

1. 先定义“最小恢复单元”。
2. 让每个单元可单独落盘。
3. 在高风险边界前优先写入。
4. 恢复时优先信任持久化记录，不信任进程内缓存。

一句话总结：

resume 的本质不是“把历史再喂给模型”，而是“从可靠日志恢复一个继续执行所需的工作状态”。

## 延伸阅读

可以继续看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
