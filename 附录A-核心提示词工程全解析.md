# 附录A：核心提示词工程全解析

Claude Code 的提示词系统横跨系统主提示词（`prompts.ts`）、工具提示词（每个工具的 `prompt.ts`）、压缩提示词（`compact/prompt.ts`）、记忆检索提示词（`findRelevantMemories.ts`）和 Agent 子代指令（`AgentTool/prompt.ts`）五大类。本附录按功能维度逐一提炼。

---

## A.1 系统主提示词的七层架构

Claude Code 的系统提示词并非一段简单的文本，而是一个精心分层的工程结构。`getSystemPrompt()` 函数将其组织为**静态区段**和**动态区段**两大部分，中间以缓存边界标记分隔。这种架构的核心目的是：静态部分可以被 API 层缓存（节省 token 开销），动态部分则随会话状态实时变化。

### 静态区段：七层不变的行为骨架

静态区段由七个函数依次拼接而成，顺序固定，内容在同一版本内不变。

#### 第一层：身份定义层（getSimpleIntroSection）

这是整个提示词的开篇，回答最基本的问题——"你是谁"。

```typescript
// 核心定义：Claude Code 是一个交互式 agent
// 注入网络安全风险指令 CYBER_RISK_INSTRUCTION
// 限制 URL 生成行为
// 如果用户配置了 outputStyle，角色描述切换为"按照输出风格回应"
```

身份定义必须出现在最前面。大语言模型对提示词头部的权重最高，将角色锚定放在第一位，确保后续所有行为指令都在正确的角色框架下被解读。`CYBER_RISK_INSTRUCTION` 的注入也遵循同样逻辑——安全约束越早出现，越不容易被后续内容覆盖。

#### 第二层：系统规则层（getSimpleSystemSection）

定义 Claude Code 运行环境的基本规则，共六条：

1. **输出可见性规则**：文本输出直接呈现给用户，不存在隐藏的"内部思考"通道
2. **工具权限模式**：声明工具调用需要遵循的权限模型
3. **system-reminder 标签说明**：告知模型这些标签的含义和来源
4. **prompt injection 防御提示**：明确警告模型注意注入攻击
5. **hooks 系统说明**：解释用户自定义钩子的运行机制
6. **上下文自动压缩说明**：告知模型对话可能被压缩过，当前看到的不一定是完整历史

这一层本质上是在建立"世界观"——让模型理解自己运行在什么样的环境中、受到什么样的约束。prompt injection 防御被放在第二层而非末尾，体现了安全指令前置的设计原则。

#### 第三层：任务执行层（getSimpleDoingTasksSection）

这是内容最丰富的一层，定义了 Claude Code 作为软件工程助手的核心行为准则。

**"先读后改"原则**是本层的基石：在修改任何文件之前，必须先读取并理解相关代码，不要对未检查的代码进行推测。

**反过度工程三原则**：
- 不添加用户未要求的功能
- 不添加不必要的错误处理
- 不创建只用一次的抽象

这三条规则直接针对大语言模型的常见倾向——模型天然倾向于生成"完善"的代码，但在工程实践中，这种完善往往是有害的过度工程。

**ant 内部版本的额外规则**更为严格：

- 默认不写注释——除非 WHY 不明显（基于"代码应当自描述"的工程信念）
- 完成前必须验证
- 发现用户理解有误时必须指出
- 如实报告结果，不美化

#### 第四层：行动安全层（getActionsSection）

建立了"三思而后行"的决策框架，按风险等级将操作分为三类：

| 风险等级 | 操作示例 | 要求 |
|---------|---------|------|
| 破坏性操作 | 删除文件/分支、DROP TABLE | 必须确认 |
| 难逆转操作 | force push、reset --hard | 高度谨慎 |
| 影响他人 | push 代码、创建 PR/Issue、发消息 | 三思而行 |

核心哲学：**"measure twice, cut once"**（量两次，切一次）。

#### 第五层：工具使用层（getUsingYourToolsSection）

强制模型使用专用工具而非通用 Bash 命令：

- 用 Read 工具代替 cat/head/tail
- 用 Edit 工具代替 sed/awk
- 用 Grep 工具代替 grep/rg
- 用 Glob 工具代替 find/ls

这不是风格偏好，而是安全和可控性的要求。专用工具经过权限检查和沙箱隔离，而 Bash 命令的行为难以预测和约束。

