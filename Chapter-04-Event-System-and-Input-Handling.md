# 第4章 事件系统与输入处理

Claude Code 实现了一套完整的终端事件系统，从原始字节流解析到 W3C 风格的捕获/冒泡分发，再到分层键绑定解析和 Vim 模式状态机。本章按信号流方向——从底层到高层——逐一剖析。

## 4.1 架构总览

输入信号的完整路径：

```
stdin 字节流
  → termio/tokenize.ts (转义序列边界检测)
  → parse-keypress.ts (语义解析: ParsedKey / ParsedMouse / ParsedResponse)
  → input-event.ts (EventEmitter 旧路径: InputEvent → useInput)
  → keyboard-event.ts (DOM 新路径: KeyboardEvent → Dispatcher → 捕获/冒泡)
  → keybindings/ (分层键绑定解析: context + chord)
  → vim/ (可选: Vim 模式纯函数状态机)
```

涉及的核心源码文件：

| 文件 | 职责 |
|------|------|
| `src/ink/parse-keypress.ts` | 终端原始输入解析 |
| `src/ink/events/terminal-event.ts` | 事件基类 (W3C 风格) |
| `src/ink/events/keyboard-event.ts` | 键盘事件 |
| `src/ink/events/focus-event.ts` | 焦点事件 |
| `src/ink/events/event-handlers.ts` | 事件处理器 props 映射 |
| `src/ink/events/dispatcher.ts` | 捕获/冒泡分发器 |
| `src/ink/events/input-event.ts` | 旧路径 InputEvent |
| `src/ink/focus.ts` | 焦点管理 |
| `src/ink/hooks/use-input.ts` | useInput hook |
| `src/keybindings/parser.ts` | 键绑定配置解析 |
| `src/keybindings/match.ts` | 按键匹配 |
| `src/keybindings/resolver.ts` | 键绑定解析 + chord 状态机 |
| `src/keybindings/defaultBindings.ts` | 默认键绑定配置 |
| `src/vim/types.ts` | Vim 状态机类型定义 |
| `src/vim/transitions.ts` | Vim 状态转移函数 |
| `src/vim/motions.ts` | Vim 光标移动函数 |
| `src/vim/textObjects.ts` | Vim 文本对象查找 |

## 4.2 终端原始输入解析

### 4.2.1 输入来源与分词

终端输入是一个字节流，不同类型的输入（普通字符、转义序列、鼠标事件、终端响应）混在一起。`parse-keypress.ts` 负责将这个字节流解析为结构化的输入事件。

顶层入口函数：

```typescript
export function parseMultipleKeypresses(
  prevState: KeyParseState,
  input: Buffer | string | null = '',
): [ParsedInput[], KeyParseState] {
  const isFlush = input === null
  const inputString = isFlush ? '' : inputToString(input)

  const tokenizer = prevState._tokenizer ?? createTokenizer({ x10Mouse: true })
  const tokens = isFlush ? tokenizer.flush() : tokenizer.feed(inputString)

  const keys: ParsedInput[] = []
  let inPaste = prevState.mode === 'IN_PASTE'
  let pasteBuffer = prevState.pasteBuffer

  for (const token of tokens) {
    if (token.type === 'sequence') {
      if (token.value === PASTE_START) {
        inPaste = true
        pasteBuffer = ''
      } else if (token.value === PASTE_END) {
        keys.push(createPasteKey(pasteBuffer))
        inPaste = false
        pasteBuffer = ''
      } else if (inPaste) {
        pasteBuffer += token.value
      } else {
        const response = parseTerminalResponse(token.value)
        if (response) {
          keys.push({ kind: 'response', sequence: token.value, response })
        } else {
          const mouse = parseMouseEvent(token.value)
          if (mouse) {
            keys.push(mouse)
          } else {
            keys.push(parseKeypress(token.value))
          }
        }
      }
    } else if (token.type === 'text') {
      if (inPaste) {
        pasteBuffer += token.value
      } else {
        keys.push(parseKeypress(token.value))
      }
    }
  }
  // ...
}
```

解析流程分为三个层级：

1. **Tokenizer 层**：`termio/tokenize.ts` 中的状态机识别转义序列的边界，输出 `sequence` 和 `text` 两类 token。
2. **分类层**：对 `sequence` token 按优先级尝试：粘贴模式标记 → 终端响应 → 鼠标事件 → 键盘输入。
3. **语义层**：`parseKeypress` 将原始序列映射为 `ParsedKey` 结构。

状态在调用间通过 `KeyParseState` 保持：

```typescript
export type KeyParseState = {
  mode: 'NORMAL' | 'IN_PASTE'
  incomplete: string
  pasteBuffer: string
  _tokenizer?: Tokenizer
}
```

Tokenizer 实例在 state 中持久化，跨 `stdin` 读取保持转义序列的分割状态。

### 4.2.2 CSI 序列解析

传统终端序列（VT100/xterm）通过正则匹配：

```typescript
const FN_KEY_RE =
  /^(?:\x1b+)(O|N|\[|\[\[)(?:(\d+)(?:;(\d+))?([~^$])|(?:1;)?(\d+)?([a-zA-Z]))/
```

这个正则覆盖多种前缀（`ESC O`、`ESC [`、`ESC [[`）和多种终止符（`~`、`^`、`$`、字母），并提取参数和修饰符。

键名通过查表映射：

```typescript
const keyName: Record<string, string> = {
  '[A': 'up',
  '[B': 'down',
  '[C': 'right',
  '[D': 'left',
  '[15~': 'f5',
  '[17~': 'f6',
  // ... 60+ 条目
}
```

### 4.2.3 Kitty 键盘协议

Kitty 协议使用 `CSI u` 格式，可以精确表达修饰键组合：

