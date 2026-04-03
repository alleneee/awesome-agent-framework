# 第11章：插件、Skills与Hook系统

Claude Code 的扩展性建立在三大支柱之上：**插件**（Plugin）、**Skills**（技能）和 **Hooks**（钩子）。三者互相交织，共同构成了一套完整的"可编程能力注入"体系。本章将逐一拆解其内部实现。

## 11.1 插件三来源架构

Claude Code 的插件系统采用三来源架构：**Built-in**（内置）、**Marketplace**（市场）和 **Session**（会话级）。核心类型定义位于 `src/types/plugin.ts`。

### 11.1.1 BuiltinPluginDefinition 类型

```typescript
// src/types/plugin.ts
export type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean
  defaultEnabled?: boolean
}
```

一个内置插件可以同时提供三种组件：skills、hooks 和 mcpServers。`isAvailable` 函数用于动态判断当前系统是否支持（例如某些插件只在 macOS 上可用），返回 `false` 时插件完全隐藏。

### 11.1.2 LoadedPlugin 类型

```typescript
// src/types/plugin.ts
export type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string
  source: string
  repository: string
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string
  commandsPath?: string
  commandsPaths?: string[]
  agentsPath?: string
  skillsPath?: string
  skillsPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

`LoadedPlugin` 是运行时统一的插件表示。无论来源是 Built-in、Marketplace 还是 Session，加载后都归一化为此类型。`source` 字段标识来源（如 `myPlugin@builtin`、`myPlugin@marketplace-name`）。

### 11.1.3 内置插件注册与状态管理

```typescript
// src/plugins/builtinPlugins.ts
const BUILTIN_PLUGINS: Map<string, BuiltinPluginDefinition> = new Map()

export function registerBuiltinPlugin(
  definition: BuiltinPluginDefinition,
): void {
  BUILTIN_PLUGINS.set(definition.name, definition)
}
```

内置插件在启动时通过 `registerBuiltinPlugin()` 注册到全局 Map 中。ID 格式为 `{name}@builtin`。

状态管理的核心在 `getBuiltinPlugins()` 函数：

```typescript
// src/plugins/builtinPlugins.ts
export function getBuiltinPlugins(): {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
} {
  const settings = getSettings_DEPRECATED()
  const enabled: LoadedPlugin[] = []
  const disabled: LoadedPlugin[] = []

  for (const [name, definition] of BUILTIN_PLUGINS) {
    if (definition.isAvailable && !definition.isAvailable()) {
      continue  // 不可用的插件完全跳过
    }

    const pluginId = `${name}@${BUILTIN_MARKETPLACE_NAME}`
    const userSetting = settings?.enabledPlugins?.[pluginId]
    // 优先级：用户设置 > 插件默认值 > true
    const isEnabled =
      userSetting !== undefined
        ? userSetting === true
        : (definition.defaultEnabled ?? true)

    const plugin: LoadedPlugin = {
      name,
      manifest: { name, description: definition.description, version: definition.version },
      path: BUILTIN_MARKETPLACE_NAME,  // 哨兵值，无文件系统路径
      source: pluginId,
      repository: pluginId,
      enabled: isEnabled,
      isBuiltin: true,
      hooksConfig: definition.hooks,
      mcpServers: definition.mcpServers,
    }

    if (isEnabled) {
      enabled.push(plugin)
    } else {
      disabled.push(plugin)
    }
  }

  return { enabled, disabled }
}
```

三层决策逻辑：
1. `isAvailable()` 返回 false -> 完全隐藏
2. 用户在 settings 中显式设置了 `enabledPlugins[pluginId]` -> 遵从用户设置
3. 都没有 -> 取 `defaultEnabled`，默认为 `true`

### 11.1.4 插件加载与合并流程

`AppState` 中的 `plugins` 域存储了所有已加载的插件状态：

```typescript
// src/state/AppStateStore.ts
plugins: {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  commands: Command[]
  errors: PluginError[]
  installationStatus: { ... }
  needsRefresh: boolean
}
```

Marketplace 插件通过 Git 克隆到本地缓存目录，然后解析 `manifest.json` 中声明的 skills、hooks、MCP servers 等组件路径。三种来源的插件最终合并到同一个 `plugins` 状态域中。

## 11.2 Skills 系统

### 11.2.1 Skills 的六种来源

Skills 的加载来源由 `LoadedFrom` 类型定义：

```typescript
// src/skills/loadSkillsDir.ts
export type LoadedFrom =
  | 'commands_DEPRECATED'  // 旧版 /commands/ 目录（已废弃）
  | 'skills'               // 文件系统 skills/ 目录
  | 'plugin'               // 插件提供的 skills
  | 'managed'              // 企业管理策略注入的 skills
  | 'bundled'              // 编译进二进制的内置 skills
  | 'mcp'                  // MCP 服务器暴露的 skills
