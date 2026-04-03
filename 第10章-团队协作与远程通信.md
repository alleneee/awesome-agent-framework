# 第10章：团队协作与远程通信

本章讲解 Claude Code 的多 Agent 团队协作机制，包括 Team 创建与管理、消息路由、Shutdown 协议、Plan 审批、远程 WebSocket 通信、以及进程内 Teammate 的完整运行循环。

---

## 10.1 Team 创建与配置

### 10.1.1 TeamCreateTool

文件：`src/tools/TeamCreateTool/TeamCreateTool.ts`

**输入 Schema**：

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    team_name: z.string().describe('Name for the new team to create.'),
    description: z.string().optional().describe('Team description/purpose.'),
    agent_type: z.string().optional().describe(
      'Type/role of the team lead (e.g., "researcher", "test-runner").'
    ),
  })
)
```

**创建流程**：

```typescript
async call(input, context) {
  const { setAppState, getAppState } = context
  const { team_name, description, agent_type } = input

  // 1. 检查是否已有团队（每个 Leader 只能管理一个）
  const existingTeam = appState.teamContext?.teamName
  if (existingTeam) {
    throw new Error(`Already leading team "${existingTeam}". ...`)
  }

  // 2. 团队名冲突时自动生成唯一名称
  const finalTeamName = generateUniqueTeamName(team_name)

  // 3. 创建 Leader 的确定性 Agent ID
  const leadAgentId = formatAgentId(TEAM_LEAD_NAME, finalTeamName)

  // 4. 构造 TeamFile
  const teamFile: TeamFile = {
    name: finalTeamName,
    description,
    createdAt: Date.now(),
    leadAgentId,
    leadSessionId: getSessionId(),
    members: [{
      agentId: leadAgentId, name: TEAM_LEAD_NAME,
      agentType: leadAgentType, model: leadModel,
      joinedAt: Date.now(), tmuxPaneId: '', cwd: getCwd(),
      subscriptions: [],
    }],
  }

  // 5. 写入磁盘并注册清理
  await writeTeamFileAsync(finalTeamName, teamFile)
  registerTeamForSessionCleanup(finalTeamName)

  // 6. 重置任务列表
  const taskListId = sanitizeName(finalTeamName)
  await resetTaskList(taskListId)
  await ensureTasksDir(taskListId)
  setLeaderTeamName(sanitizeName(finalTeamName))

  // 7. 更新 AppState
  setAppState(prev => ({
    ...prev,
    teamContext: {
      teamName: finalTeamName, teamFilePath,
      leadAgentId,
      teammates: {
        [leadAgentId]: {
          name: TEAM_LEAD_NAME, agentType: leadAgentType,
          color: assignTeammateColor(leadAgentId),
          tmuxSessionName: '', tmuxPaneId: '',
          cwd: getCwd(), spawnedAt: Date.now(),
        },
      },
    },
  }))
}
```

**TeamFile 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 团队名称 |
| `description` | string? | 团队描述 |
| `createdAt` | number | 创建时间戳 |
| `leadAgentId` | string | Leader 的 Agent ID，格式 `team-lead@teamName` |
| `leadSessionId` | string | Leader 的 session ID |
| `members` | TeamMember[] | 成员列表（包括 Leader） |

成员结构包含 `agentId`、`name`、`agentType`、`model`、`joinedAt`、`tmuxPaneId`、`cwd`、`subscriptions`、`isActive`、`backendType` 等字段。

关键设计决策：

- **Leader 不设置 `CLAUDE_CODE_AGENT_ID` 环境变量**：注释明确说明，Leader 不是"Teammate"，`isTeammate()` 应该对 Leader 返回 `false`。设置此环境变量会错误地让 Leader 开始轮询收件箱。
- **`registerTeamForSessionCleanup`**：确保 session 结束时团队目录被清理，防止磁盘泄漏。
- **任务列表与团队绑定**：每个 Team = Project = TaskList，保证任务编号从 1 开始。

### 10.1.2 TeamDeleteTool

文件：`src/tools/TeamDeleteTool/TeamDeleteTool.ts`

```typescript
async call(_input, context) {
  const teamName = appState.teamContext?.teamName

  if (teamName) {
    const teamFile = readTeamFile(teamName)
    if (teamFile) {
      // 检查是否有活跃成员（排除 Leader）
      const nonLeadMembers = teamFile.members.filter(m => m.name !== TEAM_LEAD_NAME)
      const activeMembers = nonLeadMembers.filter(m => m.isActive !== false)

      if (activeMembers.length > 0) {
        return {
          data: {
            success: false,
            message: `Cannot cleanup team with ${activeMembers.length} active member(s): ${memberNames}. Use requestShutdown to gracefully terminate teammates first.`,
          },
        }
      }
    }

    await cleanupTeamDirectories(teamName)
    unregisterTeamForSessionCleanup(teamName)
    clearTeammateColors()
    clearLeaderTeamName()
  }

  setAppState(prev => ({
    ...prev,
    teamContext: undefined,
    inbox: { messages: [] },
  }))
}
```

删除前的安全检查：
1. 只检查**非 Leader** 成员
2. 区分"真正活跃"和"闲置/已退出"（`isActive === false`）
3. 如果有活跃成员，**拒绝删除**，要求先用 `requestShutdown` 优雅终止

---

## 10.2 SendMessage 消息路由

文件：`src/tools/SendMessageTool/SendMessageTool.ts`

SendMessageTool 是团队通信的核心枢纽。它支持 5 种消息路由路径，优先级从高到低：

### 10.2.1 路由一：Bridge（跨机器远程通信）

```typescript
if (feature('UDS_INBOX') && typeof input.message === 'string') {
  const addr = parseAddress(input.to)
  if (addr.scheme === 'bridge') {
    if (!getReplBridgeHandle() || !isReplBridgeActive()) {
      return { data: { success: false, message: 'Remote Control disconnected...' } }
    }
    const { postInterClaudeMessage } = require('../../bridge/peerSessions.js')
    const result = await postInterClaudeMessage(addr.target, input.message)
    return { data: { success: result.ok, message: ... } }
  }
}
```

发送到 `bridge:<session-id>` 格式的地址，通过 Anthropic 服务器中转到另一台机器上的 Claude 实例。需要 Remote Control 连接活跃。安全检查在 `checkPermissions` 中通过 `safetyCheck` 强制用户确认。

### 10.2.2 路由二：UDS Socket（本地跨进程通信）

```typescript
if (addr.scheme === 'uds') {
  const { sendToUdsSocket } = require('../../utils/udsClient.js')
  await sendToUdsSocket(addr.target, input.message)
  return { data: { success: true, message: `"${preview}" -> ${input.to}` } }
}
```

发送到 `uds:<socket-path>` 格式的地址，通过 Unix Domain Socket 通信。用于同一机器上不同 Claude 实例间的消息传递。

### 10.2.3 路由三：进程内 Subagent（直接内存路由）

```typescript
if (typeof input.message === 'string' && input.to !== '*') {
  const appState = context.getAppState()
  const registered = appState.agentNameRegistry.get(input.to)
  const agentId = registered ?? toAgentId(input.to)
  if (agentId) {
    const task = appState.tasks[agentId]
    if (isLocalAgentTask(task) && !isMainSessionTask(task)) {
      if (task.status === 'running') {
        // 运行中 -> 排队到 pendingMessages
        queuePendingMessage(agentId, input.message,
          context.setAppStateForTasks ?? context.setAppState)
        return { data: { success: true, message: 'Message queued...' } }
      }
      // 已停止 -> 自动恢复
      const result = await resumeAgentBackground({ agentId, prompt: input.message, ... })
      return { data: { success: true, message: `Agent "${input.to}" was stopped; resumed...` } }
    }
  }
}
```

优先检查 `agentNameRegistry`（注册名到 ID 的映射），然后尝试 `toAgentId` 格式匹配。三种子情况：
- **运行中**：通过 `queuePendingMessage()` 写入 `pendingMessages` 队列，在 Agent 的下一个 tool-round 边界被排空
- **已停止但 task 存在**：自动从 transcript 恢复并在后台运行
- **task 已被驱逐**：同样尝试从磁盘 transcript 恢复

### 10.2.4 路由四：广播（团队全员）

```typescript
if (input.to === '*') {
  return handleBroadcast(input.message, input.summary, context)
}
```

```typescript
async function handleBroadcast(content, summary, context) {
  const teamFile = await readTeamFileAsync(teamName)
  const recipients = teamFile.members
    .filter(m => m.name.toLowerCase() !== senderName.toLowerCase())
    .map(m => m.name)
  for (const recipientName of recipients) {
    await writeToMailbox(recipientName, { from: senderName, text: content, ... }, teamName)
  }
  return { data: { success: true, message: `Message broadcast to ${recipients.length} teammate(s)`, recipients } }
}
```

广播排除发送者自己，遍历所有成员写入各自的文件邮箱。

### 10.2.5 路由五：点对点（文件邮箱）

```typescript
return handleMessage(input.to, input.message, input.summary, context)
```

```typescript
async function handleMessage(recipientName, content, summary, context) {
  await writeToMailbox(recipientName, {
    from: senderName, text: content, summary,
    timestamp: new Date().toISOString(), color: senderColor,
  }, teamName)
  return { data: { success: true, message: `Message sent to ${recipientName}'s inbox` } }
}
```

最基础的路由：写入接收者的文件邮箱。邮箱路径基于团队名和成员名确定。

### 10.2.6 路由优先级总结

| 优先级 | 路由类型 | 条件 | 传输方式 |
|--------|----------|------|----------|
| 1 | Bridge | `bridge:` 前缀 | HTTP/Anthropic 服务器 |
| 2 | UDS Socket | `uds:` 前缀 | Unix Domain Socket |
| 3 | 进程内 Subagent | 名称匹配 agentNameRegistry 或 task | AppState 内存 |
| 4 | 广播 | `to === '*'` | 文件邮箱（遍历） |
| 5 | 点对点 | 其他 | 文件邮箱 |

---

## 10.3 结构化消息协议

### 10.3.1 消息类型定义

```typescript
const StructuredMessage = lazySchema(() =>
  z.discriminatedUnion('type', [
    z.object({
      type: z.literal('shutdown_request'),
      reason: z.string().optional(),
    }),
    z.object({
      type: z.literal('shutdown_response'),
      request_id: z.string(),
      approve: semanticBoolean(),
      reason: z.string().optional(),
    }),
    z.object({
      type: z.literal('plan_approval_response'),
      request_id: z.string(),
      approve: semanticBoolean(),
      feedback: z.string().optional(),
    }),
  ])
)
```

三种结构化消息：`shutdown_request`、`shutdown_response`、`plan_approval_response`。使用 `semanticBoolean()` 解析布尔值（支持 `"yes"` / `"true"` / `1` 等模糊表示）。

结构化消息的限制：
- 不能广播（`to: '*'`）
- 不能跨 session 发送（bridge / UDS）
- `shutdown_response` 只能发给 `team-lead`
- 拒绝 shutdown 时必须提供 `reason`

### 10.3.2 Shutdown 协议三步握手

**步骤一：Leader 发起 shutdown_request**

```typescript
async function handleShutdownRequest(targetName, reason, context) {
  const requestId = generateRequestId('shutdown', targetName)
  const shutdownMessage = createShutdownRequestMessage({ requestId, from: senderName, reason })
  await writeToMailbox(targetName, { from: senderName, text: jsonStringify(shutdownMessage), ... }, teamName)
  return { data: { success: true, request_id: requestId, target: targetName } }
}
```

生成唯一的 `requestId`，写入目标 Teammate 的邮箱。

**步骤二：Teammate 回应 shutdown_response (approve)**

```typescript
async function handleShutdownApproval(requestId, context) {
  // 获取自己的 pane ID 和 backend type
  const selfMember = teamFile.members.find(m => m.agentId === agentId)
  ownPaneId = selfMember.tmuxPaneId
  ownBackendType = selfMember.backendType

  // 构造审批消息
  const approvedMessage = createShutdownApprovedMessage({
    requestId, from: agentName, paneId: ownPaneId, backendType: ownBackendType,
  })
  await writeToMailbox(TEAM_LEAD_NAME, { from: agentName, text: jsonStringify(approvedMessage), ... }, teamName)

  // 根据后端类型执行退出
  if (ownBackendType === 'in-process') {
    // 进程内 Teammate：中止 AbortController
    const task = findTeammateTaskByAgentId(agentId, appState.tasks)
    task?.abortController?.abort()
  } else {
    // 进程外 Teammate：gracefulShutdown
    setImmediate(async () => { await gracefulShutdown(0, 'other') })
  }
}
```

区分两种退出路径：
- **进程内 Teammate**：通过 `abortController.abort()` 触发 Agent 循环退出
- **进程外 Teammate**（tmux/iTerm2）：调用 `gracefulShutdown()` 优雅退出整个进程

**步骤二（替代）：Teammate 拒绝 shutdown**

```typescript
async function handleShutdownRejection(requestId, reason) {
  const rejectedMessage = createShutdownRejectedMessage({ requestId, from: agentName, reason })
  await writeToMailbox(TEAM_LEAD_NAME, { ... }, teamName)
  return { data: { message: `Shutdown rejected. Reason: "${reason}". Continuing to work.` } }
}
```

拒绝后 Teammate 继续工作，Leader 收到通知可以决定后续操作。

**步骤三：Leader 收到审批确认后执行 TeamDelete**

Leader 确认所有 Teammate 已停止后，调用 `TeamDeleteTool` 清理团队。

### 10.3.3 Plan 审批协议

**Teammate 在 Plan 模式下完成计划后，Leader 审批：**

```typescript
async function handlePlanApproval(recipientName, requestId, context) {
  if (!isTeamLead(appState.teamContext)) {
    throw new Error('Only the team lead can approve plans.')
  }
  const leaderMode = appState.toolPermissionContext.mode
  const modeToInherit = leaderMode === 'plan' ? 'default' : leaderMode
  const approvalResponse = {
    type: 'plan_approval_response',
    requestId, approved: true,
    timestamp: new Date().toISOString(),
    permissionMode: modeToInherit,
  }
  await writeToMailbox(recipientName, { from: TEAM_LEAD_NAME, text: jsonStringify(approvalResponse), ... }, teamName)
}
```

审批时传递 `permissionMode`：
- 如果 Leader 处于 `plan` 模式，Teammate 继承 `default`（避免无限 plan 循环）
- 否则继承 Leader 当前模式

**拒绝审批**：

```typescript
async function handlePlanRejection(recipientName, requestId, feedback, context) {
  if (!isTeamLead(appState.teamContext)) {
    throw new Error('Only the team lead can reject plans.')
  }
  const rejectionResponse = {
    type: 'plan_approval_response', requestId,
    approved: false, feedback,
    timestamp: new Date().toISOString(),
  }
  await writeToMailbox(recipientName, { ... }, teamName)
}
```

拒绝时附带反馈，Teammate 根据反馈修改计划。

---

## 10.4 远程 Agent WebSocket 通信

### 10.4.1 SessionsWebSocket

文件：`src/remote/SessionsWebSocket.ts`

**连接协议**：

```
1. 连接到 wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe?organization_uuid=...
2. 通过 HTTP Header 传递 OAuth token：Authorization: Bearer <token>
3. 接收 SDKMessage 流
```

**常量配置**：

```typescript
const RECONNECT_DELAY_MS = 2000
const MAX_RECONNECT_ATTEMPTS = 5
const PING_INTERVAL_MS = 30000
const MAX_SESSION_NOT_FOUND_RETRIES = 3
const PERMANENT_CLOSE_CODES = new Set([4003]) // unauthorized
```

**双运行时支持**：

```typescript
if (typeof Bun !== 'undefined') {
  const ws = new globalThis.WebSocket(url, { headers, proxy: getWebSocketProxyUrl(url), tls: getWebSocketTLSOptions() })
  // Bun WebSocket event handlers...
} else {
  const { default: WS } = await import('ws')
  const ws = new WS(url, { headers, agent: getWebSocketProxyAgent(url), ...getWebSocketTLSOptions() })
  // Node.js ws event handlers...
}
```

同时支持 Bun 原生 WebSocket 和 Node.js `ws` 库。Bun 环境下使用 `globalThis.WebSocket`（支持 `proxy` 和 `tls` 选项），Node.js 环境下使用 `ws` 包。

**重连策略**：

```typescript
private handleClose(closeCode: number): void {
  // 永久关闭码（如 4003 unauthorized）-> 不重连
  if (PERMANENT_CLOSE_CODES.has(closeCode)) {
    this.callbacks.onClose?.()
    return
  }

  // 4001 session not found -> 有限重试（压缩期间可能暂时找不到）
  if (closeCode === 4001) {
    this.sessionNotFoundRetries++
    if (this.sessionNotFoundRetries > MAX_SESSION_NOT_FOUND_RETRIES) {
      this.callbacks.onClose?.()
      return
    }
    this.scheduleReconnect(RECONNECT_DELAY_MS * this.sessionNotFoundRetries, ...)
    return
  }

  // 普通断连 -> 5 次重试
  if (previousState === 'connected' && this.reconnectAttempts < MAX_RECONNECT_ATTEMPTS) {
    this.reconnectAttempts++
    this.scheduleReconnect(RECONNECT_DELAY_MS, ...)
  } else {
    this.callbacks.onClose?.()
  }
}
```

4001 错误码特殊处理的原因在注释中说明：compaction 期间服务器可能暂时认为 session 过期，短暂重试窗口允许客户端恢复。

**心跳机制**：

```typescript
private startPingInterval(): void {
  this.pingInterval = setInterval(() => {
    if (this.ws && this.state === 'connected') {
      this.ws.ping?.()
    }
  }, PING_INTERVAL_MS)  // 30秒
}
```

**消息发送**：

```typescript
sendControlResponse(response: SDKControlResponse): void {
  this.ws.send(jsonStringify(response))
}