本层还包含**并行工具调用的依赖判断规则**：当多个工具调用之间没有数据依赖时，应当并行发出；存在依赖时，必须等待前序调用完成。

#### 第六层：语气风格层（getSimpleToneAndStyleSection）

定义输出的形式规范：

- 不使用 emoji
- 保持简洁
- 代码引用必须带文件路径和行号（方便用户直接跳转）
- GitHub 引用使用 `owner/repo#123` 格式
- 工具调用前不使用冒号（避免渲染时多余字符）

#### 第七层：输出效率层（getOutputEfficiencySection）

这一层在外部版本和 ant 内部版本之间存在**根本性差异**：

**外部版本**采用极简风格："直奔主题，一句话能说清楚的不用三句话。"

**ant 内部版本**则完全相反，采用**倒金字塔写作法**：使用完整语法句子、写流畅的散文而非碎片化要点、假设用户已经离开需要从头理解上下文。

这种差异反映了两种使用场景：外部用户在终端前实时交互，简洁即效率；内部用户可能在异步审查 agent 输出，完整性比简洁性更重要。

### 缓存边界标记

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

这个字符串是整个架构的关键枢纽。它将提示词物理分割为两部分：边界之上的静态内容可以被缓存逻辑复用，边界之下的动态内容每次都重新生成。

源码中的 WARNING 注释明确指出：**不可移除或重排序此边界**，否则 `src/utils/api.ts` 和 `src/services/api/claude.ts` 中的缓存逻辑将崩溃。

### 动态区段：随会话状态变化的部分

动态区段通过 `systemPromptSection()` 注册器管理，支持按名称去重和缓存：

| 注册名 | 功能 | 特殊性 |
|--------|------|--------|
| `session_guidance` | Agent 工具使用指引、Fork/Explore 模式、Skill 发现 | 包含 ant-only 的 Verification Agent A/B 测试 |
| `memory` | 记忆系统提示词 | 通过 `loadMemoryPrompt()` 加载 |
| `env_info_simple` | CWD、Git、平台、Shell、OS、模型名、知识截止日期 | 每次会话不同 |
| `language` | 语言偏好 | 如 "Always respond in chinese" |
| `output_style` | 自定义输出风格 | 用户配置驱动 |
| `mcp_instructions` | MCP 服务器指令 | 使用 `DANGEROUS_uncachedSystemPromptSection`，因 MCP 在 turn 间可能连接/断开 |
| `numeric_length_anchors` | 工具调用间文本不超过25词，最终回复不超过100词 | ant-only |
| `token_budget` | token 预算目标指令 | feature flag 控制 |
| `brief` | Brief 工具说明 | KAIROS-only |

### 两种特殊模式

**CLAUDE_CODE_SIMPLE 极简模式**将整个七层架构压缩为一行：

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
  return [
    `You are Claude Code, Anthropic's official CLI for Claude.\n\nCWD: ${getCwd()}\nDate: ${getSessionStartDate()}`,
  ]
}
```

**PROACTIVE/KAIROS 自主模式**则进行角色切换：

```typescript
return [
  `\nYou are an autonomous agent. Use the available tools to do useful work.\n\n${CYBER_RISK_INSTRUCTION}`,
  getSystemRemindersSection(),
  await loadMemoryPrompt(),
  envInfo,
]
```

角色从"交互式助手"变为"自主 agent"。注意即使在自主模式下，`CYBER_RISK_INSTRUCTION` 依然被保留——安全约束在任何模式下都不可省略。

### 架构总结

七层架构遵循一个清晰的逻辑递进：

```
你是谁 -> 你在什么环境 -> 你该做什么 -> 做之前想什么 -> 用什么做 -> 怎么说 -> 说多少
```

每一层都在前一层的基础上收窄行为空间，最终将一个通用大语言模型约束为一个安全、高效、风格一致的软件工程 agent。

---

## A.2 工具提示词的设计模式

每个工具目录下的 `prompt.ts` 是专门写给 LLM 看的使用手册，随系统提示词注入。以下逐一分析核心工具的提示词设计。

### A.2.1 BashTool（约370行，最复杂的单工具提示词）

源码位于 `src/tools/BashTool/prompt.ts`。这是整个工具系统中提示词最长、结构最复杂的单一文件，承担了命令执行、版本控制、沙箱安全三重职责。

#### 工具偏好指令

BashTool 的第一道防线是"工具替代"指令，强制 LLM 在能使用专用工具时不要退化为 Bash 命令：

```typescript
const toolPreferenceItems = [
  `File search: Use ${GLOB_TOOL_NAME} (NOT find or ls)`,
  `Content search: Use ${GREP_TOOL_NAME} (NOT grep or rg)`,
  `Read files: Use ${FILE_READ_TOOL_NAME} (NOT cat/head/tail)`,
  `Edit files: Use ${FILE_EDIT_TOOL_NAME} (NOT sed/awk)`,
  `Write files: Use ${FILE_WRITE_TOOL_NAME} (NOT echo >/cat <<EOF)`,
  'Communication: Output text directly (NOT echo/printf)',
]
```

这是一个"负面清单"设计模式——用括号中的 `NOT` 明确列出被禁止的替代方案。LLM 在生成 Bash 命令时会先"想到" `find`，负面清单能在那个瞬间拦截它。

ant 内部版本由于使用嵌入式 bfs/ugrep 替代了 Glob/Grep 工具，会跳过 find/grep 相关指引。

#### Git Safety Protocol

每条规则都对应一个真实的生产事故场景：

```
- NEVER update the git config
- NEVER run destructive git commands (push --force, reset --hard,
  checkout ., restore ., clean -f, branch -D) unless explicitly requested