```typescript
const CSI_U_RE = /^\x1b\[(\d+)(?:;(\d+))?u/

if ((match = CSI_U_RE.exec(s))) {
  const codepoint = parseInt(match[1]!, 10)
  const modifier = match[2] ? parseInt(match[2], 10) : 1
  const mods = decodeModifier(modifier)
  const name = keycodeToName(codepoint)
  return {
    kind: 'key',
    name,
    ctrl: mods.ctrl,
    meta: mods.meta,
    shift: mods.shift,
    super: mods.super,
    // ...
  }
}
```

修饰符编码规则：`modifier = 1 + shift*1 + alt*2 + ctrl*4 + super*8`

```typescript
function decodeModifier(modifier: number): {
  shift: boolean; meta: boolean; ctrl: boolean; super: boolean
} {
  const m = modifier - 1
  return {
    shift: !!(m & 1),
    meta: !!(m & 2),
    ctrl: !!(m & 4),
    super: !!(m & 8),
  }
}
```

Kitty 协议的关键优势是能区分 `super`（Cmd/Win）和 `meta`（Alt/Option）。传统终端中 Alt 键用 ESC 前缀表示，和真正的 ESC 键不可区分。

### 4.2.4 xterm modifyOtherKeys

另一种扩展协议，参数顺序与 CSI u 相反：

```typescript
const MODIFY_OTHER_KEYS_RE = /^\x1b\[27;(\d+);(\d+)~/

if ((match = MODIFY_OTHER_KEYS_RE.exec(s))) {
  const mods = decodeModifier(parseInt(match[1]!, 10))  // modifier 在前
  const name = keycodeToName(parseInt(match[2]!, 10))    // keycode 在后
  // ...
}
```

Ghostty、tmux 在 SSH 环境下可能使用此协议，因为 `TERM` 嗅探无法识别 Ghostty，不会推送 Kitty 模式。

### 4.2.5 鼠标事件解析

SGR 鼠标格式：`CSI < button ; col ; row M/m`

```typescript
const SGR_MOUSE_RE = /^\x1b\[<(\d+);(\d+);(\d+)([Mm])$/

function parseMouseEvent(s: string): ParsedMouse | null {
  const match = SGR_MOUSE_RE.exec(s)
  if (!match) return null
  const button = parseInt(match[1]!, 10)
  if ((button & 0x40) !== 0) return null  // 滚轮事件走 ParsedKey
  return {
    kind: 'mouse',
    button,
    action: match[4] === 'M' ? 'press' : 'release',
    col: parseInt(match[2]!, 10),  // 1-indexed
    row: parseInt(match[3]!, 10),  // 1-indexed
    sequence: s,
  }
}
```

按钮编码：bit 6 (`0x40`) = 滚轮，bit 5 (`0x20`) = 拖拽/移动。滚轮事件故意不走 `ParsedMouse`，而是作为 `ParsedKey`（`wheelup`/`wheeldown`）进入键绑定系统，这样滚轮可以绑定到滚动动作。

X10 传统格式也被支持：`CSI M Cb Cx Cy`（3 个原始字节）：

```typescript
if (s.length === 6 && s.startsWith('\x1b[M')) {
  const button = s.charCodeAt(3) - 32
  if ((button & 0x43) === 0x40) return createNavKey(s, 'wheelup', false)
  if ((button & 0x43) === 0x41) return createNavKey(s, 'wheeldown', false)
}
```

### 4.2.6 终端响应解析

终端对查询的回复也在输入流中：

```typescript
function parseTerminalResponse(s: string): TerminalResponse | null {
  if (s.startsWith('\x1b[')) {
    if ((m = DECRPM_RE.exec(s)))     return { type: 'decrpm', mode, status }
    if ((m = DA1_RE.exec(s)))        return { type: 'da1', params }
    if ((m = DA2_RE.exec(s)))        return { type: 'da2', params }
    if ((m = KITTY_FLAGS_RE.exec(s))) return { type: 'kittyKeyboard', flags }
    if ((m = CURSOR_POSITION_RE.exec(s))) return { type: 'cursorPosition', row, col }
  }
  if (s.startsWith('\x1b]'))  // OSC 响应 (如背景色查询)
  if (s.startsWith('\x1bP'))  // DCS 响应 (如 XTVERSION)
}
```

响应类型覆盖：DECRPM（模式查询）、DA1/DA2（设备属性）、Kitty 键盘标志、光标位置、OSC 响应、XTVERSION。这些响应通过语法特征（`CSI ?` 前缀等）与键盘输入区分，不会误判为按键。

### 4.2.7 ParsedInput 联合类型

解析器的输出是三种类型的联合：

```typescript
export type ParsedInput = ParsedKey | ParsedMouse | ParsedResponse
```

每种类型用 `kind` 字段区分：`'key'`、`'mouse'`、`'response'`。下游代码通过 discriminated union 处理不同类型。

## 4.3 双事件路径

Claude Code 的事件系统存在两条并行路径：

### 4.3.1 EventEmitter 旧路径

这是从原始 Ink 继承的路径。`input-event.ts` 定义了 `InputEvent` 类，它从 `ParsedKey` 构造出标准化的 `Key` 对象和 `input` 字符串：

```typescript
export class InputEvent extends Event {
  readonly keypress: ParsedKey
  readonly key: Key
  readonly input: string

  constructor(keypress: ParsedKey) {
    super()
    const [key, input] = parseKey(keypress)
    this.keypress = keypress
    this.key = key
    this.input = input
  }
}
```

`parseKey` 函数处理大量边界情况：

```typescript
function parseKey(keypress: ParsedKey): [Key, string] {
  const key: Key = {
    upArrow: keypress.name === 'up',
    downArrow: keypress.name === 'down',
    return: keypress.name === 'return',
    escape: keypress.name === 'escape',
    ctrl: keypress.ctrl,
    shift: keypress.shift,
    meta: keypress.meta || keypress.name === 'escape' || keypress.option,
    super: keypress.super,
    // ...
  }

  let input = keypress.ctrl ? keypress.name : keypress.sequence
  // ...
}
```