sendControlRequest(request: SDKControlRequestInner): void {
  const controlRequest: SDKControlRequest = {
    type: 'control_request',
    request_id: randomUUID(),
    request,
  }
  this.ws.send(jsonStringify(controlRequest))
}
```

控制消息分为 `control_request`（客户端发起，如 interrupt）和 `control_response`（客户端回应服务端的权限请求）。

### 10.4.2 消息类型验证

```typescript
function isSessionsMessage(value: unknown): value is SessionsMessage {
  if (typeof value !== 'object' || value === null || !('type' in value)) {
    return false
  }
  return typeof value.type === 'string'
}
```

设计上接受**任何带 string `type` 字段的消息**，不使用硬编码的允许列表。原因：下游处理器决定如何处理未知类型，硬编码列表会在后端添加新消息类型时静默丢弃消息。

---

## 10.5 RemoteSessionManager 三通道协调

文件：`src/remote/RemoteSessionManager.ts`

### 10.5.1 架构概览

```typescript
export class RemoteSessionManager {
  private websocket: SessionsWebSocket | null = null
  private pendingPermissionRequests: Map<string, SDKControlPermissionRequest> = new Map()

  constructor(
    private readonly config: RemoteSessionConfig,
    private readonly callbacks: RemoteSessionCallbacks,
  ) {}
}
```

三个通道：
1. **WebSocket 接收通道**：订阅远程 session 的消息流
2. **HTTP POST 发送通道**：向远程 session 发送用户消息
3. **权限代理通道**：处理远程 Agent 的权限请求/响应

### 10.5.2 消息接收与分发

```typescript
private handleMessage(message): void {
  // 权限请求 -> 存入 pending 并通知回调
  if (message.type === 'control_request') {
    this.handleControlRequest(message)
    return
  }

  // 权限取消 -> 从 pending 移除并通知
  if (message.type === 'control_cancel_request') {
    this.pendingPermissionRequests.delete(request_id)
    this.callbacks.onPermissionCancelled?.(request_id, pendingRequest?.tool_use_id)
    return
  }

  // 控制响应（确认） -> 仅记录
  if (message.type === 'control_response') { return }

  // SDK 消息 -> 转发给回调
  if (isSDKMessage(message)) {
    this.callbacks.onMessage(message)
  }
}
```

四种消息处理路径清晰分离。权限请求存入 `pendingPermissionRequests` Map，以 `request_id` 为键，支持后续响应时查找。

### 10.5.3 权限代理流程

**远程 Agent 请求权限**：

```typescript
private handleControlRequest(request: SDKControlRequest): void {
  const { request_id, request: inner } = request
  if (inner.subtype === 'can_use_tool') {
    this.pendingPermissionRequests.set(request_id, inner)
    this.callbacks.onPermissionRequest(inner, request_id)
  } else {
    // 不认识的 subtype -> 返回错误响应，避免服务端无限等待
    const response: SDKControlResponse = {
      type: 'control_response',
      response: { subtype: 'error', request_id,
        error: `Unsupported control request subtype: ${inner.subtype}` },
    }
    this.websocket?.sendControlResponse(response)
  }
}
```

**本地用户响应权限**：

```typescript
respondToPermissionRequest(requestId: string, result: RemotePermissionResponse): void {
  const pendingRequest = this.pendingPermissionRequests.get(requestId)
  if (!pendingRequest) { logError(...); return }
  this.pendingPermissionRequests.delete(requestId)

  const response: SDKControlResponse = {
    type: 'control_response',
    response: {
      subtype: 'success', request_id: requestId,
      response: {
        behavior: result.behavior,
        ...(result.behavior === 'allow'
          ? { updatedInput: result.updatedInput }
          : { message: result.message }),
      },
    },
  }
  this.websocket?.sendControlResponse(response)
}
```

权限代理的完整流程：

```
远程 Agent 需要权限 -> CCR 服务器
    -> WebSocket control_request -> 本地 RemoteSessionManager
    -> onPermissionRequest 回调 -> 本地 UI 展示权限对话框
    -> 用户允许/拒绝
    -> respondToPermissionRequest -> WebSocket control_response
    -> CCR 服务器 -> 远程 Agent 继续/中止