```

加载路径由 `getSkillsPath()` 函数按来源路由：

```typescript
// src/skills/loadSkillsDir.ts
export function getSkillsPath(
  source: SettingSource | 'plugin',
  dir: 'skills' | 'commands',
): string {
  switch (source) {
    case 'policySettings':
      return join(getManagedFilePath(), '.claude', dir)
    case 'userSettings':
      return join(getClaudeConfigHomeDir(), dir)
    case 'projectSettings':
      return `.claude/${dir}`
    case 'plugin':
      return 'plugin'
    default:
      return ''
  }
}
```

六种来源的优先级排列如下：
- **policySettings** -> 企业管理目录下的 `.claude/skills/`
- **userSettings** -> 用户 home 目录下的 `~/.claude/skills/`
- **projectSettings** -> 项目目录下的 `.claude/skills/`
- **plugin** -> 通过插件系统加载的 skills
- **bundled** -> 编译进二进制的内置 skills
- **mcp** -> 从 MCP 服务器拉取的 skills

### 11.2.2 Skill 文件格式：SKILL.md + frontmatter

Skills 采用目录格式：`skill-name/SKILL.md`。不支持单文件格式。

```typescript
// src/skills/loadSkillsDir.ts
async function loadSkillsFromSkillsDir(
  basePath: string,
  source: SettingSource,
): Promise<SkillWithPath[]> {
  const fs = getFsImplementation()
  let entries
  try {
    entries = await fs.readdir(basePath)
  } catch (e: unknown) {
    if (!isFsInaccessible(e)) logError(e)
    return []
  }

  const results = await Promise.all(
    entries.map(async (entry): Promise<SkillWithPath | null> => {
      // 只支持目录格式
      if (!entry.isDirectory() && !entry.isSymbolicLink()) {
        return null
      }

      const skillDirPath = join(basePath, entry.name)
      const skillFilePath = join(skillDirPath, 'SKILL.md')

      let content: string
      try {
        content = await fs.readFile(skillFilePath, { encoding: 'utf-8' })
      } catch (e: unknown) {
        if (!isENOENT(e)) {
          logForDebugging(`[skills] failed to read ${skillFilePath}: ${e}`, {
            level: 'warn',
          })
        }
        return null
      }
      // ... 解析 frontmatter 并创建 Command
    }),
  )
}
```

SKILL.md 的 frontmatter 支持丰富的元数据字段：

```typescript
// src/skills/loadSkillsDir.ts
export function parseSkillFrontmatterFields(
  frontmatter: FrontmatterData,
  markdownContent: string,
  resolvedName: string,
): {
  displayName: string | undefined
  description: string
  hasUserSpecifiedDescription: boolean
  allowedTools: string[]
  argumentHint: string | undefined
  argumentNames: string[]
  whenToUse: string | undefined
  version: string | undefined
  model: ReturnType<typeof parseUserSpecifiedModel> | undefined
  disableModelInvocation: boolean
  userInvocable: boolean
  hooks: HooksSettings | undefined
  executionContext: 'fork' | undefined
  agent: string | undefined
  effort: EffortValue | undefined
  shell: FrontmatterShell | undefined
} {
  // ... 解析逻辑
}
```

一个完整的 SKILL.md 示例 frontmatter：

```markdown
---
name: my-skill
description: 执行某个特定任务
when_to_use: 当用户需要...
allowed-tools: ["Bash", "Read", "Write"]
context: fork
model: claude-sonnet-4-6
argument-hint: "<文件路径>"
arguments: [file, mode]
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "echo pre-hook"
---