注意 `meta` 字段的特殊处理：`keypress.name === 'escape'` 也被设为 `meta: true`。这是终端的历史遗留——ESC 键就是 Meta 前缀。后续的键绑定匹配必须特殊处理这一点。

`useInput` hook 通过 EventEmitter 监听这些事件：

```typescript
const useInput = (inputHandler: Handler, options: Options = {}) => {
  const { setRawMode, internal_eventEmitter } = useStdin()

  useLayoutEffect(() => {
    if (options.isActive === false) return
    setRawMode(true)
    return () => { setRawMode(false) }
  }, [options.isActive, setRawMode])

  const handleData = useEventCallback((event: InputEvent) => {
    if (options.isActive === false) return
    const { input, key } = event
    if (!(input === 'c' && key.ctrl) || !internal_exitOnCtrlC) {
      inputHandler(input, key, event)
    }
  })

  useEffect(() => {
    internal_eventEmitter?.on('input', handleData)
    return () => { internal_eventEmitter?.off('input', handleData) }
  }, [internal_eventEmitter, handleData])
}
```

旧路径的特点：扁平的监听器列表，没有事件冒泡，通过 `stopImmediatePropagation` 阻止同级监听器。

### 4.3.2 DOM 新路径

新路径实现了完整的 W3C 事件模型。入口是 `KeyboardEvent`：

```typescript
export class KeyboardEvent extends TerminalEvent {
  readonly key: string
  readonly ctrl: boolean
  readonly shift: boolean
  readonly meta: boolean
  readonly superKey: boolean
  readonly fn: boolean

  constructor(parsedKey: ParsedKey) {
    super('keydown', { bubbles: true, cancelable: true })
    this.key = keyFromParsed(parsedKey)
    this.ctrl = parsedKey.ctrl
    this.shift = parsedKey.shift
    this.meta = parsedKey.meta || parsedKey.option
    this.superKey = parsedKey.super
    this.fn = parsedKey.fn
  }
}
```

`keyFromParsed` 遵循浏览器 `KeyboardEvent.key` 的语义：

```typescript
function keyFromParsed(parsed: ParsedKey): string {
  if (parsed.ctrl) return name              // ctrl+c → 'c'
  if (seq.length === 1) {
    const code = seq.charCodeAt(0)
    if (code >= 0x20 && code !== 0x7f) return seq  // 可打印字符 → 字面值
  }
  return name || seq                         // 特殊键 → 名称 ('down', 'return')
}
```

DOM 路径通过 `Dispatcher.dispatchDiscrete` 进入捕获/冒泡循环，事件从焦点节点出发、沿 DOM 树传播。

## 4.4 W3C 事件模型在终端中的实现

### 4.4.1 TerminalEvent 基类

```typescript
export class TerminalEvent extends Event {
  readonly type: string
  readonly timeStamp: number
  readonly bubbles: boolean
  readonly cancelable: boolean

  private _target: EventTarget | null = null
  private _currentTarget: EventTarget | null = null
  private _eventPhase: EventPhase = 'none'
  private _propagationStopped = false
  private _defaultPrevented = false

  stopPropagation(): void {
    this._propagationStopped = true
  }

  override stopImmediatePropagation(): void {
    super.stopImmediatePropagation()
    this._propagationStopped = true
  }

  preventDefault(): void {
    if (this.cancelable) {
      this._defaultPrevented = true
    }
  }

  _prepareForTarget(_target: EventTarget): void {}
}
```

完全模拟浏览器 `Event` 的行为：`target`（初始派发目标）、`currentTarget`（当前处理器所在节点）、`eventPhase`（none/capturing/at_target/bubbling）、`stopPropagation`、`stopImmediatePropagation`、`preventDefault`。

`_prepareForTarget` 是一个钩子方法，子类可以在每个处理器执行前做节点级的准备工作。

### 4.4.2 事件处理器注册

处理器通过 React props 声明，遵循 React 命名约定：

```typescript
export type EventHandlerProps = {
  onKeyDown?: KeyboardEventHandler
  onKeyDownCapture?: KeyboardEventHandler
  onFocus?: FocusEventHandler
  onFocusCapture?: FocusEventHandler
  onBlur?: FocusEventHandler
  onBlurCapture?: FocusEventHandler
  onPaste?: PasteEventHandler
  onPasteCapture?: PasteEventHandler
  onResize?: ResizeEventHandler
  onClick?: ClickEventHandler
  onMouseEnter?: HoverEventHandler
  onMouseLeave?: HoverEventHandler
}
```

`onEventName` 用于冒泡阶段，`onEventNameCapture` 用于捕获阶段。

反向查找表实现 O(1) 的处理器定位：

```typescript
export const HANDLER_FOR_EVENT: Record<
  string,
  { bubble?: keyof EventHandlerProps; capture?: keyof EventHandlerProps }
> = {
  keydown: { bubble: 'onKeyDown', capture: 'onKeyDownCapture' },
  focus: { bubble: 'onFocus', capture: 'onFocusCapture' },
  blur: { bubble: 'onBlur', capture: 'onBlurCapture' },
  click: { bubble: 'onClick' },
  // ...
}
```

处理器在 reconciler 中被存储到 `_eventHandlers`（而非 `attributes`），这样 handler 身份变化不会触发 `markDirty`，避免无谓的 Yoga 重新布局。

### 4.4.3 Dispatcher -- 捕获/冒泡分发

`Dispatcher` 是事件系统的核心，源码位于 `src/ink/events/dispatcher.ts`。

**监听器收集**（react-dom 的双阶段累积模式）：