```

### 10.5.4 消息发送

```typescript
async sendMessage(content: RemoteMessageContent, opts?): Promise<boolean> {
  const success = await sendEventToRemoteSession(this.config.sessionId, content, opts)
  return success
}
```

发送通过 HTTP POST API（`sendEventToRemoteSession`），而非 WebSocket。这是因为 WebSocket 是单向的（subscribe），发送使用独立的 HTTP 通道。

### 10.5.5 中断控制

```typescript
cancelSession(): void {
  this.websocket?.sendControlRequest({ subtype: 'interrupt' })
}
```

发送 `interrupt` 控制请求取消远程 session 的当前请求。

### 10.5.6 RemoteSessionConfig

```typescript
export type RemoteSessionConfig = {
  sessionId: string
  getAccessToken: () => string
  orgUuid: string
  hasInitialPrompt?: boolean
  viewerOnly?: boolean  // 纯查看模式：Ctrl+C 不发中断，无重连超时
}
```

`viewerOnly` 模式用于 `claude assistant` 命令，此模式下：
- Ctrl+C/Escape 不发送 interrupt
- 60 秒重连超时被禁用
- Session title 不更新

---

## 10.6 SDK 消息适配

文件：`src/remote/sdkMessageAdapter.ts`

### 10.6.1 消息转换

`sdkMessageAdapter` 将 CCR 后端的 SDK 格式消息转换为 REPL 内部的 Message 类型。

```typescript
export function convertSDKMessage(msg: SDKMessage, opts?): ConvertedMessage {
  switch (msg.type) {
    case 'assistant':
      return { type: 'message', message: convertAssistantMessage(msg) }
    case 'user':
      // 工具结果消息需要特殊处理
      // 用户文本消息根据模式决定是否转换
      return ...
    case 'stream_event':
      return { type: 'stream_event', event: convertStreamEvent(msg) }
    case 'result':
      // 只转换错误结果，成功结果是噪音
      if (msg.subtype !== 'success') {
        return { type: 'message', message: convertResultMessage(msg) }
      }
      return { type: 'ignored' }
    case 'system':
      // init / status / compact_boundary 各有处理
      // hook_response 等忽略
      return ...
  }
}
```

**转换类型对照表**：

| SDKMessage 类型 | 转换结果 | 说明 |
|----------------|----------|------|
| `assistant` | `AssistantMessage` | 直接映射 |
| `stream_event` | `StreamEvent` | 流式增量 |
| `user` (tool_result) | `UserMessage` | 仅 `convertToolResults` 模式 |
| `user` (text) | `UserMessage` | 仅 `convertUserTextMessages` 模式 |
| `result` (error) | `SystemMessage` | 错误结果 |
| `result` (success) | ignored | 成功结果是噪音 |
| `system` (init) | `SystemMessage` | 远程 session 初始化 |
| `system` (status) | `SystemMessage` | 如 "compacting" |
| `system` (compact_boundary) | `SystemMessage` | 对话压缩标记 |
| `tool_progress` | `SystemMessage` | 工具进度（简化版） |

**转换选项**：

```typescript
type ConvertOptions = {
  convertToolResults?: boolean       // direct connect 模式需要
  convertUserTextMessages?: boolean  // 历史事件回放需要
}
```

- `convertToolResults`：Direct connect 模式下，工具结果来自远程服务器，需要转换为本地消息以正确渲染和折叠
- `convertUserTextMessages`：历史事件转换时需要，因为用户消息没有被本地 REPL 添加过

### 10.6.2 Assistant 消息转换

```typescript
function convertAssistantMessage(msg: SDKAssistantMessage): AssistantMessage {
  return {
    type: 'assistant',
    message: msg.message,
    uuid: msg.uuid,
    requestId: undefined,
    timestamp: new Date().toISOString(),
    error: msg.error,
  }
}
```

`requestId` 设为 `undefined`，因为远程消息没有本地 request 上下文。

### 10.6.3 Stream Event 转换

```typescript
function convertStreamEvent(msg: SDKPartialAssistantMessage): StreamEvent {
  return { type: 'stream_event', event: msg.event }
}
```

直接透传事件，不添加额外元数据。

---

## 10.7 进程内 Teammate 的 Spawn 与运行循环

### 10.7.1 Spawn 流程

文件：`src/utils/swarm/spawnInProcess.ts`

```typescript
export async function spawnInProcessTeammate(config, context): Promise<InProcessSpawnOutput> {
  const { name, teamName, prompt, color, planModeRequired, model } = config

  // 1. 生成确定性 Agent ID
  const agentId = formatAgentId(name, teamName)  // "name@team"
  const taskId = generateTaskId('in_process_teammate')  // "t" + 8位随机

  // 2. 创建独立的 AbortController
  const abortController = createAbortController()

  // 3. 创建 TeammateIdentity（存储在 AppState 的纯数据）
  const identity: TeammateIdentity = {
    agentId, agentName: name, teamName, color,
    planModeRequired, parentSessionId: getSessionId(),
  }

  // 4. 创建 TeammateContext（用于 AsyncLocalStorage）
  const teammateContext = createTeammateContext({
    agentId, agentName: name, teamName, color,
    planModeRequired, parentSessionId, abortController,
  })

  // 5. 注册 Perfetto 追踪
  if (isPerfettoTracingEnabled()) {
    registerPerfettoAgent(agentId, name, parentSessionId)
  }

  // 6. 构造任务状态
  const taskState: InProcessTeammateTaskState = {
    ...createTaskStateBase(taskId, 'in_process_teammate', description, context.toolUseId),
    type: 'in_process_teammate',
    status: 'running',
    identity, prompt, model, abortController,
    awaitingPlanApproval: false,
    spinnerVerb: sample(getSpinnerVerbs()),  // 随机选择加载动词
    pastTenseVerb: sample(TURN_COMPLETION_VERBS),
    permissionMode: planModeRequired ? 'plan' : 'default',
    isIdle: false, shutdownRequested: false,
    lastReportedToolCount: 0, lastReportedTokenCount: 0,
    pendingUserMessages: [],
    messages: [],
  }

  // 7. 注册清理回调
  const unregisterCleanup = registerCleanup(async () => {
    abortController.abort()
  })
  taskState.unregisterCleanup = unregisterCleanup

  // 8. 写入 AppState
  registerTask(taskState, setAppState)

  return { success: true, agentId, taskId, abortController, teammateContext }
}
```

`TeammateIdentity` 和 `TeammateContext` 的区别：
- `TeammateIdentity`：纯数据，存储在 AppState 中，用于 UI 展示和序列化
- `TeammateContext`：运行时对象，存储在 AsyncLocalStorage 中，用于上下文隔离

### 10.7.2 Kill 流程

```typescript
export function killInProcessTeammate(taskId, setAppState): boolean {
  let killed = false
  setAppState((prev: AppState) => {
    const task = prev.tasks[taskId]
    if (!task || task.type !== 'in_process_teammate') return prev
    const teammateTask = task as InProcessTeammateTaskState
    if (teammateTask.status !== 'running') return prev

    killed = true
    teamName = teammateTask.identity.teamName
    agentId = teammateTask.identity.agentId

    teammateTask.abortController?.abort()
    teammateTask.unregisterCleanup?.()

    return {
      ...prev,
      tasks: { ...prev.tasks, [taskId]: {
        ...teammateTask,
        status: 'killed', endTime: Date.now(),
        abortController: undefined, unregisterCleanup: undefined,
        currentWorkAbortController: undefined, selectedAgent: undefined,
      }}
    }
  })

  if (killed) {
    void evictTaskOutput(taskId)
    unregisterPerfettoAgent(agentId)
    emitTaskTerminatedSdk(taskId, toolUseId, description)
    // 延迟驱逐给 UI 时间展示终态
    setTimeout(() => evictTerminalTask(taskId, setAppState), STOPPED_DISPLAY_MS)
    // 从团队文件中移除成员
    void removeMemberByAgentId(teamName, agentId)
  }
}
```

Kill 后的 3 秒展示期（`STOPPED_DISPLAY_MS`），让 UI 有时间显示终态状态后再从 AppState 中驱逐。

### 10.7.3 运行循环

文件：`src/utils/swarm/inProcessRunner.ts`

`runInProcessTeammate()` 是进程内 Teammate 的核心运行循环：

```typescript
export async function runInProcessTeammate(config): Promise<InProcessRunnerResult> {
  // 构建系统提示（支持 replace/append/default 模式）
  let teammateSystemPrompt: string
  if (systemPromptMode === 'replace' && systemPrompt) {
    teammateSystemPrompt = systemPrompt
  } else {
    const systemPromptParts = [...fullSystemPromptParts, TEAMMATE_SYSTEM_PROMPT_ADDENDUM]
    if (agentDefinition) {
      systemPromptParts.push(`\n# Custom Agent Instructions\n${customPrompt}`)
    }
    if (systemPromptMode === 'append' && systemPrompt) {
      systemPromptParts.push(systemPrompt)
    }
    teammateSystemPrompt = systemPromptParts.join('\n')
  }

  // 确保团队工具始终可用
  const resolvedAgentDefinition = {
    agentType: identity.agentName,
    tools: agentDefinition?.tools
      ? [...new Set([...agentDefinition.tools,
          SEND_MESSAGE_TOOL_NAME, TEAM_CREATE_TOOL_NAME,
          TEAM_DELETE_TOOL_NAME, TASK_CREATE_TOOL_NAME,
          TASK_GET_TOOL_NAME, TASK_LIST_TOOL_NAME,
          TASK_UPDATE_TOOL_NAME,
        ])]
      : ['*'],
    permissionMode: 'default',
  }

  const allMessages: Message[] = []
  let currentPrompt = wrappedInitialPrompt
  let shouldExit = false

  // 主循环
  while (!abortController.signal.aborted && !shouldExit) {
    // 1. 创建每轮次的 AbortController
    const currentWorkAbortController = createAbortController()
    updateTaskState(taskId, task => ({ ...task, currentWorkAbortController }), setAppState)

    // 2. 准备上下文（可能需要 compaction）
    let contextMessages = allMessages
    const tokenCount = tokenCountWithEstimation(allMessages)
    if (tokenCount > getAutoCompactThreshold(model)) {
      const compactedSummary = await compactConversation(allMessages, isolatedContext, ...)
      contextMessages = buildPostCompactMessages(compactedSummary, allMessages)
      resetMicrocompactState(allMessages)
    }

    // 3. 在 TeammateContext 中运行 Agent
    await runWithTeammateContext(teammateContext, async () => {
      await runWithAgentContext(agentContext, async () => {
        for await (const message of runAgent({
          agentDefinition: resolvedAgentDefinition,
          promptMessages, toolUseContext, canUseTool,
          isAsync: true,
          canShowPermissionPrompts: allowPermissionPrompts ?? true,
          forkContextMessages: contextMessages,
          querySource: 'agent:teammate',
          override: {
            abortController: currentWorkAbortController,
            agentId: toAgentId(identity.agentId),
            systemPrompt: asSystemPrompt(teammateSystemPrompt),
          },
          preserveToolUseResults: true,
          availableTools, allowedTools,
          contentReplacementState: teammateReplacementState,
        })) {
          allMessages.push(message)
          // 更新进度和 UI 消息
          if (message.type === 'assistant') {
            updateProgressFromMessage(tracker, message, ...)
            updateTaskState(taskId, task => ({
              ...task,
              progress: getProgressUpdate(tracker),
              messages: appendCappedMessage(task.messages, message),
            }), setAppState)
          }
        }
      })
    })

    // 4. 设为 idle 状态
    updateTaskState(taskId, task => ({ ...task, isIdle: true }), setAppState)

    // 5. 发送 idle 通知到 Leader 邮箱
    await sendIdleNotification(identity.agentName, identity.color, identity.teamName, ...)

    // 6. 等待新消息或 shutdown
    const waitResult = await waitForNextPromptOrShutdown(
      identity, abortController, taskId, getAppState, setAppState, taskListId
    )

    switch (waitResult.type) {
      case 'shutdown_request':
        // 注入 shutdown 请求到对话，让模型决定
        currentPrompt = formatAsTeammateMessage(waitResult.request.from, waitResult.originalMessage)
        updateTaskState(taskId, task => ({ ...task, isIdle: false, shutdownRequested: true }), setAppState)
        break
      case 'new_message':
        currentPrompt = formatAsTeammateMessage(waitResult.from, waitResult.message, ...)
        updateTaskState(taskId, task => ({ ...task, isIdle: false }), setAppState)
        break
      case 'aborted':
        shouldExit = true
        break
    }
  }
}
```

### 10.7.4 等待循环的消息优先级

```typescript
async function waitForNextPromptOrShutdown(identity, abortController, taskId, getAppState, setAppState, taskListId): Promise<WaitResult> {
  while (!abortController.signal.aborted) {
    // 优先级 1: 进程内 pendingUserMessages（用户在 transcript 视图中发送的消息）
    const task = getAppState().tasks[taskId]
    if (task?.pendingUserMessages.length > 0) {
      return { type: 'new_message', message: task.pendingUserMessages[0] }
    }

    // 优先级 2: 邮箱中的 shutdown 请求
    const allMessages = await readMailbox(identity.agentName, identity.teamName)
    for (const m of allMessages) {
      if (!m.read) {
        const parsed = isShutdownRequest(m.text)
        if (parsed) {
          return { type: 'shutdown_request', request: parsed, originalMessage: m.text }
        }
      }
    }

    // 优先级 3: Leader 的邮箱消息
    for (const m of allMessages) {
      if (!m.read && m.from === TEAM_LEAD_NAME) {
        return { type: 'new_message', message: m.text, from: m.from }
      }
    }

    // 优先级 4: 任何其他未读邮箱消息
    const firstUnread = allMessages.find(m => !m.read)
    if (firstUnread) {
      return { type: 'new_message', message: firstUnread.text, from: firstUnread.from }
    }

    // 优先级 5: 任务列表中的待认领任务
    const taskPrompt = await tryClaimNextTask(taskListId, identity.agentName)
    if (taskPrompt) {
      return { type: 'new_message', message: taskPrompt, from: 'task-list' }
    }

    await sleep(500)
  }
  return { type: 'aborted' }
}
```

消息优先级设计原则：
1. **内存中的用户消息最优先**：直接的用户交互不应该等待
2. **Shutdown 请求优先于普通消息**：防止消息洪水阻塞 shutdown
3. **Leader 消息优先于 Peer 消息**：Leader 代表用户意图和协调指令
4. **Peer 消息 FIFO**：同等级消息按到达顺序处理
5. **任务列表作为兜底**：无消息时主动认领待完成任务

### 10.7.5 权限代理（进程内 Teammate）

```typescript
function createInProcessCanUseTool(identity, abortController, onPermissionWaitMs?): CanUseToolFn {
  return async (tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision) => {
    const result = forceDecision ?? await hasPermissionsToUseTool(...)
    if (result.behavior !== 'ask') { return result }

    // Bash 命令先尝试分类器自动审批
    if (feature('BASH_CLASSIFIER') && tool.name === BASH_TOOL_NAME && result.pendingClassifierCheck) {
      const classifierDecision = await awaitClassifierAutoApproval(...)
      if (classifierDecision) { return { behavior: 'allow', ... } }
    }

    // 优先路径：使用 Leader 的 ToolUseConfirm 对话框
    const setToolUseConfirmQueue = getLeaderToolUseConfirmQueue()
    if (setToolUseConfirmQueue) {
      return new Promise(resolve => {
        setToolUseConfirmQueue(queue => [...queue, {
          tool, description, input, toolUseContext, toolUseID,
          permissionResult: result,
          workerBadge: identity.color ? { name: identity.agentName, color: identity.color } : undefined,
          onAllow(...) { resolve({ behavior: 'allow', ... }) },
          onReject(feedback?) { resolve({ behavior: 'ask', message: ... }) },
          recheckPermission() { ... },
        }])
      })
    }

    // 降级路径：通过邮箱系统
    return new Promise(resolve => {
      const request = createPermissionRequest({
        toolName, toolUseId, input, description,
        workerId: identity.agentId, workerName: identity.agentName,
        workerColor: identity.color, teamName: identity.teamName,
      })
      registerPermissionCallback({ requestId: request.id, onAllow, onReject })
      void sendPermissionRequestViaMailbox(request)
      // 轮询邮箱等待响应...
    })
  }
}
```

两条权限路径：
- **Leader UI 队列**（优先）：直接复用 Leader 的 `ToolUseConfirm` 对话框，显示 worker badge（名称+颜色），用户体验与 Leader 自己的工具权限提示一致
- **邮箱系统**（降级）：当 Leader UI 不可用时，通过文件邮箱异步交换权限请求和响应

权限允许后，更新会写回 Leader 的共享上下文，但**保留 Leader 的模式**（`preserveMode: true`），防止 Worker 的权限上下文泄漏到 Coordinator。

---

## 10.8 消息格式化

### 10.8.1 XML Teammate Message

```typescript
function formatAsTeammateMessage(from, content, color?, summary?): string {
  const colorAttr = color ? ` color="${color}"` : ''
  const summaryAttr = summary ? ` summary="${summary}"` : ''
  return `<${TEAMMATE_MESSAGE_TAG} teammate_id="${from}"${colorAttr}${summaryAttr}>\n${content}\n</${TEAMMATE_MESSAGE_TAG}>`
}
```

统一的 XML 格式确保模型看到的消息结构一致，无论来源是 tmux Teammate、进程内 Teammate 还是文件邮箱。

### 10.8.2 Idle 通知

```typescript
async function sendIdleNotification(agentName, agentColor, teamName, options?) {
  const notification = createIdleNotification(agentName, options)
  await sendMessageToLeader(agentName, jsonStringify(notification), agentColor, teamName)
}
```

idle 通知包含可选字段：
- `idleReason`：`'available'` / `'interrupted'` / `'failed'`
- `summary`：完成的工作摘要
- `completedTaskId`：完成的任务 ID
- `completedStatus`：`'resolved'` / `'blocked'` / `'failed'`
- `failureReason`：失败原因

---

## 10.9 RemoteAgentTask

文件：`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`

### 10.9.1 远程任务类型

```typescript
const REMOTE_TASK_TYPES = ['remote-agent', 'ultraplan', 'ultrareview', 'autofix-pr', 'background-pr'] as const
export type RemoteTaskType = (typeof REMOTE_TASK_TYPES)[number]
```

五种远程任务类型，各有不同的完成检测和通知逻辑。

### 10.9.2 状态类型

```typescript
export type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  remoteTaskType: RemoteTaskType
  remoteTaskMetadata?: RemoteTaskMetadata
  sessionId: string
  command: string
  title: string
  todoList: TodoList
  log: SDKMessage[]
  isLongRunning?: boolean
  pollStartedAt: number
  isRemoteReview?: boolean
  reviewProgress?: { stage?, bugsFound, bugsVerified, bugsRefuted }
  isUltraplan?: boolean
  ultraplanPhase?: Exclude<UltraplanPhase, 'running'>
}
```

`pollStartedAt` 特殊处理：记录的是轮询**开始**时间而非任务创建时间。这样 `--resume` 恢复的会话不会因为"30 分钟前创建"而立即超时。

### 10.9.3 完成检测器

```typescript
const completionCheckers = new Map<RemoteTaskType, RemoteTaskCompletionChecker>()