Skill 正文内容...
```

### 11.2.3 BundledSkillDefinition：编译进二进制的 Skill

```typescript
// src/skills/bundledSkills.ts
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // 附带的参考文件
  getPromptForCommand: (
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}
```

`files` 字段允许内置 skill 携带参考文件。首次调用时，这些文件被延迟提取到磁盘：

```typescript
// src/skills/bundledSkills.ts
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const { files } = definition

  let skillRoot: string | undefined
  let getPromptForCommand = definition.getPromptForCommand

  if (files && Object.keys(files).length > 0) {
    skillRoot = getBundledSkillExtractDir(definition.name)
    let extractionPromise: Promise<string | null> | undefined
    const inner = definition.getPromptForCommand
    getPromptForCommand = async (args, ctx) => {
      // 闭包级记忆化：同一进程只提取一次
      // 记忆化的是 Promise 而非结果，避免并发调用者竞争写入
      extractionPromise ??= extractBundledSkillFiles(definition.name, files)
      const extractedDir = await extractionPromise
      const blocks = await inner(args, ctx)
      if (extractedDir === null) return blocks
      return prependBaseDir(blocks, extractedDir)
    }
  }
  // ... 构建 Command 并注册
}
```

提取文件时的安全措施值得关注：

```typescript
// src/skills/bundledSkills.ts
// 进程内 nonce 是防止预创建符号链接/目录的主要防御
// 显式 0o700/0o600 模式在 umask=0 时也保持目录仅主人可访问
// O_NOFOLLOW|O_EXCL 是额外的安全带
const O_NOFOLLOW = fsConstants.O_NOFOLLOW ?? 0
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW

async function safeWriteFile(p: string, content: string): Promise<void> {
  const fh = await open(p, SAFE_WRITE_FLAGS, 0o600)
  try {
    await fh.writeFile(content, 'utf8')
  } finally {
    await fh.close()
  }
}
```

路径遍历防护：

```typescript
function resolveSkillFilePath(baseDir: string, relPath: string): string {
  const normalized = normalize(relPath)
  if (
    isAbsolute(normalized) ||
    normalized.split(pathSep).includes('..') ||
    normalized.split('/').includes('..')
  ) {
    throw new Error(`bundled skill file path escapes skill dir: ${relPath}`)
  }
  return join(baseDir, normalized)
}
```

### 11.2.4 MCP Skill 桥接

MCP 服务器也可以暴露 skills。桥接层通过 `mcpSkillBuilders.ts` 实现：

```typescript
// src/skills/mcpSkillBuilders.ts
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error(
      'MCP skill builders not registered — loadSkillsDir.ts has not been evaluated yet',
    )
  }
  return builders
}
```

这是一个"写一次"注册模式，用于打破循环依赖。`loadSkillsDir.ts` 在模块初始化时注册，MCP client 在后续使用。关键约束：MCP skills 的 markdown 正文中禁止执行 shell 命令：

```typescript
// src/skills/loadSkillsDir.ts
// 安全：MCP skills 是远程且不可信的 —— 永远不执行其 markdown 正文中的
// 内联 shell 命令（!`...` / ```! ... ```）
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

## 11.3 SkillTool 的执行流程

### 11.3.1 工具定义

```typescript
// src/tools/SkillTool/SkillTool.ts
export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  name: SKILL_TOOL_NAME,  // 'Skill'
  searchHint: 'invoke a slash-command skill',
  maxResultSizeChars: 100_000,
  // ...
})
```

输入 schema 非常简洁：

```typescript
export const inputSchema = lazySchema(() =>
  z.object({
    skill: z.string().describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
    args: z.string().optional().describe('Optional arguments for the skill'),
  }),
)
```

### 11.3.2 inline vs fork 执行

SkillTool 的 `call()` 方法根据 skill 的 `context` 字段选择执行策略：

```typescript
// src/tools/SkillTool/SkillTool.ts
async call({ skill, args }, context, canUseTool, parentMessage, onProgress?) {
  const commandName = trimmed.startsWith('/') ? trimmed.substring(1) : trimmed
  const commands = await getAllCommands(context)
  const command = findCommand(commandName, commands)

  recordSkillUsage(commandName)

  // fork 模式：在隔离的子 agent 中执行
  if (command?.type === 'prompt' && command.context === 'fork') {
    return executeForkedSkill(
      command, commandName, args, context,
      canUseTool, parentMessage, onProgress,
    )
  }

  // inline 模式（默认）：在当前上下文中展开 prompt
  const { processPromptSlashCommand } = await import(
    'src/utils/processUserInput/processSlashCommand.js'
  )
  const processedCommand = await processPromptSlashCommand(
    commandName, args || '', commands, context,
  )
  // ...
}
```

**inline 模式**：skill 的 prompt 被展开后注入到当前对话流中，模型继续处理。输出 schema：

```typescript
const inlineOutputSchema = z.object({
  success: z.boolean(),
  commandName: z.string(),
  allowedTools: z.array(z.string()).optional(),
  model: z.string().optional(),
  status: z.literal('inline').optional(),
})
```

**fork 模式**：启动一个隔离的子 agent，有自己的 token 预算和消息流：