```typescript
function collectListeners(
  target: EventTarget,
  event: TerminalEvent,
): DispatchListener[] {
  const listeners: DispatchListener[] = []

  let node: EventTarget | undefined = target
  while (node) {
    const isTarget = node === target

    const captureHandler = getHandler(node, event.type, true)
    const bubbleHandler = getHandler(node, event.type, false)

    if (captureHandler) {
      listeners.unshift({
        node,
        handler: captureHandler,
        phase: isTarget ? 'at_target' : 'capturing',
      })
    }

    if (bubbleHandler && (event.bubbles || isTarget)) {
      listeners.push({
        node,
        handler: bubbleHandler,
        phase: isTarget ? 'at_target' : 'bubbling',
      })
    }

    node = node.parentNode
  }

  return listeners
}
```

算法细节：

- 从目标节点沿 `parentNode` 链向上遍历。
- 捕获处理器用 `unshift` 插入数组头部 → 根节点最先执行。
- 冒泡处理器用 `push` 追加数组尾部 → 目标节点最先执行。
- 最终顺序：`[root-cap, ..., parent-cap, target-cap, target-bub, parent-bub, ..., root-bub]`

**分发执行**：

```typescript
function processDispatchQueue(
  listeners: DispatchListener[],
  event: TerminalEvent,
): void {
  let previousNode: EventTarget | undefined

  for (const { node, handler, phase } of listeners) {
    if (event._isImmediatePropagationStopped()) break
    if (event._isPropagationStopped() && node !== previousNode) break

    event._setEventPhase(phase)
    event._setCurrentTarget(node)
    event._prepareForTarget(node)

    try {
      handler(event)
    } catch (error) {
      logError(error)
    }

    previousNode = node
  }
}
```

两级传播控制：

- `stopPropagation()`：当节点变化时停止（同一节点上的多个处理器仍然执行）。
- `stopImmediatePropagation()`：立即停止（同一节点上的后续处理器也不执行）。

### 4.4.4 事件优先级

```typescript
function getEventPriority(eventType: string): number {
  switch (eventType) {
    case 'keydown':
    case 'keyup':
    case 'click':
    case 'focus':
    case 'blur':
    case 'paste':
      return DiscreteEventPriority    // 同步优先级
    case 'resize':
    case 'scroll':
    case 'mousemove':
      return ContinuousEventPriority  // 连续优先级
    default:
      return DefaultEventPriority
  }
}
```

这直接映射到 React 的调度优先级：

- **Discrete**：用户直接交互（按键、点击），需要同步处理。通过 `dispatchDiscrete` 进入 reconciler 的 `discreteUpdates` 上下文。
- **Continuous**：高频事件（滚动、鼠标移动），可以合并。通过 `dispatchContinuous` 设置连续优先级。

```typescript
dispatchDiscrete(target: EventTarget, event: TerminalEvent): boolean {
  if (!this.discreteUpdates) {
    return this.dispatch(target, event)
  }
  return this.discreteUpdates(
    (t, e) => this.dispatch(t, e),
    target, event, undefined, undefined,
  )
}

dispatchContinuous(target: EventTarget, event: TerminalEvent): boolean {
  const previousPriority = this.currentUpdatePriority
  try {
    this.currentUpdatePriority = ContinuousEventPriority
    return this.dispatch(target, event)
  } finally {
    this.currentUpdatePriority = previousPriority
  }
}
```

## 4.5 键绑定系统分层架构

### 4.5.1 配置解析 (parser.ts)

键绑定从声明式配置（JSON 或代码常量）解析为结构化对象：

```typescript
export function parseKeystroke(input: string): ParsedKeystroke {
  const parts = input.split('+')
  const keystroke: ParsedKeystroke = {
    key: '', ctrl: false, alt: false, shift: false, meta: false, super: false,
  }
  for (const part of parts) {
    const lower = part.toLowerCase()
    switch (lower) {
      case 'ctrl': case 'control':
        keystroke.ctrl = true; break
      case 'alt': case 'opt': case 'option':
        keystroke.alt = true; break
      case 'shift':
        keystroke.shift = true; break
      case 'meta':
        keystroke.meta = true; break
      case 'cmd': case 'command': case 'super': case 'win':
        keystroke.super = true; break
      case 'esc':
        keystroke.key = 'escape'; break
      case 'return':
        keystroke.key = 'enter'; break
      case 'space':
        keystroke.key = ' '; break
      default:
        keystroke.key = lower; break
    }
  }
  return keystroke
}
```

支持丰富的修饰键别名：`ctrl`/`control`、`alt`/`opt`/`option`、`cmd`/`command`/`super`/`win`。方向键也有 Unicode 别名（`←` → `left`）。

Chord（和弦序列）通过空格分割：

```typescript
export function parseChord(input: string): Chord {
  if (input === ' ') return [parseKeystroke('space')]
  return input.trim().split(/\s+/).map(parseKeystroke)
}
```

特殊处理：单个空格字符是 `space` 键绑定，不是 chord 分隔符。

配置块到扁平绑定列表的转换：

```typescript
export function parseBindings(blocks: KeybindingBlock[]): ParsedBinding[] {
  const bindings: ParsedBinding[] = []
  for (const block of blocks) {
    for (const [key, action] of Object.entries(block.bindings)) {
      bindings.push({
        chord: parseChord(key),
        action,
        context: block.context,
      })
    }
  }
  return bindings
}
```

### 4.5.2 默认键绑定配置 (defaultBindings.ts)

配置以 `KeybindingBlock[]` 形式组织，每个 block 对应一个上下文（context）：