- NEVER skip hooks (--no-verify, --no-gpg-sign, etc) unless explicitly requested
- NEVER run force push to main/master, warn the user if they request it
- CRITICAL: Always create NEW commits rather than amending
- When staging files, prefer adding specific files by name rather than "git add -A"
- NEVER commit changes unless the user explicitly asks you to
```

`CRITICAL` 标记的 amend 规则背景：当 pre-commit hook 失败时，commit 实际上并未发生。如果此时 LLM 修复问题后使用 `--amend`，它修改的是上一个已成功的 commit——静默覆盖用户之前的工作。

"prefer adding specific files by name" 防御 `.env`、`credentials.json` 等敏感文件被 `git add -A` 意外提交。"NEVER commit unless explicitly asked" 控制了 agent 自主权边界：写代码可以主动，但提交代码必须被动。

#### 多命令执行规范

| 场景 | 方案 | 示例 |
|------|------|------|
| 独立命令 | 并行多个 Bash tool call | `git status` 和 `git diff` 分开调用 |
| 依赖命令 | 用 `&&` 串联 | `mkdir -p dir && cd dir && ls` |
| 容错命令 | 用 `;` 分隔 | `cmd1 ; cmd2`（cmd1 失败仍执行 cmd2） |
| **禁止** | 用换行符分隔命令 | — |

禁止换行符分隔是因为 LLM 倾向于将多条命令写成多行"脚本"格式，这既不利于日志追踪，也容易在命令中间插入意外的空行导致语法错误。

#### sleep 命令限制

核心意图是消除 LLM 的"人类习惯模拟"。LLM 生成 shell 脚本时会模仿人类编写的部署脚本风格（`sleep 30 && curl ...`），但在 Claude Code 的事件驱动架构中，`run_in_background` 机制提供了原生的异步等待。在 KAIROS/MONITOR 模式下，`sleep N`（N >= 2）被运行时直接阻断。

#### 沙箱系统（getSimpleSandboxSection）

1. **文件系统**：`allowOnly`（白名单）或 `denyOnly`（黑名单）模式控制读写范围
2. **网络**：主机级白名单/黑名单
3. **临时文件**：必须使用 `$TMPDIR` 而非硬编码 `/tmp`
4. **`dangerouslyDisableSandbox`**：仅在确认看到 "Operation not permitted" 等沙箱失败证据后才可启用

沙箱提示词渲染前有一个优化：`SandboxManager` 配置中可能有重复路径，用 `Set` 去重可节省约 150-200 tokens/request。

#### Commit 和 PR 完整指令

占据 BashTool 提示词约一半篇幅。

**Commit 四步流程：**

1. 信息收集（并行）：`git status` + `git diff` + `git log`
2. 分析与拟稿：总结变更性质，拟定 commit message，检查敏感文件
3. 执行（串联）：`git add <specific files>` + `git commit` + `git status`
4. 失败处理：pre-commit hook 失败 -> 修复问题 -> 创建新 commit（不要 amend）

**PR 创建三步流程：**

1. 信息收集（并行）：`git status` + `git diff` + `git log` + `git diff [base-branch]...HEAD`
2. 分析：查看全部 commit（不仅是最新的），拟定标题和 body
3. 执行（并行）：创建分支 + push + `gh pr create`

两套流程都使用 HEREDOC 格式模板（单引号 `'EOF'` 阻止变量展开）。ant 内部版本将整段替换为指向 `/commit` 和 `/commit-push-pr` 技能的简短引用。

### A.2.2 FileEditTool（约28行，最精练的提示词）

与 BashTool 形成鲜明对比，FileEditTool 的提示词极度精练：

```typescript
function getPreReadInstruction(): string {
  return '\n- You must use your `Read` tool at least once in the conversation '
    + 'before editing. This tool will error if you attempt an edit without '
    + 'reading the file.'
}
```

核心规则：

- **先读后改铁律**：未读过的文件直接编辑会报错（提示词约束 + 运行时校验双重保障）
- **精确缩进匹配**：`old_string` 必须与文件内容完全一致，包括空格和 tab
- **唯一性要求**：`old_string` 在文件中不唯一则失败
- **优先编辑而非新建**：`ALWAYS prefer editing existing files. NEVER write new files unless explicitly required`
- ant 内部额外规则：`Use the smallest old_string that's clearly unique -- usually 2-4 adjacent lines`