```typescript
// src/tools/SkillTool/SkillTool.ts
async function executeForkedSkill(command, commandName, args, context, canUseTool, parentMessage, onProgress?) {
  const agentId = createAgentId()
  const { modifiedGetAppState, baseAgent, promptMessages, skillContent } =
    await prepareForkedCommandContext(command, args || '', context)

  const agentDefinition = command.effort !== undefined
    ? { ...baseAgent, effort: command.effort }
    : baseAgent

  const agentMessages: Message[] = []

  for await (const message of runAgent({
    agentDefinition,
    promptMessages,
    toolUseContext: { ...context, getAppState: modifiedGetAppState },
    canUseTool,
    isAsync: false,
    querySource: 'agent:custom',
    model: command.model as ModelAlias | undefined,
    availableTools: context.options.tools,
    override: { agentId },
  })) {
    agentMessages.push(message)
    // 向父级报告工具使用进度
    if ((message.type === 'assistant' || message.type === 'user') && onProgress) {
      // ...
    }
  }

  const resultText = extractResultText(agentMessages, 'Skill execution completed')
  agentMessages.length = 0  // 释放消息内存

  return {
    data: {
      success: true, commandName, status: 'forked',
      agentId, result: resultText,
    },
  }
}
```

fork 模式的关键特性：
- 独立的 agentId
- 独立的 token 预算
- 执行完成后通过 `clearInvokedSkillsForAgent(agentId)` 清理状态
- 可以指定独立的 model 和 effort 级别

### 11.3.3 Skill 列表的预算管理

Skill 列表占用 system prompt 的空间有严格限制：

```typescript
// src/tools/SkillTool/prompt.ts
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 上下文窗口的 1%
export const CHARS_PER_TOKEN = 4
export const DEFAULT_CHAR_BUDGET = 8_000  // 后备值：200k * 1% * 4
export const MAX_LISTING_DESC_CHARS = 250  // 单条描述上限

export function getCharBudget(contextWindowTokens?: number): number {
  if (Number(process.env.SLASH_COMMAND_TOOL_CHAR_BUDGET)) {
    return Number(process.env.SLASH_COMMAND_TOOL_CHAR_BUDGET)
  }
  if (contextWindowTokens) {
    return Math.floor(contextWindowTokens * CHARS_PER_TOKEN * SKILL_BUDGET_CONTEXT_PERCENT)
  }
  return DEFAULT_CHAR_BUDGET
}
```

当所有 skill 的完整描述超出预算时，系统执行分层截断：
1. 先尝试完整描述
2. 超出则划分为 bundled（不截断）和 rest（截断描述）
3. 极端情况下，non-bundled skills 只保留名称

```typescript
export function formatCommandsWithinBudget(commands: Command[], contextWindowTokens?: number): string {
  const budget = getCharBudget(contextWindowTokens)
  const fullEntries = commands.map(cmd => ({ cmd, full: formatCommandDescription(cmd) }))
  const fullTotal = fullEntries.reduce((sum, e) => sum + stringWidth(e.full), 0) + (fullEntries.length - 1)

  if (fullTotal <= budget) {
    return fullEntries.map(e => e.full).join('\n')
  }

  // 分区：bundled 不截断，其余截断
  const bundledIndices = new Set<number>()
  // ...
  const maxDescLen = Math.floor(availableForDescs / restCommands.length)

  if (maxDescLen < MIN_DESC_LENGTH) {
    // 极端情况：non-bundled 只显示名称
    return commands.map((cmd, i) =>
      bundledIndices.has(i) ? fullEntries[i]!.full : `- ${cmd.name}`,
    ).join('\n')
  }
  // ...
}
```

## 11.4 Hook 系统

### 11.4.1 27 种事件类型

Hook 事件定义在 SDK 类型中：

```typescript
// src/entrypoints/sdk/coreTypes.ts
export const HOOK_EVENTS = [
  'PreToolUse',
  'PostToolUse',
  'PostToolUseFailure',
  'Notification',
  'UserPromptSubmit',
  'SessionStart',
  'SessionEnd',
  'Stop',
  'StopFailure',
  'SubagentStart',
  'SubagentStop',
  'PreCompact',
  'PostCompact',
  'PermissionRequest',
  'PermissionDenied',
  'Setup',
  'TeammateIdle',
  'TaskCreated',
  'TaskCompleted',
  'Elicitation',
  'ElicitationResult',
  'ConfigChange',
  'WorktreeCreate',
  'WorktreeRemove',
  'InstructionsLoaded',
  'CwdChanged',
  'FileChanged',
] as const
```

