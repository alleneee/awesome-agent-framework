# 问题 1：长期记忆应该放在哪里

## 问题定义

一个 agent 在长时间工作时，一定会遇到“记忆”问题。这里的“长期记忆”不是指模型内部参数，而是指这个会话、这个用户、这个项目反复需要用到的稳定信息。

典型例子：

- 用户总是希望你先给方案再改代码。
- 某个项目禁止直接改数据库 schema。
- 某个仓库必须先跑测试再提交。
- 某个系统依赖外部服务 A/B/C 的固定约束。

这些信息如果只存在聊天历史里，就会随着上下文压缩、恢复、切换 agent 而变得不稳定。

## 给小白的直白解释

可以把 agent 想成一个很聪明、但短期记忆不可靠的实习生。

- 你在今天上午口头说过一次的事情，他下午可能就忘了。
- 你写进公司 wiki 的事情，他每次开工前都能再查一遍。

所以长期记忆最重要的原则不是“让模型记住”，而是“让模型每次都能重新读到”。

## 为什么这是 agent 工程问题

如果长期记忆只放在对话消息里，会有四个问题：

1. `上下文窗口有限`
   聊久了，旧消息会被压缩或丢弃。
2. `子 agent 难继承`
   一个新开的 agent 不一定带着完整历史。
3. `恢复不稳定`
   进程重启后，只有显式持久化的东西才可靠。
4. `规则容易漂移`
   如果规则只是“模型之前好像见过”，那就会时有时无。

## 本项目答案

这个项目把长期记忆做成“外部文件 + 每轮注入”的模式。

核心入口在 [src/context.ts](/Users/niko/claude-code-public/src/context.ts#L155) 的 `getUserContext()`：

1. 判断是否应该禁用 `CLAUDE.md`。
2. 读取 memory files。
3. 过滤掉不该注入的记忆文件。
4. 组装成 `claudeMd` 文本。
5. 在每轮 query 前作为 `userContext` 注入。

这意味着：

- 记忆不是“只在第一轮出现一次”。
- 记忆也不是“全靠 resume 时重建”。
- 而是“每轮都重新放进 prompt 的稳定部分”。

## 关键代码

```ts
export const getUserContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    const shouldDisableClaudeMd =
      isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
      (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

    const claudeMd = shouldDisableClaudeMd
      ? null
      : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

    setCachedClaudeMdContent(claudeMd || null)

    return {
      ...(claudeMd && { claudeMd }),
      currentDate: `Today's date is ${getLocalISODate()}.`,
    }
  },
)
```

## 这段代码在做什么

逐步看：

1. `getMemoryFiles()`
   读取项目相关的记忆文件来源。
2. `filterInjectedMemoryFiles(...)`
   过滤不应该直接放进 prompt 的文件。
3. `getClaudeMds(...)`
   把记忆文件整理成系统可消费的文本。
4. `setCachedClaudeMdContent(...)`
   缓存内容给其他模块使用。
5. `return { claudeMd, currentDate }`
   最终把这些内容作为 `userContext` 注入主循环。

这其实是在做一件很工程化的事情：

“把长期记忆从一次性上下文，变成可重复构建的上下文。”

## 这个设计为什么比“直接记在聊天里”更好

因为它解决了两个不同层次的问题：

1. `语义层`
   模型每轮都能重新看到这些规则。
2. `系统层`
   即使前文被 compact，规则仍然存在。

如果只把规则写在聊天里，那么 compact 之后它很可能变成一句模糊摘要。  
但如果写进 `CLAUDE.md` / memory files，它会在下一轮再次被完整注入。

## 设计取舍

这种方案也不是没有代价：

- 每轮都要读一次文件或命中缓存。
- prompt 前缀会变长。
- 如果记忆文件写得很差，会污染每一轮对话。

所以长期记忆不应该是“什么都往里塞”，而应该是高价值、低频变化的信息。

## 适合放进长期记忆的内容

- 用户偏好
- 工作方式要求
- 项目外部约束
- 组织级规则
- 无法从代码仓库直接推导的信息

## 不适合放进长期记忆的内容

- 当前 diff
- 最近一次工具输出
- 某个文件今天的具体实现
- git 状态
- 代码结构这类可动态读取的信息

这些东西应该在运行时获取，而不是作为长期记忆缓存。

## 可复用方案

如果你在做自己的 agent 系统，建议最少拆成三类记忆：

1. `user memory`
   用户偏好与工作风格。
2. `project memory`
   项目的外部上下文和限制。
3. `reference memory`
   外部系统地址、文档入口、固定指针。

并遵守一个原则：

长期记忆一定要“可重建、可注入、可审查”，不要做成只能靠模型自己回忆的隐式状态。

## 延伸阅读

如果你想把这个问题放回整体系统里理解，可以继续看：

- [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)