### A.2.3 FileReadTool

```typescript
export function renderPromptTemplate(lineFormat, maxSizeInstruction, offsetInstruction): string {
  return `Reads a file from the local filesystem. You can access any file directly...
- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to ${MAX_LINES_TO_READ} lines starting from the beginning
- This tool allows Claude Code to read images (PNG, JPG, etc)
- This tool can read PDF files (.pdf). For large PDFs (>10 pages),
  you MUST provide the pages parameter
- This tool can read Jupyter notebooks (.ipynb)
- This tool can only read files, not directories`
}
```

`MAX_LINES_TO_READ = 2000`。文件未变化时返回 `FILE_UNCHANGED_STUB` 避免重复传输。提示词模板使用参数化渲染，支持不同运行环境动态调整。

### A.2.4 FileWriteTool

```typescript
return `Writes a file to the local filesystem.
- This tool will overwrite the existing file.
- If this is an existing file, you MUST use the Read tool first.
- Prefer the Edit tool for modifying existing files -- it only sends the diff.
- NEVER create documentation files (*.md) or README files unless explicitly requested.`
```

主动引导使用 Edit 而非 Write（Write 传输完整文件，Edit 只传差异，token 消耗差异巨大）。文档文件禁令约束 LLM"过度热心"地在完成任务后自动生成文档。

### A.2.5 AgentTool（最长的提示词，约290行）

三种模式的提示词截然不同：

**基础模式（无 Fork）：** 核心是"何时不该使用"的负面指引。简单文件搜索用 Glob/Grep/Read，不要为此创建 Agent。

**Fork 模式（isForkSubagentEnabled）：**
- "When to fork"区段：fork 用于不需要保留中间输出的场景
- **"Don't peek"规则**：不要读取 fork 的 `output_file`，等待通知
- **"Don't race"规则**：不要预测 fork 结果，fork 没返回就说"还在跑"
- fork prompt 用指令风格（directive），因为 fork 继承上下文

**Coordinator 模式：** 返回精简提示词（slim prompt），coordinator 自身的系统提示词已包含完整指引。

**Agent 列表注入策略**：

```typescript
// The dynamic agent list was ~10.2% of fleet cache_creation tokens
// MCP async connect, /reload-plugins, or permission-mode changes mutate the list
// -> description changes -> full tool-schema cache bust
```

将 agent 列表从工具描述移到 attachment 消息，避免 MCP 连接变化导致工具 schema 的 prompt cache 完全失效。

### A.2.6 ToolSearch（延迟加载决策）

```typescript
export function isDeferredTool(tool: Tool): boolean {
  if (tool.alwaysLoad === true) return false      // 强制加载
  if (tool.isMcp === true) return true             // MCP 工具总是延迟
  if (tool.name === TOOL_SEARCH_TOOL_NAME) return false  // 自身不延迟
  // Fork 模式下 Agent 工具不延迟（需要 turn 1 可用）
  // Brief 工具不延迟（KAIROS 模式的主通信通道）
  return tool.shouldDefer === true
}
```

查询格式：

| 格式 | 语义 | 示例 |
|------|------|------|
| `select:Name1,Name2` | 按名称精确获取 | `select:Read,Edit,Grep` |
| `keyword1 keyword2` | 关键词搜索 | `notebook jupyter` |
| `+name keyword` | 名称必含 + 关键词排序 | `+slack send` |