这些事件覆盖了 Claude Code 的完整生命周期：
- **工具级**：PreToolUse、PostToolUse、PostToolUseFailure
- **会话级**：SessionStart、SessionEnd、UserPromptSubmit
- **采样级**：Stop、StopFailure
- **子代理级**：SubagentStart、SubagentStop
- **压缩级**：PreCompact、PostCompact
- **权限级**：PermissionRequest���PermissionDenied
- **系统级**：Setup、ConfigChange、CwdChanged、FileChanged、InstructionsLoaded
- **任务级**：TaskCreated、TaskCompleted、TeammateIdle
- **MCP 级**：Elicitation、ElicitationResult
- **Worktree 级**：WorktreeCreate、WorktreeRemove
- **通知级**：Notification

### 11.4.2 4 种 Hook 类型

Hook 的 schema 定义在 `src/schemas/hooks.ts` 中，使用 Zod discriminatedUnion：

```typescript
// src/schemas/hooks.ts
export const HookCommandSchema = lazySchema(() => {
  const { BashCommandHookSchema, PromptHookSchema, AgentHookSchema, HttpHookSchema } = buildHookSchemas()
  return z.discriminatedUnion('type', [
    BashCommandHookSchema,
    PromptHookSchema,
    AgentHookSchema,
    HttpHookSchema,
  ])
})
```

**1. command 类型（Shell 命令）**：

```typescript
const BashCommandHookSchema = z.object({
  type: z.literal('command'),
  command: z.string(),
  if: IfConditionSchema(),
  shell: z.enum(SHELL_TYPES).optional(),
  timeout: z.number().positive().optional(),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),       // 执行一次后自动移除
  async: z.boolean().optional(),      // 后台运行，不阻塞
  asyncRewake: z.boolean().optional(), // 后台运行，exit code 2 时唤醒模型
})
```

`asyncRewake` 是一个精妙的设计：hook 在后台运行，如果退出码为 2（表示"阻塞性错误"），会通过消息队列唤醒模型处理。

**2. prompt 类型（LLM 提示）**：

