# Agent Engineering Problems

这组文档是按“问题驱动”整理的学习材料，目标读者是刚开始接触 agent 系统设计的人。

这组文档现在包含两种阅读方式：

- `问题驱动文档`
  按工程问题拆分，适合学习和设计评审。
- `系统总览文档`
  把 Claude Code 的系统级说明合并成一篇，适合建立全局视角。

## 阅读路线

如果你是小白，建议按下面顺序读：

1. [长期记忆应该放在哪里](./01-long-term-memory.md)
2. [会话中断后如何恢复](./02-session-resume.md)
3. [窗口快爆了时如何处理上下文](./03-context-window-management.md)
4. [压缩后如何补回真正的任务状态](./04-post-compact-task-restoration.md)
5. [计划如何持久化并参与恢复](./05-plan-persistence.md)
6. [后台 agent 状态如何管理](./06-background-agent-state.md)
7. [子 agent 如何恢复运行](./07-subagent-resume.md)
8. [多 agent 之间如何可靠通信](./08-multi-agent-messaging.md)
9. [异常、循环过长和输出截断如何收口](./09-failure-recovery-and-loop-bounds.md)
10. [消息结构和工具调用不变量如何维护](./10-message-invariants.md)
11. [Claude Code 系统总览](./11-claude-code-system-overview.md)

## 这组文档回答什么

一个能长期工作的 agent 系统，通常绕不开下面 4 类问题：

1. `记忆问题`
   agent 怎么记住规则、目标、计划、历史，而不是每轮都重新“失忆”。
2. `执行问题`
   agent 怎么跑起来、怎么继续跑、怎么停下来、怎么切后台。
3. `协作问题`
   多个 agent 怎么通信、怎么避免重复工作、怎么回收结果。
4. `可靠性问题`
   上下文爆了、输出被截断、API 报错、工具调用失配时怎么办。

## 两种文档怎么配合

如果你先想建立“整体系统观”，建议先读 [11-claude-code-system-overview.md](./11-claude-code-system-overview.md)。

如果你更关心某一个工程难题，比如：

- 记忆为什么会丢
- compact 后为什么任务会漂移
- 子 agent 为什么恢复不稳

那就先读对应的编号文档，再回头看总览文档补齐全局关系。

## 怎么使用这些文档

如果你的目标是“学会设计 agent 系统”，建议每篇都按这个顺序读：

1. `问题定义`
   先理解这个问题为什么存在。
2. `给小白的直白解释`
   用非源码语言建立直觉。
3. `本项目答案`
   看 Claude Code 选了什么实现路径。
4. `关键代码`
   结合源码确认它不是空谈。
5. `设计取舍`
   理解为什么不用更简单的做法。
6. `可复用方案`
   把做法迁移到你自己的 agent 框架里。

## 一个重要提醒

这些文档不是在说“Claude Code 的做法就是唯一正确答案”。  
它们更像是在说明：

- 这个问题真实存在；
- 这个项目是怎么处理的；
- 这种处理为什么有效；
- 它的代价是什么；
- 你做自己的 agent 时可以怎么借鉴。