### A.2.7 SkillTool（技能预算与截断策略）

```typescript
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 上下文窗口的 1%
export const CHARS_PER_TOKEN = 4
export const DEFAULT_CHAR_BUDGET = 8_000           // 200k x 4 x 1% 的 fallback
export const MAX_LISTING_DESC_CHARS = 250          // 单条描述上限
```

截断策略（formatCommandsWithinBudget）体现"分级降级"：

1. 尝试完整描述
2. 超预算 -> 分离 bundled skills（永不截断）和 rest
3. 计算非 bundled 均分描述长度
4. 均分长度 < 20 字符 -> 非 bundled 只显示名称
5. 否则 -> 均匀截断非 bundled 描述

### A.2.8 GrepTool 和 GlobTool

GrepTool 开篇即最强硬的排他指令：

```
ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command.
The Grep tool has been optimized for correct permissions and access.
```

"optimized for correct permissions and access" 暗示 GrepTool 内置了沙箱权限处理，直接在 Bash 中运行 grep 可能因沙箱限制返回不完整结果。

GlobTool 拥有最精简的提示词（7行）：glob 匹配是 LLM 已充分掌握的概念，不需要额外教学。

### A.2.9 TodoWriteTool（任务管理）

最详尽的"使用指南式"提示词（约185行），包含 5 种应使用场景 + 4 种不应使用场景，4 个正面示例 + 4 个反面示例，每个都有 `<reasoning>` 解释。

任务状态机：`pending -> in_progress -> completed`。关键约束：**任何时刻只有一个 in_progress 任务**，确保注意力集中。

每个任务有双形式描述：content（祈使句，如 "Run tests"）和 activeForm（进行时，如 "Running tests"），让同一数据在不同 UI 上下文中自然呈现。

### A.2.10 设计模式总结

| 模式 | 应用工具 | 核心思路 |
|------|----------|----------|
| 负面清单 | BashTool、GrepTool | 用 NOT/NEVER 明确禁止替代方案 |
| 先读后改 | FileEditTool、FileWriteTool | 提示词约束 + 运行时校验双重保障 |
| 分级降级 | SkillTool | 预算不足时从完整描述到截断描述到仅名称 |
| 示例教学 | TodoWriteTool | 正反案例 + reasoning 建立决策边界 |
| 缓存感知 | AgentTool | 将动态内容移出工具 schema 避免缓存失效 |
| 优先级标记 | BashTool | 用 CRITICAL/IMPORTANT 提升关键规则的遵守率 |
| 模式分裂 | AgentTool | 同一工具根据运行模式生成完全不同的提示词 |

---

## A.3 压缩提示词的三重防线

源码位于 `src/services/compact/prompt.ts`。当对话上下文接近窗口极限时，Claude Code 触发压缩。压缩本身需要调用 LLM 生成摘要，但必须阻止 LLM 在摘要过程中调用工具——否则会产生更多 token，适得其反。

### A.3.1 NO_TOOLS_PREAMBLE：前置封堵