```typescript
export const DEFAULT_BINDINGS: KeybindingBlock[] = [
  {
    context: 'Global',
    bindings: {
      'ctrl+c': 'app:interrupt',
      'ctrl+d': 'app:exit',
      'ctrl+l': 'app:redraw',
      'ctrl+t': 'app:toggleTodos',
      'ctrl+r': 'history:search',
    },
  },
  {
    context: 'Chat',
    bindings: {
      escape: 'chat:cancel',
      'ctrl+x ctrl+k': 'chat:killAgents',  // chord 绑定
      enter: 'chat:submit',
      up: 'history:previous',
      'ctrl+_': 'chat:undo',
      'ctrl+shift+-': 'chat:undo',
    },
  },
  {
    context: 'Scroll',
    bindings: {
      pageup: 'scroll:pageUp',
      wheelup: 'scroll:lineUp',
      'ctrl+shift+c': 'selection:copy',
      'cmd+c': 'selection:copy',
    },
  },
  // ... Autocomplete, Confirmation, Tabs, Transcript 等
]
```

设计要点：

- **双绑定兼容性**：`ctrl+_` 和 `ctrl+shift+-` 都绑定 undo——前者是传统终端发送 `\x1f` 控制字符，后者是 Kitty 协议发送物理按键+修饰符。
- **平台自适应**：`IMAGE_PASTE_KEY` 在 Windows 上是 `alt+v`（因为 `ctrl+v` 是系统粘贴），其他平台是 `ctrl+v`。
- **Chord 避冲突**：`ctrl+x ctrl+k` (kill agents) 和 `ctrl+x ctrl+e` (external editor) 使用 `ctrl+x` 前缀，避免与 readline 编辑键冲突。

### 4.5.3 按键匹配 (match.ts)

匹配核心是将 Ink 的 `Key` 对象与 `ParsedKeystroke` 进行比较：

```typescript
export function getKeyName(input: string, key: Key): string | null {
  if (key.escape) return 'escape'
  if (key.return) return 'enter'
  if (key.tab) return 'tab'
  if (key.backspace) return 'backspace'
  if (key.upArrow) return 'up'
  if (key.downArrow) return 'down'
  if (input.length === 1) return input.toLowerCase()
  return null
}
```

修饰符匹配中的 alt/meta 合并处理：

```typescript
function modifiersMatch(inkMods: InkModifiers, target: ParsedKeystroke): boolean {
  if (inkMods.ctrl !== target.ctrl) return false
  if (inkMods.shift !== target.shift) return false

  // Alt 和 meta 在终端中不可区分，合并处理
  const targetNeedsMeta = target.alt || target.meta
  if (inkMods.meta !== targetNeedsMeta) return false

  // Super (cmd/win) 是独立修饰键
  if (inkMods.super !== target.super) return false

  return true
}
```

ESC 键的特殊处理：

```typescript
export function matchesKeystroke(input: string, key: Key, target: ParsedKeystroke): boolean {
  const keyName = getKeyName(input, key)
  if (keyName !== target.key) return false

  const inkMods = getInkModifiers(key)

  // Ink 在按下 ESC 时设置 key.meta=true，这是终端历史遗留
  // 匹配 ESC 键时忽略 meta 修饰符
  if (key.escape) {
    return modifiersMatch({ ...inkMods, meta: false }, target)
  }

  return modifiersMatch(inkMods, target)
}
```

### 4.5.4 键绑定解析 (resolver.ts)

单键解析（Phase 1）：

```typescript
export function resolveKey(
  input: string,
  key: Key,
  activeContexts: KeybindingContextName[],
  bindings: ParsedBinding[],
): ResolveResult {
  let match: ParsedBinding | undefined
  const ctxSet = new Set(activeContexts)

  for (const binding of bindings) {
    if (binding.chord.length !== 1) continue  // 只匹配单键绑定
    if (!ctxSet.has(binding.context)) continue
    if (matchesBinding(input, key, binding)) {
      match = binding  // 后定义的覆盖先定义的
    }
  }

  if (!match) return { type: 'none' }
  if (match.action === null) return { type: 'unbound' }
  return { type: 'match', action: match.action }
}
```

"最后一个匹配胜出"规则使得用户绑定（加载在默认绑定之后）自动覆盖默认值。`action === null` 表示显式解绑。

## 4.6 Chord（和弦序列）实现

Chord 是多键序列绑定，如 `ctrl+x ctrl+k`。实现在 `resolveKeyWithChordState` 中：

```typescript
export function resolveKeyWithChordState(
  input: string,
  key: Key,
  activeContexts: KeybindingContextName[],
  bindings: ParsedBinding[],
  pending: ParsedKeystroke[] | null,
): ChordResolveResult {
  // ESC 取消正在进行的 chord
  if (key.escape && pending !== null) {
    return { type: 'chord_cancelled' }
  }

  const currentKeystroke = buildKeystroke(input, key)
  if (!currentKeystroke) {
    if (pending !== null) return { type: 'chord_cancelled' }
    return { type: 'none' }
  }

  const testChord = pending ? [...pending, currentKeystroke] : [currentKeystroke]

  const ctxSet = new Set(activeContexts)
  const contextBindings = bindings.filter(b => ctxSet.has(b.context))

  // 检查是否有更长的 chord 以当前序列为前缀
  const chordWinners = new Map<string, string | null>()
  for (const binding of contextBindings) {
    if (binding.chord.length > testChord.length &&
        chordPrefixMatches(testChord, binding)) {
      chordWinners.set(chordToString(binding.chord), binding.action)
    }
  }
  let hasLongerChords = false
  for (const action of chordWinners.values()) {
    if (action !== null) { hasLongerChords = true; break }
  }

  // 如果存在更长的 chord 匹配，进入等待状态
  if (hasLongerChords) {
    return { type: 'chord_started', pending: testChord }
  }

  // 检查精确匹配
  let exactMatch: ParsedBinding | undefined
  for (const binding of contextBindings) {
    if (chordExactlyMatches(testChord, binding)) {
      exactMatch = binding
    }
  }

  if (exactMatch) {
    if (exactMatch.action === null) return { type: 'unbound' }
    return { type: 'match', action: exactMatch.action }
  }

  if (pending !== null) return { type: 'chord_cancelled' }
  return { type: 'none' }
}
```