```typescript
const PromptHookSchema = z.object({
  type: z.literal('prompt'),
  prompt: z.string(),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional(),
  model: z.string().optional(),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

prompt hook 将输入交给 LLM 处理，`$ARGUMENTS` 占位符会被替换为 hook 输入的 JSON。

**3. http 类型（HTTP 回调）**：

```typescript
const HttpHookSchema = z.object({
  type: z.literal('http'),
  url: z.string().url(),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional(),
  headers: z.record(z.string(), z.string()).optional(),
  allowedEnvVars: z.array(z.string()).optional(),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

headers 字段支持环境变量插值（`$VAR_NAME` 或 `${VAR_NAME}`），但只有在 `allowedEnvVars` 中显式列出的变量才会被解析。

**4. agent 类型（代理验证器）**：

```typescript
const AgentHookSchema = z.object({
  type: z.literal('agent'),
  prompt: z.string(),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional(),
  model: z.string().optional(),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

agent hook 启动一个独立的 agent 来验证操作，比 prompt hook 更重量级，可以调用工具。

### 11.4.3 Hook 配置合并：三层架构

Hook 配置来自三个层级：

```typescript
// src/schemas/hooks.ts
export const HooksSchema = lazySchema(() =>
  z.partialRecord(z.enum(HOOK_EVENTS), z.array(HookMatcherSchema())),
)

export const HookMatcherSchema = lazySchema(() =>
  z.object({
    matcher: z.string().optional(),
    hooks: z.array(HookCommandSchema()),
  }),
)
```

三层来源：
1. **Settings 层**：`settings.json` 中的 `hooks` 字段，通过常规 settings 合并机制（详见第12章）
2. **SDK/Plugin 层**：插件通过 `hooksConfig` 字段注册，Skill 通过 frontmatter `hooks:` 字段注册
3. **Session 层**：运行时通过 `registerSessionHook()` 动态注册的临时 hook

```typescript
// src/utils/hooks.ts
const TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000

const SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500
export function getSessionEndHookTimeoutMs(): number {
  const raw = process.env.CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS
  const parsed = raw ? parseInt(raw, 10) : NaN
  return Number.isFinite(parsed) && parsed > 0
    ? parsed
    : SESSION_END_HOOK_TIMEOUT_MS_DEFAULT
}
```

工具 hook 默认超时 10 分钟，SessionEnd hook 更紧（1.5 秒），因为它在关闭/清理路径上。

### 11.4.4 Hook 匹配逻辑

匹配函数 `matchesPattern()` 支持三种模式：

```typescript
// src/utils/hooks.ts
function matchesPattern(matchQuery: string, matcher: string): boolean {
  if (!matcher || matcher === '*') {
    return true  // 空 matcher 或 * 匹配所有
  }

  // 纯字母数字 + 管道分隔符：精确匹配
  if (/^[a-zA-Z0-9_|]+$/.test(matcher)) {
    if (matcher.includes('|')) {
      // 管道分隔的精确匹配列表
      const patterns = matcher.split('|').map(p => normalizeLegacyToolName(p.trim()))
      return patterns.includes(matchQuery)
    }
    // 单值精确匹配
    return matchQuery === normalizeLegacyToolName(matcher)
  }

  // 否则视为正则表达式
  try {
    const regex = new RegExp(matcher)
    if (regex.test(matchQuery)) {
      return true
    }
    // 也测试遗留工具名，使 "^Task$" 等模式仍然有效
    for (const legacyName of getLegacyToolNames(matchQuery)) {
      if (regex.test(legacyName)) {
        return true
      }
    }
    return false
  } catch {
    logForDebugging(`Invalid regex pattern in hook matcher: ${matcher}`)
    return false
  }
}
```

三种匹配模式总结：
- **精确匹配**：`"Write"` 匹配工具名 Write
- **管道分隔匹配**：`"Edit|Write"` 匹配 Edit 或 Write
- **正则匹配**：`"^(Read|Grep)"` 匹配以 Read 或 Grep 开头的工具名

`if` 条件字段提供了更细粒度的过滤，使用权限规则语法（如 `"Bash(git *)"` 只在 Bash 工具执行 git 命令时触发）。

### 11.4.5 Hook 的 JSON stdout 协议

Hook 的 shell 命令通过 stdout 返回 JSON 来控制 Claude Code 的行为。相关类型定义在 `src/types/hooks.ts` 中：

```typescript
// src/types/hooks.ts（通过 import 可见）
type SyncHookJSONOutput = {
  decision?: 'allow' | 'deny' | 'block'
  reason?: string
  suppressOutput?: boolean
  // ... 权限修改等
}

type AsyncHookJSONOutput = {
  async?: boolean
  processId?: string
}
```

同步输出允许 hook 做出权限决策（allow/deny/block），异步输出则将 hook 推入后台注册。

后台执行逻辑：

```typescript
// src/utils/hooks.ts
function executeInBackground({ processId, hookId, shellCommand, asyncResponse, hookEvent, hookName, command, asyncRewake, pluginId }) {
  if (asyncRewake) {
    // asyncRewake hook 不进入注册表，完成后如果 exit code 2 则唤醒模型
    void shellCommand.result.then(async result => {
      await new Promise(resolve => setImmediate(resolve))
      const stdout = await shellCommand.taskOutput.getStdout()
      const stderr = shellCommand.taskOutput.getStderr()
      shellCommand.cleanup()
      if (result.code === 2) {
        enqueuePendingNotification({
          value: wrapInSystemReminder(
            `Stop hook blocking error from command "${hookName}": ${stderr || stdout}`,
          ),
          mode: 'task-notification',
        })
      }
    })
    return true
  }

  // 普通异步 hook 推入后台
  if (!shellCommand.background(processId)) {
    return false
  }
  registerPendingAsyncHook({ processId, hookId, asyncResponse, hookEvent, hookName, command, shellCommand, pluginId })
  return true
}
```

### 11.4.6 信任与安全

Hook 执行受到工作区信任约束。来自 `projectSettings` 的 hook 只在用户接受了信任对话框后才执行。来自 `policySettings` 的 managed hook 在企业环境下可以绕过此限制。`shouldAllowManagedHooksOnly()` 和 `shouldDisableAllHooksIncludingManaged()` 提供了精细的控制粒度。

## 11.5 MCP 工具创建模板模式

### 11.5.1 MCPTool 作为模板

`MCPTool` 的定义是一个"空壳"模���：

```typescript
// src/tools/MCPTool/MCPTool.ts
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',                 // 被覆盖
  maxResultSizeChars: 100_000,
  async description() { return DESCRIPTION },  // 被覆盖
  async prompt() { return PROMPT },            // 被覆盖
  async call() { return { data: '' } },        // 被覆盖
  // ...
})
```

`prompt.ts` 中两个常量都是空字符串：

```typescript
// src/tools/MCPTool/prompt.ts
export const PROMPT = ''
export const DESCRIPTION = ''
```

### 11.5.2 具体 MCP 工具的生成

真正的工具实例在 `src/services/mcp/client.ts` 中通过展开操作符 `...MCPTool` 创建：

```typescript
// src/services/mcp/client.ts
return toolsToProcess.map((tool): Tool => {
  const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
  return {
    ...MCPTool,
    name: skipPrefix ? tool.name : fullyQualifiedName,
    mcpInfo: { serverName: client.name, toolName: tool.name },
    isMcp: true,
    searchHint: typeof tool._meta?.['anthropic/searchHint'] === 'string'
      ? tool._meta['anthropic/searchHint'].replace(/\s+/g, ' ').trim() || undefined
      : undefined,
    alwaysLoad: tool._meta?.['anthropic/alwaysLoad'] === true,
    async description() {
      return tool.description ?? ''
    },
    async prompt() {
      const desc = tool.description ?? ''
      return desc.length > MAX_MCP_DESCRIPTION_LENGTH
        ? desc.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '... [truncated]'
        : desc
    },
    isConcurrencySafe() {
      return tool.annotations?.readOnlyHint ?? false
    },
    isReadOnly() {
      return tool.annotations?.readOnlyHint ?? false
    },
    isDestructive() {
      return tool.annotations?.destructiveHint ?? false
    },
    isOpenWorld() {
      return tool.annotations?.openWorldHint ?? false
    },
    inputJSONSchema: tool.inputSchema as Tool['inputJSONSchema'],
    async checkPermissions() {
      return {
        behavior: 'passthrough' as const,
        message: 'MCPTool requires permission.',
        suggestions: [{
          type: 'addRules' as const,
          rules: [{ toolName: fullyQualifiedName, ruleContent: undefined }],
          behavior: 'allow' as const,
          destination: 'localSettings' as const,
        }],
      }
    },
    async call(args, context, _canUseTool, parentMessage, onProgress?) {
      // 实际 MCP 工具调用逻辑
      // ...
    },
  }
})
```

设计要点：
- 通过 `...MCPTool` 继承模板的基础设施（renderUI、mapToolResult 等）
- 覆盖 name、description、prompt、call 等关键方法
- `mcpInfo` 存储原始的服务器名和工具名，用于权限检查
- `annotations` 元数据（readOnlyHint、destructiveHint、openWorldHint）从 MCP 工具声明中透传
- `'anthropic/searchHint'` 和 `'anthropic/alwaysLoad'` 是 Claude Code 特有的 MCP `_meta` 扩���

### 11.5.3 资源预取架构

```typescript
// src/services/mcp/client.ts
export function prefetchAllMcpResources(
  mcpConfigs: Record<string, ScopedMcpServerConfig>,
): Promise<{
  clients: MCPServerConnection[]
  tools: Tool[]
  commands: Command[]
}> {
  return new Promise(resolve => {
    let pendingCount = Object.keys(mcpConfigs).length
    let completedCount = 0

    if (pendingCount === 0) {
      void resolve({ clients: [], tools: [], commands: [] })
      return
    }

    const clients: MCPServerConnection[] = []
    const tools: Tool[] = []
    const commands: Command[] = []

    getMcpToolsCommandsAndResources(result => {
      clients.push(result.client)
      tools.push(...result.tools)
      commands.push(...result.commands)
      completedCount++
      if (completedCount >= pendingCount) {
        void resolve({ clients, tools, commands })
      }
    }, mcpConfigs).catch(error => {
      void resolve({ clients: [], tools: [], commands: [] })
    })
  })
}
```

采用回调计数器模式而非 `Promise.all()`，这样每个服务器连接完成后立即处理，且单个服务器失败不阻塞其他服务器。

## 11.6 Cron 调度机制

### 11.6.1 三个工具

Cron 系统由三个工具组成：
- **CronCreate**：创建定时任务
- **CronDelete**：删除定时任务
- **CronList**：列出所有定时任务

### 11.6.2 内存 vs 持久化

```typescript
// src/tools/ScheduleCronTool/CronCreateTool.ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    cron: z.string().describe('Standard 5-field cron expression in local time'),
    prompt: z.string().describe('The prompt to enqueue at each fire time.'),
    recurring: semanticBoolean(z.boolean().optional()).describe(
      `true (default) = fire on every cron match. false = fire once then auto-delete.`,
    ),
    durable: semanticBoolean(z.boolean().optional()).describe(
      'true = persist to .claude/scheduled_tasks.json. false (default) = in-memory only.',
    ),
  }),
)
```

两种持久化模式：
- `durable: false`（默认）：任务仅存在于内存，进程退出后消失
- `durable: true`：写入 `.claude/scheduled_tasks.json`，跨会话持久

```typescript
async call({ cron, prompt, recurring = true, durable = false }) {
  const effectiveDurable = durable && isDurableCronEnabled()
  const id = await addCronTask(cron, prompt, recurring, effectiveDurable, getTeammateContext()?.agentId)
  setScheduledTasksEnabled(true)
  return {
    data: { id, humanSchedule: cronToHuman(cron), recurring, durable: effectiveDurable },
  }
}
```

### 11.6.3 抖动设计（Jitter）

Cron 的抖动设计是防止"雷鸣畜群"效应的精妙工程：

```typescript
// src/utils/cronTasks.ts
export type CronJitterConfig = {
  recurringFrac: number      // 周期的 10%
  recurringCapMs: number     // 最大 15 分钟
  oneShotMaxMs: number       // 一次性任务最大 90 秒
  oneShotFloorMs: number     // 最小前移量
  oneShotMinuteMod: number   // 触发抖动的分钟模数（30 = :00 和 :30）
  recurringMaxAgeMs: number  // 循环任务最大存活 7 天
}