```typescript
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.

`
```

设计要点：

- **放在提示词最前面**。源码注释说明：Sonnet 4.6+ 的 adaptive-thinking 模型有时会忽略尾部较弱的指令而尝试调用工具。由于压缩设置了 `maxTurns: 1`，工具调用被拒绝后不会有文本输出，整轮被浪费。
- **穷举列出所有工具名称**，不留模糊空间。
- **威胁后果量化**："will waste your only turn"——直接告知这是唯一机会。
- **锁定输出格式**：`<analysis>` + `<summary>` 两个 XML 块，使模型没有"调用工具"的输出路径。

背景数据：Sonnet 4.6 的工具调用泄漏率为 2.79%（对比 4.5 的 0.01%），前置封堵专门为此增加。

### A.3.2 NO_TOOLS_TRAILER：尾部双重保险

```typescript
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'
```

**前后夹击**：PREAMBLE 在提示词头部，TRAILER 在尾部。利用 LLM 注意力分布的两个峰值——序列开头和结尾——确保至少有一处禁令被强烈感知。

最终提示词结构：`[NO_TOOLS_PREAMBLE] + [摘要模板] + [用户自定义指令(可选)] + [NO_TOOLS_TRAILER]`。

### A.3.3 完全压缩的九段式摘要模板

`BASE_COMPACT_PROMPT` 定义了严格的九段式摘要框架：

1. **Primary Request and Intent**：捕获用户的显式请求和意图
2. **Key Technical Concepts**：列出讨论过的技术概念、框架
3. **Files and Code Sections**：列举检查/修改/创建的文件，包含完整代码片段
4. **Errors and fixes**：列出所有错误及修复方式，特别关注用户反馈
5. **Problem Solving**：记录已解决的问题和进行中的排查
6. **All user messages**：列出所有非工具结果的用户消息
7. **Pending Tasks**：列出待完成任务
8. **Current Work**：精确描述压缩请求前正在做的工作
9. **Optional Next Step**：下一步操作（必须与用户最近请求直接相关）

关键设计：

- **第6段**强制保留所有用户原始消息，防止对话漂移。压缩中最容易丢失的是用户在中间轮次给出的修正指令（如"不要用递归方案"）。
- **第9段**约束："Do not start on tangential requests or really old requests that were already completed"。还要求"include direct quotes"——通过逐字引用锚定任务方向，避免语义漂移。

### A.3.4 部分压缩的方向性变体

**PARTIAL_COMPACT_PROMPT（direction='from'）：**
- 保留前面的消息，只压缩后面新增部分
- "The earlier messages are being kept intact and do NOT need to be summarized"

**PARTIAL_COMPACT_UP_TO_PROMPT（direction='up_to'）：**
- 压缩前面的消息，保留后面新增部分
- 第8段改为"Work Completed"，第9段改为"Context for Continuing Work"

三种压缩模式的选择在上游决定（compact service 根据缓存命中来选择方向），此处只负责生成对应的提示词。

### A.3.5 `<analysis>` 草稿区

```typescript
const DETAILED_ANALYSIS_INSTRUCTION_BASE = `Before providing your final summary,
wrap your analysis in <analysis> tags to organize your thoughts...

1. Chronologically analyze each message and section of the conversation.
   For each section thoroughly identify:
   - The user's explicit requests and intents
   - Key decisions, technical concepts and code patterns
   - Specific details like: file names, full code snippets, function signatures
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback
2. Double-check for technical accuracy and completeness`
```

`<analysis>` 块等同于 chain-of-thought：让模型先在草稿区按时间线梳理，再产出结构化摘要。`formatCompactSummary()` 函数在最终结果中剥离 `<analysis>` 块，只保留 `<summary>` 内容——草稿消耗的 token 不占用后续上下文空间。

### A.3.6 压缩后的用户消息

```typescript
let baseSummary = `This session is being continued from a previous conversation
that ran out of context. The summary below covers the earlier portion...`
```

根据上下文附加：
- **transcript 路径**：摘要不够精确时可回读原始记录
- **近期消息保留标记**："Recent messages are preserved verbatim"
- **suppressFollowUpQuestions 模式**："Resume directly -- do not acknowledge the summary, do not recap. Pick up the last task as if the break never happened."
- **PROACTIVE/KAIROS 模式**："This is NOT a first wake-up -- you were already working autonomously before compaction."

---

## A.4 记忆检索提示词

源码位于 `src/memdir/findRelevantMemories.ts`。

### A.4.1 AI 选择器的系统提示

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be
useful to Claude Code as it processes a user's query. You will be given the
user's query and a list of available memory files with their filenames and
descriptions.

Return a list of filenames for the memories that will clearly be useful (up to 5).
Only include memories that you are certain will be helpful.
- If you are unsure if a memory will be useful, do not include it.
- If there are no clearly useful memories, return an empty list.
- If a list of recently-used tools is provided, do not select memories that are
  usage reference or API documentation for those tools. DO still select memories
  containing warnings, gotchas, or known issues about those tools — active use
  is exactly when those matter.`