返回类型是五态联合：

```typescript
export type ChordResolveResult =
  | { type: 'match'; action: string }
  | { type: 'none' }
  | { type: 'unbound' }
  | { type: 'chord_started'; pending: ParsedKeystroke[] }
  | { type: 'chord_cancelled' }
```

Chord 的 null-override 处理是一个精巧的设计：如果用户在配置中将 `ctrl+x ctrl+k` 设为 `null`（解绑），但存在另一个以 `ctrl+x` 为前缀的活跃 chord，那么 `ctrl+x` 仍会进入 chord 等待。但如果 `ctrl+x` 前缀下的所有 chord 都被解绑了，`ctrl+x` 就不会进入 chord 等待，单键绑定（如果存在）能正常触发。这通过 `chordWinners` Map 中检查是否有非 null action 实现。

Keystroke 比较中 alt/meta 的合并：

```typescript
export function keystrokesEqual(a: ParsedKeystroke, b: ParsedKeystroke): boolean {
  return (
    a.key === b.key &&
    a.ctrl === b.ctrl &&
    a.shift === b.shift &&
    (a.alt || a.meta) === (b.alt || b.meta) &&
    a.super === b.super
  )
}
```

终端无法区分 alt 和 meta，所以 `alt+k` 和 `meta+k` 被视为同一按键。

## 4.7 Vim 模式纯函数状态机

### 4.7.1 状态类型定义

Vim 模式的类型定义在 `src/vim/types.ts` 中，它本身就是完整的设计文档。

顶层状态：

```typescript
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

INSERT 模式跟踪输入的文本（用于 dot-repeat）。NORMAL 模式包含一个命令解析状态机。

NORMAL 模式的命令状态机：

```typescript
export type CommandState =
  | { type: 'idle' }
  | { type: 'count'; digits: string }
  | { type: 'operator'; op: Operator; count: number }
  | { type: 'operatorCount'; op: Operator; count: number; digits: string }
  | { type: 'operatorFind'; op: Operator; count: number; find: FindType }
  | { type: 'operatorTextObj'; op: Operator; count: number; scope: TextObjScope }
  | { type: 'find'; find: FindType; count: number }
  | { type: 'g'; count: number }
  | { type: 'operatorG'; op: Operator; count: number }
  | { type: 'replace'; count: number }
  | { type: 'indent'; dir: '>' | '<'; count: number }
```

状态转移图（摘自类型文件注释）：

```
idle ──┬─[d/c/y]──► operator
       ├─[1-9]────► count
       ├─[fFtT]───► find
       ├─[g]──────► g
       ├─[r]──────► replace
       └─[><]─────► indent

operator ─┬─[motion]──► execute
           ├─[0-9]────► operatorCount
           ├─[ia]─────► operatorTextObj
           └─[fFtT]───► operatorFind