export const DEFAULT_CRON_JITTER_CONFIG: CronJitterConfig = {
  recurringFrac: 0.1,
  recurringCapMs: 15 * 60 * 1000,
  oneShotMaxMs: 90 * 1000,
  oneShotFloorMs: 0,
  oneShotMinuteMod: 30,
  recurringMaxAgeMs: 7 * 24 * 60 * 60 * 1000,
}
```

抖动分数来自 taskId 的前 8 位十六进制字符：

```typescript
function jitterFrac(taskId: string): number {
  const frac = parseInt(taskId.slice(0, 8), 16) / 0x1_0000_0000
  return Number.isFinite(frac) ? frac : 0
}
```

**周期性任务**的正向抖动：

```typescript
export function jitteredNextCronRunMs(
  cron: string, fromMs: number, taskId: string,
  cfg: CronJitterConfig = DEFAULT_CRON_JITTER_CONFIG,
): number | null {
  const t1 = nextCronRunMs(cron, fromMs)
  if (t1 === null) return null
  const t2 = nextCronRunMs(cron, t1)
  if (t2 === null) return t1
  // 抖动 = min(taskId分数 * 10% * 周期间隔, 15分钟)
  const jitter = Math.min(
    jitterFrac(taskId) * cfg.recurringFrac * (t2 - t1),
    cfg.recurringCapMs,
  )
  return t1 + jitter
}
```

每小时任务的抖动范围是 [0, 6分钟)，每分钟任务的抖动只有几秒。

**一次性任务**的反向抖动：

```typescript
export function oneShotJitteredNextCronRunMs(
  cron: string, fromMs: number, taskId: string,
  cfg: CronJitterConfig = DEFAULT_CRON_JITTER_CONFIG,
): number | null {
  const t1 = nextCronRunMs(cron, fromMs)
  if (t1 === null) return null
  // 只对整点和半点触发抖动
  if (new Date(t1).getMinutes() % cfg.oneShotMinuteMod !== 0) return t1
  const lead = cfg.oneShotFloorMs +
    jitterFrac(taskId) * (cfg.oneShotMaxMs - cfg.oneShotFloorMs)
  return Math.max(t1 - lead, fromMs)
}
```

一次性任务使用反向抖动（提前执行），因为延迟会破坏"提醒我在3点"的承诺，而提前几十秒用户不会察觉。只在 :00 和 :30 这种"热点"分钟触发抖动。

### 11.6.4 Gate 控制

```typescript
// src/tools/ScheduleCronTool/prompt.ts
export function isKairosCronEnabled(): boolean {
  return feature('AGENT_TRIGGERS')
    ? !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CRON) &&
        getFeatureValue_CACHED_WITH_REFRESH(
          'tengu_kairos_cron',
          true,           // 默认开启
          KAIROS_CRON_REFRESH_MS,  // 5 分钟刷新
        )
    : false
}
```

双层 gate：编译时 feature flag（死代码消除）+ 运行时 GrowthBook gate（5 分钟刷新作为紧急关闭开关）。默认值为 `true` 是因为 `/loop` 已 GA，Bedrock/Vertex/Foundry 用户无法访问 GrowthBook。

## 11.7 本章小结

Claude Code 的扩展体系形成了一个完整的能力注入闭环：

1. **插件**提供宏观的能力包（skills + hooks + MCP servers），支持启用/禁用切换
2. **Skills** 是面向模型的"可调用程序"，六种来源、两种执行模式（inline/fork），支持丰富的 frontmatter 配置
3. **Hooks** 是面向开发者的"事件回调"，27 种事件、4 种类型、三层配置合并、三种匹配模式
4. **MCP** 工具通过模板模式批量生成，将外部能力无缝接入
5. **Cron** 调度为异步任务提供了内存/持久双模式和精巧的抖动设计