```

### A.4.2 设计策略

**精确度优先于召回率**："Only include memories that you are **certain** will be helpful"、"Be **selective and discerning**"、"feel free to return an **empty list**"。宁可漏选，也不错选——不相关的记忆不仅浪费空间，还可能误导模型。

**工具感知过滤**：正在使用 Bash 工具时，不需要 Bash 使用手册，但关于 Bash 的踩坑记录（如"macOS 上 sed -i 需要备份后缀"）此刻最为重要。

**轻量级实现**：

| 设计点 | 实现 |
|--------|------|
| 选择器模型 | Sonnet（非 Opus），快速且便宜 |
| 输入 | 只传 filename + description（前30行 frontmatter），不加载全文 |
| 上限 | 最多返回 5 个文件，扫描上限 200 个 |
| 调用方式 | sideQuery 独立 API 调用，不影响主对话 token 计数 |
| 预过滤 | alreadySurfaced 排除已展示记忆，节省选择名额 |
| 容错 | 任何失败静默降级为空列表 |
| 验证 | validFilenames 集合验证返回的文件名确实存在，防止幻觉 |

---

## A.5 提示词工程的关键设计原则

从源码中提炼出六条核心设计原则。

### 原则一：Fail-Closed 默认值

```typescript
const TOOL_DEFAULTS = {
  isConcurrencySafe: (_input?) => false,    // 默认不可并发
  isReadOnly: (_input?) => false,            // 默认有副作用
}
```

工具作者忘记声明安全属性时，系统假设该工具**不可并发**且**有副作用**——触发更严格的权限检查和串行化执行。代价是性能损失（不必要的串行化），收益是安全保障。

### 原则二：缓存稳定性优先

至少六个层面的投入：

| 层面 | 机制 |
|------|------|
| 静态/动态分界线 | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 保证静态部分字节级不变 |
| 工具排序固定 | 内置工具 sort by name 作为连续前缀 |
| 沙箱路径抽象 | 用 `$TMPDIR` 替代实际路径，避免跨用户缓存失效 |
| Agent 列表外置 | 移到 attachment 避免 MCP 连接变化导致缓存失效 |
| Fork 子代占位符 | 统一文本确保字节级前缀匹配 |
| 动态内容后置 | 数字长度锚点、session 指引放在边界之后 |

### 原则三：分层防御

安全约束通过多层独立防线实现纵深防御：

| 系统 | 防线层级 |
|------|----------|
| BashTool | Git Safety Protocol（提示词级）+ 沙箱隔离（运行时级）+ 命令规范（输入级） |
| 压缩提示词 | NO_TOOLS_PREAMBLE（头部）+ NO_TOOLS_TRAILER（尾部） |
| 工具权限 | allow / deny / ask / passthrough 四态模型 |

任何单一防线被突破时，其余防线仍然有效。

### 原则四：编译期消除

```typescript
// feature() 函数在构建时被替换为布尔常量
// 未启用的提示词代码被 tree-shaking 完全移除
```

最终交付的代码中，不存在"休眠的"提示词分支。ant 内部功能的提示词在外部构建产物中被物理移除，而不仅仅是"不执行"。

好处：
1. **安全性**：内部专用提示词不会泄露到外部版本
2. **包体积**：减少不必要的字符串常量

### 原则五：写给 AI 的行为指令

工具提示词不是面向人类的 API 文档，而是面向 LLM 的行为训练指令：

| 人类文档风格 | AI 行为指令风格 |
|-------------|----------------|
| "建议在操作前备份" | "**NEVER** run destructive operations without confirmation" |
| "可选：指定超时时间" | "**CRITICAL**: Always quote file paths containing spaces" |
| "注意并发安全性" | "You **MUST** wait for previous calls to finish" |

大写的 NEVER、CRITICAL、MUST 是经过验证的、能有效影响模型行为的信号词，遵从度显著高于委婉表达。

### 原则六：内外版本差异化

通过 `process.env.USER_TYPE === 'ant'` 分支维护两套行为规范：

| 维度 | 外部版本 | ant 内部版本 |
|------|---------|-------------|
| 输出 | 极简，一句话能说的不用三句 | 倒金字塔写作法，完整句子 |
| 注释 | 遵循通用最佳实践 | 默认不写注释（除非 WHY 不明显） |
| 验证 | 信任用户判断 | 完成前必须验证，如实报告不美化 |
| 实验功能 | 无 | Verification Agent、Explore & Plan Agent、数字长度锚点 |

内部版本是外部版本的超集加上更严格的约束。通过 `feature()` 函数和编译期消除（原则四），两套规范在构建时被分离为完全独立的产物。

### 六条原则的内在联系

- **原则一**（fail-closed）和**原则三**（分层防御）共同保障安全性
- **原则二**（缓存稳定性）和**原则四**（编译期消除）共同优化性能
- **原则五**（行为指令）和**原则六**（内外差异化）共同塑造模型行为

底层逻辑是一致的：**在不确定性面前选择安全，在工程实践中追求效率，在模型交互中保持精确控制**。