```

持久状态（跨命令保持）：

```typescript
export type PersistentState = {
  lastChange: RecordedChange | null   // dot-repeat 记录
  lastFind: { type: FindType; char: string } | null  // ;/, 重复查找
  register: string                     // 粘贴寄存器
  registerIsLinewise: boolean          // 行级粘贴标记
}
```

### 4.7.2 transition 函数逐行讲解

核心转移函数在 `src/vim/transitions.ts`：

```typescript
export function transition(
  state: CommandState,
  input: string,
  ctx: TransitionContext,
): TransitionResult {
  switch (state.type) {
    case 'idle':        return fromIdle(input, ctx)
    case 'count':       return fromCount(state, input, ctx)
    case 'operator':    return fromOperator(state, input, ctx)
    case 'operatorCount': return fromOperatorCount(state, input, ctx)
    case 'operatorFind':  return fromOperatorFind(state, input, ctx)
    case 'operatorTextObj': return fromOperatorTextObj(state, input, ctx)
    case 'find':        return fromFind(state, input, ctx)
    case 'g':           return fromG(state, input, ctx)
    case 'operatorG':   return fromOperatorG(state, input, ctx)
    case 'replace':     return fromReplace(state, input, ctx)
    case 'indent':      return fromIndent(state, input, ctx)
  }
}
```

返回类型：

```typescript
export type TransitionResult = {
  next?: CommandState   // 新状态（省略 = 回到 idle）
  execute?: () => void  // 要执行的副作用
}
```

**fromIdle -- 空闲状态的入口逻辑**：

```typescript
function fromIdle(input: string, ctx: TransitionContext): TransitionResult {
  if (/[1-9]/.test(input)) {
    return { next: { type: 'count', digits: input } }
  }
  if (input === '0') {
    return {
      execute: () => ctx.setOffset(ctx.cursor.startOfLogicalLine().offset),
    }
  }
  const result = handleNormalInput(input, 1, ctx)
  if (result) return result
  return {}
}
```

`0` 在 idle 状态是行首移动（不是 count 前缀），这是 Vim 的经典规则。非零数字进入 count 状态。其他输入委托给 `handleNormalInput`。

**handleNormalInput -- 共享的普通输入处理**：

```typescript
function handleNormalInput(
  input: string,
  count: number,
  ctx: TransitionContext,
): TransitionResult | null {
  if (isOperatorKey(input)) {
    return { next: { type: 'operator', op: OPERATORS[input], count } }
  }

  if (SIMPLE_MOTIONS.has(input)) {
    return {
      execute: () => {
        const target = resolveMotion(input, ctx.cursor, count)
        ctx.setOffset(target.offset)
      },
    }
  }

  if (FIND_KEYS.has(input)) {
    return { next: { type: 'find', find: input as FindType, count } }
  }

  if (input === 'x') {
    return { execute: () => executeX(count, ctx) }
  }
  if (input === 'D') {
    return { execute: () => executeOperatorMotion('delete', '$', 1, ctx) }
  }
  if (input === '.') {
    return { execute: () => ctx.onDotRepeat?.() }
  }
  if (input === 'i') {
    return { execute: () => ctx.enterInsert(ctx.cursor.offset) }
  }
  if (input === 'A') {
    return {
      execute: () => ctx.enterInsert(ctx.cursor.endOfLogicalLine().offset),
    }
  }
  // ...
  return null
}
```

这个函数被 `fromIdle` 和 `fromCount` 共享——count 只影响传入的 `count` 参数，不影响逻辑结构。

**fromOperator -- 等待 motion/text-object**：

```typescript
function fromOperator(state, input, ctx): TransitionResult {
  // dd, cc, yy = 行操作
  if (input === state.op[0]) {
    return { execute: () => executeLineOp(state.op, state.count, ctx) }
  }

  if (/[0-9]/.test(input)) {
    return {
      next: {
        type: 'operatorCount',
        op: state.op,
        count: state.count,
        digits: input,
      },
    }
  }

  const result = handleOperatorInput(state.op, state.count, input, ctx)
  if (result) return result

  return { next: { type: 'idle' } }
}
```

`dd` 的检测：如果输入字符等于 operator 键的首字母（`d` 的首字母是 `d`，`c` 的首字母是 `c`），触发行操作。这自然地支持了 `dd`、`cc`、`yy`。

**handleOperatorInput -- operator 后的 motion 处理**：

```typescript
function handleOperatorInput(
  op: Operator,
  count: number,
  input: string,
  ctx: TransitionContext,
): TransitionResult | null {
  if (isTextObjScopeKey(input)) {
    return {
      next: { type: 'operatorTextObj', op, count, scope: TEXT_OBJ_SCOPES[input] },
    }
  }

  if (FIND_KEYS.has(input)) {
    return {
      next: { type: 'operatorFind', op, count, find: input as FindType },
    }
  }

  if (SIMPLE_MOTIONS.has(input)) {
    return { execute: () => executeOperatorMotion(op, input, count, ctx) }
  }

  if (input === 'G') {
    return { execute: () => executeOperatorG(op, count, ctx) }
  }

  if (input === 'g') {
    return { next: { type: 'operatorG', op, count } }
  }

  return null
}
```

`i`/`a` 在 operator 后面是文本对象作用域（inner/around），不是 insert 命令。这是上下文敏感的——同一个按键在不同状态有不同含义。

**fromOperatorCount -- count 乘法**：

```typescript
function fromOperatorCount(state, input, ctx): TransitionResult {
  if (/[0-9]/.test(input)) {
    const newDigits = state.digits + input
    const parsedDigits = Math.min(parseInt(newDigits, 10), MAX_VIM_COUNT)
    return { next: { ...state, digits: String(parsedDigits) } }
  }

  const motionCount = parseInt(state.digits, 10)
  const effectiveCount = state.count * motionCount  // count 相乘
  const result = handleOperatorInput(state.op, effectiveCount, input, ctx)
  if (result) return result
  return { next: { type: 'idle' } }
}
```

`3d2w` = 删除 6 个单词。预命令 count (3) 和 motion count (2) 相乘。`MAX_VIM_COUNT = 10000` 防止意外的巨大数字。

### 4.7.3 Motion 函数

`src/vim/motions.ts` 实现纯函数的光标移动：

```typescript
export function resolveMotion(key: string, cursor: Cursor, count: number): Cursor {
  let result = cursor
  for (let i = 0; i < count; i++) {
    const next = applySingleMotion(key, result)
    if (next.equals(result)) break  // 到达边界则停止
    result = next
  }
  return result
}

function applySingleMotion(key: string, cursor: Cursor): Cursor {
  switch (key) {
    case 'h': return cursor.left()
    case 'l': return cursor.right()
    case 'j': return cursor.downLogicalLine()
    case 'k': return cursor.upLogicalLine()
    case 'gj': return cursor.down()      // 显示行移动
    case 'gk': return cursor.up()        // 显示行移动
    case 'w': return cursor.nextVimWord()
    case 'b': return cursor.prevVimWord()
    case 'e': return cursor.endOfVimWord()
    case '0': return cursor.startOfLogicalLine()
    case '^': return cursor.firstNonBlankInLogicalLine()
    case '$': return cursor.endOfLogicalLine()
    default: return cursor
  }
}
```

关键设计：motion 是 immutable 的——`cursor.left()` 返回新的 `Cursor` 对象，不修改原对象。这使得整个 transition 流程是纯函数，可以安全地测试和推理。

边界检测通过 `equals` 比较实现：如果 motion 后位置不变（到达文档边界），count 循环提前退出。

Motion 的分类影响 operator 行为：

```typescript
export function isInclusiveMotion(key: string): boolean {
  return 'eE$'.includes(key)
}