export function registerCompletionChecker(
  remoteTaskType: RemoteTaskType,
  checker: RemoteTaskCompletionChecker
): void {
  completionCheckers.set(remoteTaskType, checker)
}
```

插件式设计：不同的远程任务类型可以注册自己的完成检测器。检测器在每次轮询时被调用，返回非 null 字符串表示任务完成。

### 10.9.4 元数据持久化

```typescript
async function persistRemoteAgentMetadata(meta: RemoteAgentMetadata): Promise<void> {
  await writeRemoteAgentMetadata(meta.taskId, meta)
}

async function removeRemoteAgentMetadata(taskId: string): Promise<void> {
  await deleteRemoteAgentMetadata(taskId)
}
```

元数据写入 session sidecar，确保 `--resume` 可以恢复。任务完成/kill 时删除，防止恢复时复活已完成的任务。

### 10.9.5 Review 内容提取

```typescript
function extractReviewFromLog(log: SDKMessage[]): string | null {
  // 路径一：hook_progress / hook_response（bughunter 模式）
  for (let i = log.length - 1; i >= 0; i--) {
    const msg = log[i]
    if (msg?.type === 'system' && (msg.subtype === 'hook_progress' || msg.subtype === 'hook_response')) {
      const tagged = extractTag(msg.stdout, REMOTE_REVIEW_TAG)
      if (tagged?.trim()) return tagged.trim()
    }
  }

  // 路径二：assistant 消息（prompt 模式）
  for (let i = log.length - 1; i >= 0; i--) {
    const msg = log[i]
    if (msg?.type !== 'assistant') continue
    const tagged = extractTag(fullText, REMOTE_REVIEW_TAG)
    if (tagged?.trim()) return tagged.trim()
  }

  // 路径三：Hook stdout 拼接（大 JSON 跨事件分割）
  const hookStdout = log.filter(...).map(msg => msg.stdout).join('')
  const hookTagged = extractTag(hookStdout, REMOTE_REVIEW_TAG)
  if (hookTagged?.trim()) return hookTagged.trim()

  // 降级：拼接所有 assistant 文本
  const allText = log.filter(...).map(...).join('\n').trim()
  return allText || null
}
```

四级提取策略，从最精确到最宽松：
1. 单个 hook 事件中的 `<remote-review>` 标签
2. 单个 assistant 消息中的标签
3. 跨 hook 事件拼接后的标签（管道缓冲区分割场景）
4. 所有 assistant 文本拼接（兜底）

---

## 10.10 小结

Claude Code 的团队协作架构展现了几个核心设计原则：

1. **多层消息路由**：从高性能的进程内内存传递，到跨机器的 Bridge 通信，统一由 SendMessageTool 分发。每一层都有明确的适用场景和降级路径。

2. **协议化通信**：Shutdown 三步握手、Plan 审批协议通过结构化 JSON 消息实现，避免自然语言解析的歧义性。

3. **权限安全**：进程内 Teammate 的权限请求优先通过 Leader 的 UI 队列展示，降级到邮箱系统。跨机器消息需要 safetyCheck 级别的用户确认。

4. **容错设计**：WebSocket 的分级重连策略（永久错误/瞬时错误/session-not-found 各有不同处理），远程任务的 pollStartedAt 机制防止恢复后误超时。

5. **上下文隔离**：AsyncLocalStorage 为进程内 Teammate 提供身份隔离，独立的 AbortController 和 permissionMode 确保 Teammate 之间互不干扰。

6. **内存安全**：50 条消息 UI 上限、任务驱逐机制、定时清理回调，在支持数百并发 Agent 的同时控制内存增长。