export function isLinewiseMotion(key: string): boolean {
  return 'jkG'.includes(key) || key === 'gg'
}
```

`de` 删除到单词末尾（包含末尾字符），`dj` 删除两行（行级操作）。`gj`/`gk` 是显示行移动，按 `:help gj` 定义为 characterwise exclusive（不是 linewise）。

### 4.7.4 文本对象

`src/vim/textObjects.ts` 实现文本对象的范围查找：

```typescript
export function findTextObject(
  text: string,
  offset: number,
  objectType: string,
  isInner: boolean,
): TextObjectRange {
  if (objectType === 'w')
    return findWordObject(text, offset, isInner, isVimWordChar)
  if (objectType === 'W')
    return findWordObject(text, offset, isInner, ch => !isVimWhitespace(ch))

  const pair = PAIRS[objectType]
  if (pair) {
    const [open, close] = pair
    return open === close
      ? findQuoteObject(text, offset, open, isInner)
      : findBracketObject(text, offset, open, close, isInner)
  }

  return null
}
```

三种查找策略：

1. **Word object** (`iw`/`aw`)：基于 grapheme 分词，区分 word char、punctuation、whitespace。`isInner` 只包含单词本身，`around` 还包含周围空白。
2. **Quote object** (`i"`/`a"`)：在当前行内配对引号，支持嵌套（按 0-1, 2-3, 4-5 配对）。
3. **Bracket object** (`i(`/`a(`)：深度计数的括号匹配，支持嵌套。

Word object 使用 `Intl.Segmenter` 做 grapheme 安全的迭代：

```typescript
function findWordObject(text, offset, isInner, isWordChar) {
  const graphemes: Array<{ segment: string; index: number }> = []
  for (const { segment, index } of getGraphemeSegmenter().segment(text)) {
    graphemes.push({ segment, index })
  }
  // 基于 grapheme 索引而非 codepoint 索引进行范围计算
  // ...
}
```

这确保了 emoji 和 CJK 字符被正确处理为单个单位。

## 4.8 焦点管理机制

源码位于 `src/ink/focus.ts`。`FocusManager` 是一个纯状态管理器，存储在 `ink-root` 的 `focusManager` 字段上。

```typescript
export class FocusManager {
  activeElement: DOMElement | null = null
  private dispatchFocusEvent: (target: DOMElement, event: FocusEvent) => boolean
  private enabled = true
  private focusStack: DOMElement[] = []

  focus(node: DOMElement): void {
    if (node === this.activeElement) return
    if (!this.enabled) return

    const previous = this.activeElement
    if (previous) {
      const idx = this.focusStack.indexOf(previous)
      if (idx !== -1) this.focusStack.splice(idx, 1)
      this.focusStack.push(previous)
      if (this.focusStack.length > MAX_FOCUS_STACK) this.focusStack.shift()
      this.dispatchFocusEvent(previous, new FocusEvent('blur', node))
    }
    this.activeElement = node
    this.dispatchFocusEvent(node, new FocusEvent('focus', previous))
  }
}
```

焦点转移的流程：

1. 将当前焦点元素压入 `focusStack`（去重后压栈）。
2. 向旧元素分发 `blur` 事件（`relatedTarget` 指向新元素）。
3. 设置新的 `activeElement`。
4. 向新元素分发 `focus` 事件（`relatedTarget` 指向旧元素）。

焦点栈有 `MAX_FOCUS_STACK = 32` 的大小限制，防止无限增长。Tab 循环会反复 push 同一组元素，去重逻辑避免了栈膨胀。

节点移除时的焦点恢复：

```typescript
handleNodeRemoved(node: DOMElement, root: DOMElement): void {
  this.focusStack = this.focusStack.filter(
    n => n !== node && isInTree(n, root),
  )

  if (!this.activeElement) return
  if (this.activeElement !== node && isInTree(this.activeElement, root)) return

  const removed = this.activeElement
  this.activeElement = null
  this.dispatchFocusEvent(removed, new FocusEvent('blur', null))

  while (this.focusStack.length > 0) {
    const candidate = this.focusStack.pop()!
    if (isInTree(candidate, root)) {
      this.activeElement = candidate
      this.dispatchFocusEvent(candidate, new FocusEvent('focus', removed))
      return
    }
  }
}
```

当焦点元素或其祖先被移除时：先清理焦点栈中所有不再在树中的节点，然后从栈顶弹出最近的仍存活元素来恢复焦点。这保证了组件卸载后焦点不会悬空。

Tab 导航：

```typescript
focusNext(root: DOMElement): void {
  this.moveFocus(1, root)
}

private moveFocus(direction: 1 | -1, root: DOMElement): void {
  const tabbable = collectTabbable(root)
  if (tabbable.length === 0) return

  const currentIndex = this.activeElement
    ? tabbable.indexOf(this.activeElement)
    : -1

  const nextIndex =
    currentIndex === -1
      ? direction === 1 ? 0 : tabbable.length - 1
      : (currentIndex + direction + tabbable.length) % tabbable.length

  const next = tabbable[nextIndex]
  if (next) this.focus(next)
}
```

`collectTabbable` 做深度优先遍历，收集所有 `tabIndex >= 0` 的节点。`tabIndex = -1` 表示只能通过编程方式获取焦点（类似浏览器的 `tabindex="-1"`）。焦点循环使用模运算实现——到达末尾后回到开头。

`FocusEvent` 的设计遵循 react-dom 的语义：

```typescript
export class FocusEvent extends TerminalEvent {
  readonly relatedTarget: EventTarget | null

  constructor(type: 'focus' | 'blur', relatedTarget: EventTarget | null = null) {
    super(type, { bubbles: true, cancelable: false })
    this.relatedTarget = relatedTarget
  }
}
```

`bubbles: true` 使得父组件可以通过 `onFocus`/`onBlur` 观察子组件的焦点变化，这与 react-dom 使用 `focusin`/`focusout`（冒泡版 focus/blur）的行为一致。

## 4.9 小结

Claude Code 的事件系统是一个分层架构：

1. **底层解析**：termio tokenizer 处理转义序列边界，parse-keypress 做语义映射，支持传统 VT100、Kitty 协议和 xterm modifyOtherKeys 三种输入格式。
2. **双事件路径**：EventEmitter 旧路径（`useInput`）处理字符级输入，DOM 新路径（`Dispatcher`）实现 W3C 捕获/冒泡模型。两条路径共存，前者面向简单字符处理，后者面向结构化事件分发。
3. **键绑定系统**：三级匹配——context 过滤、keystroke 比较、chord 状态机。配置驱动，用户覆盖默认值，null 显式解绑。
4. **Vim 状态机**：纯函数设计，状态 + 输入 → 新状态 + 可选副作用。TypeScript discriminated union 确保穷举处理。
5. **焦点管理**：栈式历史记录，节点移除时自动恢复，Tab 循环导航。
