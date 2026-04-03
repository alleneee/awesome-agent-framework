# 第3章 终端UI渲染引擎

Claude Code 并未直接使用社区版 Ink 框架，而是在其基础上构建了一套完整的终端 UI 渲染引擎。该引擎包含自定义 React reconciler、轻量 DOM 模型、Yoga 布局适配层、帧调度机制和双缓冲 diff 渲染算法。本章将逐层剖析这些子系统的设计与实现。

## 3.1 架构总览

渲染管线从 React 组件树到终端像素的完整路径如下：

```
React 组件 → reconciler → DOM 树 → Yoga 布局 → Screen Buffer → Diff → 终端输出
```

涉及的核心源码文件：

| 文件 | 职责 |
|------|------|
| `src/ink/reconciler.ts` | 自定义 React reconciler host config |
| `src/ink/dom.ts` | 轻量 DOM 模型 (DOMElement / TextNode) |
| `src/ink/layout/node.ts` | LayoutNode 抽象接口 |
| `src/ink/layout/yoga.ts` | Yoga 适配器实现 |
| `src/ink/layout/engine.ts` | 布局节点工厂 |
| `src/ink/screen.ts` | Screen buffer 与 packed bitmap 格式 |
| `src/ink/ink.tsx` | Ink 主类：帧调度、双缓冲、渲染循环 |
| `src/ink/renderer.ts` | 渲染器：DOM 树 → Screen buffer |
| `src/ink/log-update.ts` | Diff 引擎：双缓冲比较与终端输出 |
| `src/ink/constants.ts` | 帧间隔常量 |

## 3.2 为什么要自定义 React Reconciler

社区版 Ink 使用 react-reconciler 构建了一个终端渲染器，但 Claude Code 对其进行了深度重写。主要原因包括：

1. **W3C 事件模型**：Claude Code 需要捕获/冒泡事件分发，社区版 Ink 只有扁平的 EventEmitter。
2. **React 19 适配**：reconciler 需要实现 `commitUpdate` 直接接收新旧 props（而非 updatePayload），以及 `maySuspendCommit` 等新方法。
3. **性能优先的 dirty 标记**：通过 `markDirty` 和 `stylesEqual` 等机制减少不必要的 Yoga 重新测量。
4. **与 Dispatcher 集成**：reconciler 的优先级系统（`getCurrentUpdatePriority` / `resolveUpdatePriority`）直接委托给自定义的 `Dispatcher`。
5. **调试能力**：通过 `getOwnerChain` 追踪组件栈用于 repaint 归因。

## 3.3 Reconciler Host Config 关键实现

源码位于 `src/ink/reconciler.ts`。整个 reconciler 通过 `createReconciler` 创建，类型参数指定了宿主环境的类型映射：

```typescript
const reconciler = createReconciler<
  ElementNames,    // 宿主元素类型名
  Props,           // props 对象类型
  DOMElement,      // 容器类型
  DOMElement,      // 实例类型
  TextNode,        // 文本实例类型
  DOMElement,      // 挂起实例类型
  unknown,         // 公共实例类型
  unknown,         // 水合实例类型
  DOMElement,      // HostContext 持有者
  HostContext,     // 宿主上下文
  null,            // UpdatePayload - React 19 不再使用
  NodeJS.Timeout,  // setTimeout 返回类型
  -1,              // noTimeout 哨兵值
  null             // 过渡状态
>({
  // ... host config 方法
})
```

### 3.3.1 createInstance -- 创建宿主元素

```typescript
createInstance(
  originalType: ElementNames,
  newProps: Props,
  _root: DOMElement,
  hostContext: HostContext,
  internalHandle?: unknown,
): DOMElement {
  if (hostContext.isInsideText && originalType === 'ink-box') {
    throw new Error(`<Box> can't be nested inside <Text> component`)
  }

  const type =
    originalType === 'ink-text' && hostContext.isInsideText
      ? 'ink-virtual-text'
      : originalType

  const node = createNode(type)
  if (COMMIT_LOG) _createCount++

  for (const [key, value] of Object.entries(newProps)) {
    applyProp(node, key, value)
  }

  if (isDebugRepaintsEnabled()) {
    node.debugOwnerChain = getOwnerChain(internalHandle)
  }

  return node
}
```

核心逻辑：

- **类型降级**：当 `ink-text` 嵌套在文本上下文中时，自动降级为 `ink-virtual-text`。`ink-virtual-text` 不创建 Yoga 节点，因为它不参与布局计算，只是文本内容的容器。
- **props 应用**：遍历所有 props 调用 `applyProp`，区分 `style`、`textStyles`、事件处理器和普通属性四种情况。
- **调试追踪**：当环境变量 `CLAUDE_CODE_DEBUG_REPAINTS` 开启时，通过 React Fiber 的 `_debugOwner` 链追踪组件栈。

### 3.3.2 createTextInstance -- 创建文本节点

```typescript
createTextInstance(
  text: string,
  _root: DOMElement,
  hostContext: HostContext,
): TextNode {
  if (!hostContext.isInsideText) {
    throw new Error(
      `Text string "${text}" must be rendered inside <Text> component`,
    )
  }
  return createTextNode(text)
}
```

强制要求纯文本必须在 `<Text>` 组件内部渲染，这是 Ink 对终端渲染的约束——裸文本没有样式信息，无法确定如何着色。

### 3.3.3 commitUpdate -- React 19 的 props diff

```typescript
commitUpdate(
  node: DOMElement,
  _type: ElementNames,
  oldProps: Props,
  newProps: Props,
): void {
  const props = diff(oldProps, newProps)
  const style = diff(oldProps['style'] as Styles, newProps['style'] as Styles)

  if (props) {
    for (const [key, value] of Object.entries(props)) {
      if (key === 'style') {
        setStyle(node, value as Styles)
        continue
      }
      if (key === 'textStyles') {
        setTextStyles(node, value as TextStyles)
        continue
      }
      if (EVENT_HANDLER_PROPS.has(key)) {
        setEventHandler(node, key, value)
        continue
      }
      setAttribute(node, key, value as DOMNodeAttribute)
    }
  }

  if (style && node.yogaNode) {
    applyStyles(node.yogaNode, style, newProps['style'] as Styles)
  }
}
```

React 19 重大变化：`commitUpdate` 直接接收新旧 props，不再使用 `prepareUpdate` 返回的 `updatePayload`。这里用 `diff` 函数做浅比较，只更新实际变化的属性。事件处理器被特殊存储到 `_eventHandlers` 字段中，避免触发 `markDirty`��

### 3.3.4 resetAfterCommit -- 渲染触发点

```typescript
resetAfterCommit(rootNode) {
  // ... 性能计数省略
  if (typeof rootNode.onComputeLayout === 'function') {
    rootNode.onComputeLayout()
  }
  // 测试环境直接渲染
  if (process.env.NODE_ENV === 'test') {
    rootNode.onImmediateRender?.()
    return
  }
  // 生产环境走节流渲染
  rootNode.onRender?.()
}
```

这是 React reconciler 完成一次 commit 后的回调。两个关键操作：

1. **布局计算**：调用 `onComputeLayout` 触发 Yoga 布局，这样后续的 `useLayoutEffect` 就能拿到新鲜的布局数据。
2. **渲染调度**：`onRender` 指向 `Ink` 类上的 `scheduleRender`（节流版），而测试环境用 `onImmediateRender` 同步渲染。

### 3.3.5 优先级与事件系统集成

```typescript
getCurrentUpdatePriority: () => dispatcher.currentUpdatePriority,
setCurrentUpdatePriority(newPriority: number): void {
  dispatcher.currentUpdatePriority = newPriority
},
resolveUpdatePriority(): number {
  return dispatcher.resolveEventPriority()
},
resolveEventType(): string | null {
  return dispatcher.currentEvent?.type ?? null
},
resolveEventTimeStamp(): number {
  return dispatcher.currentEvent?.timeStamp ?? -1.1
},
```

reconciler 的优先级系统完全委托给 `Dispatcher`，后者在事件分发时设置 `currentEvent`，使得事件驱动的 React 更新能获得正确的调度优先级（discrete / continuous / default）。

文件末尾的关键连接：

```typescript
dispatcher.discreteUpdates = reconciler.discreteUpdates.bind(reconciler)
```

这打破了 `dispatcher.ts` 和 `reconciler.ts` 之间的循环依赖：Dispatcher 需要 reconciler 的 `discreteUpdates` 来实现同步优先级分发，但 reconciler 又依赖 Dispatcher 来解析事件优先级。通过延迟注入解决。

## 3.4 DOM 模型设计

源码位于 `src/ink/dom.ts`。这是一个极简的 DOM 模型，不追求浏览器兼容性，只保留终端渲染所需的最小字段集。

### 3.4.1 DOMElement 类型定义

```typescript
export type DOMElement = {
  nodeName: ElementNames
  attributes: Record<string, DOMNodeAttribute>
  childNodes: DOMNode[]
  textStyles?: TextStyles

  onComputeLayout?: () => void
  onRender?: () => void
  onImmediateRender?: () => void
  hasRenderedContent?: boolean

  dirty: boolean
  isHidden?: boolean
  _eventHandlers?: Record<string, unknown>

  scrollTop?: number
  pendingScrollDelta?: number
  scrollClampMin?: number
  scrollClampMax?: number
  scrollHeight?: number
  scrollViewportHeight?: number
  scrollViewportTop?: number
  stickyScroll?: boolean
  scrollAnchor?: { el: DOMElement; offset: number }
  focusManager?: FocusManager

  debugOwnerChain?: string[]
} & InkNode
```

各字段职责：

| 字段 | 说明 |
|------|------|
| `nodeName` | 元素类型，共7种：`ink-root`、`ink-box`、`ink-text`、`ink-virtual-text`、`ink-link`、`ink-progress`、`ink-raw-ansi` |
| `attributes` | 普通属性（tabIndex、rawWidth 等） |
| `childNodes` | 子节点数组（DOMElement 或 TextNode） |
| `dirty` | 脏标记，标识需要重新渲染 |
| `isHidden` | reconciler 的 `hideInstance` 设置，对应 `display: none` |
| `_eventHandlers` | 事件处理器独立存储，handler 身份变化不触发 markDirty |
| `scrollTop` / `pendingScrollDelta` | 滚动状态，renderer 在渲染时读取 |
| `stickyScroll` | 自动跟随底部 |
| `scrollAnchor` | 锚点滚动，渲染时延迟读取元素位置 |
| `focusManager` | 仅 `ink-root` 持有，子节点通过 `parentNode` 链到达 |
| `debugOwnerChain` | 调试用组件栈追踪 |

`InkNode` 是共享基类：

```typescript
type InkNode = {
  parentNode: DOMElement | undefined
  yogaNode?: LayoutNode
  style: Styles
}
```

### 3.4.2 TextNode

```typescript
export type TextNode = {
  nodeName: TextName  // '#text'
  nodeValue: string
} & InkNode
```

文本节点只有值，没有子节点和属性。但它也有 `parentNode`、`style`（继承自祖先）和可选的 `yogaNode`（通常为 `undefined`，因为文本节点不直接参与布局）。

### 3.4.3 节点创建

```typescript
export const createNode = (nodeName: ElementNames): DOMElement => {
  const needsYogaNode =
    nodeName !== 'ink-virtual-text' &&
    nodeName !== 'ink-link' &&
    nodeName !== 'ink-progress'
  const node: DOMElement = {
    nodeName,
    style: {},
    attributes: {},
    childNodes: [],
    parentNode: undefined,
    yogaNode: needsYogaNode ? createLayoutNode() : undefined,
    dirty: false,
  }

  if (nodeName === 'ink-text') {
    node.yogaNode?.setMeasureFunc(measureTextNode.bind(null, node))
  } else if (nodeName === 'ink-raw-ansi') {
    node.yogaNode?.setMeasureFunc(measureRawAnsiNode.bind(null, node))
  }

  return node
}
```

关键设计决策：

- `ink-virtual-text`、`ink-link`、`ink-progress` 不创建 Yoga 节点，它们不参与布局计算。
- `ink-text` 设置 `measureFunc` 让 Yoga 能查询文本的固有尺寸。测量函数会考虑文本换行（`wrapText`）和 tab 扩展（`expandTabs`）。
- `ink-raw-ansi` 直接从属性读取预计算的尺寸，跳过 `stringWidth` 计算。

### 3.4.4 markDirty -- 脏标记上传

```typescript
export const markDirty = (node?: DOMNode): void => {
  let current: DOMNode | undefined = node
  let markedYoga = false

  while (current) {
    if (current.nodeName !== '#text') {
      ;(current as DOMElement).dirty = true
      if (
        !markedYoga &&
        (current.nodeName === 'ink-text' ||
          current.nodeName === 'ink-raw-ansi') &&
        current.yogaNode
      ) {
        current.yogaNode.markDirty()
        markedYoga = true
      }
    }
    current = current.parentNode
  }
}
```

从当前节点沿 `parentNode` 链一路标记 `dirty = true`，直到根节点。同时，对 `ink-text` 和 `ink-raw-ansi` 叶节点调用 Yoga 的 `markDirty`，触发布局重新测量。这是级联标记——任何属性变更都会将整条祖先链标脏，但 Yoga 的 `markDirty` 只在叶节点调用一��。

### 3.4.5 树操作

`appendChildNode` 同时操作 DOM 树和 Yoga 树：

```typescript
export const appendChildNode = (
  node: DOMElement,
  childNode: DOMElement,
): void => {
  if (childNode.parentNode) {
    removeChildNode(childNode.parentNode, childNode)
  }
  childNode.parentNode = node
  node.childNodes.push(childNode)
  if (childNode.yogaNode) {
    node.yogaNode?.insertChild(
      childNode.yogaNode,
      node.yogaNode.getChildCount(),
    )
  }
  markDirty(node)
}
```

`insertBeforeNode` 需要计算正确的 Yoga 索引，因为某些 DOM 子节点（`ink-progress`、`ink-virtual-text`）没有 Yoga 节点，DOM 索引和 Yoga 索引并不一一对应：

```typescript
let yogaIndex = 0
if (newChildNode.yogaNode && node.yogaNode) {
  for (let i = 0; i < index; i++) {
    if (node.childNodes[i]?.yogaNode) {
      yogaIndex++
    }
  }
}
```

`removeChildNode` 在移除时收集被删除子树中的缓存矩形，用于帧间 blit 优化的失效处理。

## 3.5 Yoga 布局引擎适配层

### 3.5.1 LayoutNode 抽象接口

源码位于 `src/ink/layout/node.ts`。这是一个纯接口，将 Yoga 的具体 API 隐藏在抽象层之后：

```typescript
export type LayoutNode = {
  // 树操作
  insertChild(child: LayoutNode, index: number): void
  removeChild(child: LayoutNode): void
  getChildCount(): number
  getParent(): LayoutNode | null

  // 布局计算
  calculateLayout(width?: number, height?: number): void
  setMeasureFunc(fn: LayoutMeasureFunc): void
  unsetMeasureFunc(): void
  markDirty(): void

  // 布局结果读取
  getComputedLeft(): number
  getComputedTop(): number
  getComputedWidth(): number
  getComputedHeight(): number
  getComputedBorder(edge: LayoutEdge): number
  getComputedPadding(edge: LayoutEdge): number

  // 样式设置器（setWidth, setHeight, setFlexDirection 等）
  // ... 省略40+个设置器方法

  // 生命周期
  free(): void
  freeRecursive(): void
}
```

为什么需要这个抽象层？

1. **解耦 Yoga 版本**：Claude Code 使用的是 TypeScript 原生移植版的 Yoga（`src/native-ts/yoga-layout`），而非 WASM 版本。抽象层使得替换布局引擎不影响上层代码。
2. **类型安全的枚举映射**：Yoga 内部用数字枚举（`Display.Flex = 0`），LayoutNode 使用字符串枚举（`LayoutDisplay.Flex = 'flex'`），可读性更好且不依赖 Yoga 的数值约定。
3. **测量模式标准化**：`LayoutMeasureMode` 将 Yoga 的 `MeasureMode.Exactly` / `AtMost` / `Undefined` 统一为字符串标识符。

### 3.5.2 YogaLayoutNode 适配器

源码位于 `src/ink/layout/yoga.ts`。每个方法都是到 Yoga 原生 API 的直通委托，加上枚举映射：

```typescript
export class YogaLayoutNode implements LayoutNode {
  readonly yoga: YogaNode

  constructor(yoga: YogaNode) {
    this.yoga = yoga
  }

  setMeasureFunc(fn: LayoutMeasureFunc): void {
    this.yoga.setMeasureFunc((w, wMode) => {
      const mode =
        wMode === MeasureMode.Exactly
          ? LayoutMeasureMode.Exactly
          : wMode === MeasureMode.AtMost
            ? LayoutMeasureMode.AtMost
            : LayoutMeasureMode.Undefined
      return fn(w, mode)
    })
  }

  setDisplay(display: LayoutDisplay): void {
    this.yoga.setDisplay(display === 'flex' ? Display.Flex : Display.None)
  }

  calculateLayout(width?: number, _height?: number): void {
    this.yoga.calculateLayout(width, undefined, Direction.LTR)
  }
  // ...
}
```

注意 `calculateLayout` 硬编码为 LTR 方向，且忽略 height 参数（终端内容高度由布局自动确定，不做约束）。

### 3.5.3 工厂函数

```typescript
// engine.ts
export function createLayoutNode(): LayoutNode {
  return createYogaLayoutNode()
}

// yoga.ts
export function createYogaLayoutNode(): LayoutNode {
  return new YogaLayoutNode(Yoga.Node.create())
}
```

两层间接调用看似冗余，但 `engine.ts` 是上层代码（`dom.ts`）的唯一入口点。如果需要替换 Yoga 为其他布局引擎，只需修改 `engine.ts` 的返回值。

## 3.6 帧调度机制

### 3.6.1 16ms 节流 + 微任务

帧间隔定义在 `src/ink/constants.ts`：

```typescript
export const FRAME_INTERVAL_MS = 16
```

`Ink` 类的构造函数中设置调度器：

```typescript
const deferredRender = (): void => queueMicrotask(this.onRender);
this.scheduleRender = throttle(deferredRender, FRAME_INTERVAL_MS, {
  leading: true,
  trailing: true
});
```

这是一个两层调度：

1. **lodash throttle（16ms）**：保证渲染频率不超过 60fps。`leading: true` 表示首次调用立即执行；`trailing: true` 表示节流期间的最后一次调用会在冷却���执行。
2. **queueMicrotask**：将实际渲染延迟到微任务队列。这解决了一个关键时序问题——reconciler 的 `resetAfterCommit` 在 React 的 commit 阶段触发，早于 layout 阶段（ref 赋值 + useLayoutEffect）。如果同步渲染，`useDeclaredCursor` 设置的光标位置会比渲染晚一帧。微任务延迟确保 layout effects 先执行。

### 3.6.2 rootNode 上的回调挂载

```typescript
this.rootNode.onRender = this.scheduleRender;
this.rootNode.onImmediateRender = this.onRender;
this.rootNode.onComputeLayout = () => {
  if (this.isUnmounted) return;
  if (this.rootNode.yogaNode) {
    const t0 = performance.now();
    this.rootNode.yogaNode.setWidth(this.terminalColumns);
    this.rootNode.yogaNode.calculateLayout(this.terminalColumns);
    const ms = performance.now() - t0;
    recordYogaMs(ms);
  }
};
```

`onComputeLayout` 在 `resetAfterCommit` 中被调用（commit 阶段内），设置根节点宽度为终端列数，然后触发 Yoga 全树布局。这确保了 `useLayoutEffect` 中读取到的 `getComputedWidth()`/`getComputedHeight()` 是最新值。

### 3.6.3 DOM 级别的渲染触发

对于不经过 React 的 DOM 变更（如滚动），有一个独立的触发路径：

```typescript
export const scheduleRenderFrom = (node?: DOMNode): void => {
  let cur: DOMNode | undefined = node
  while (cur?.parentNode) cur = cur.parentNode
  if (cur && cur.nodeName !== '#text') (cur as DOMElement).onRender?.()
}
```

ScrollBox 的 `scrollTo` / `scrollBy` 直接修改 DOM 节点的 `scrollTop`，然后调用 `scheduleRenderFrom` 沿父链走到根节点触发渲染，完全绕过 React reconciler。这消除了滚动事件的 reconciler 开销。

## 3.7 双缓冲与渲染循环

### 3.7.1 前后缓冲

`Ink` 类持有两个 `Frame` 对象：

```typescript
private frontFrame: Frame;
private backFrame: Frame;
```

- `frontFrame`：上一帧的渲染结果，用于 diff 比较。
- `backFrame`：当前帧的渲染目标。

`onRender` 方法的核心流程：

```typescript
onRender() {
  if (this.isUnmounted || this.isPaused) return;

  // 1. 渲染 DOM 树到 backFrame
  const frame = this.renderer({
    frontFrame: this.frontFrame,
    backFrame: this.backFrame,
    isTTY: this.options.stdout.isTTY,
    terminalWidth,
    terminalRows,
    altScreen: this.altScreenActive,
    prevFrameContaminated: this.prevFrameContaminated
  });

  // 2. 叠加选择和搜索高亮
  if (this.altScreenActive) {
    if (hasSelection(this.selection)) {
      applySelectionOverlay(frame.screen, this.selection, this.stylePool);
    }
    hlActive = applySearchHighlight(frame.screen, this.searchHighlightQuery, this.stylePool);
  }

  // 3. Diff 比较并输出
  const diff = this.log.render(prevFrame, frame, this.altScreenActive, SYNC_OUTPUT_SUPPORTED);

  // 4. 交换缓冲
  this.backFrame = this.frontFrame;
  this.frontFrame = frame;
}
```

交换缓冲后，旧的 `frontFrame` 变成新的 `backFrame`，供下一帧复用（`resetScreen` 会在渲染器中清空它）。

### 3.7.2 渲染器 (renderer.ts)

渲染器负责将 DOM 树绘制到 Screen buffer：

```typescript
export default function createRenderer(
  node: DOMElement,
  stylePool: StylePool,
): Renderer {
  let output: Output | undefined
  return options => {
    const { frontFrame, backFrame, isTTY, terminalWidth, terminalRows } = options
    const width = Math.floor(node.yogaNode.getComputedWidth())
    const height = options.altScreen ? terminalRows : yogaHeight

    // Alt-screen 模式下 screen buffer 高度等于终端行数
    // Main-screen 模式下等于 Yoga 计算出的内容高度
    // ...
  }
}
```

闭包内部复用 `Output` 对象，使 charCache（字符 tokenize + grapheme 聚类的缓存）跨帧持久化——大部分文本行帧间不变，缓存命中率很高。

## 3.8 Screen Buffer 的 Cell 级 Packed Bitmap 格式

这是整个渲染引擎中最底层的数据结构，源码位于 `src/ink/screen.ts`。

### 3.8.1 设计动机

一个 200x120 的终端有 24000 个 cell。如果每个 cell 用 JavaScript 对象表示（`{ char, styleId, width, hyperlink }`），会产生 24000 个堆对象。Claude Code 的解决方案是将所有 cell 数据打包进一个连续的 `Int32Array`，彻底消除 GC 压力。

### 3.8.2 内存布局

每个 cell 占 2 个 Int32（8 字节）：

```
word0 (cells[ci]):     charId (完整 32 位)
word1 (cells[ci + 1]): styleId[31:17] | hyperlinkId[16:2] | width[1:0]
```

位域分配：

| 字段 | 位范围 | 宽度 | 说明 |
|------|--------|------|------|
| charId | word0 全部 | 32 bits | 字符池索引 |
| styleId | word1[31:17] | 15 bits | 样式池索引，bit 0 编码是否对空格可见 |
| hyperlinkId | word1[16:2] | 15 bits | 超链接池索引 |
| width | word1[1:0] | 2 bits | CellWidth 枚举 |

打包和解包的常量定义：

```typescript
const STYLE_SHIFT = 17
const HYPERLINK_SHIFT = 2
const HYPERLINK_MASK = 0x7fff // 15 bits
const WIDTH_MASK = 3          // 2 bits

function packWord1(styleId: number, hyperlinkId: number, width: number): number {
  return (styleId << STYLE_SHIFT) | (hyperlinkId << HYPERLINK_SHIFT) | width
}
```

### 3.8.3 CellWidth 枚举

```typescript
export const enum CellWidth {
  Narrow = 0,      // 普通字符，宽度 1
  Wide = 1,        // CJK/emoji，宽度 2
  SpacerTail = 2,  // 宽字符占据的第二列
  SpacerHead = 3,  // 软换行的行尾占位符
}
```

宽字符（CJK、emoji）在 buffer 中占两个 cell：第一个 cell 持有实际字符（`width = Wide`），第二个是占位符（`width = SpacerTail`，char 为空串）。

### 3.8.4 字符串池化

为避免在 packed array 中存储字符串引用，所有字符串通过池化转为整数索引：

```typescript
export class CharPool {
  private strings: string[] = [' ', '']  // 0 = 空格, 1 = 空串(spacer)
  private ascii: Int32Array = initCharAscii()

  intern(char: string): number {
    if (char.length === 1) {
      const code = char.charCodeAt(0)
      if (code < 128) {
        const cached = this.ascii[code]!
        if (cached !== -1) return cached
        const index = this.strings.length
        this.strings.push(char)
        this.ascii[code] = index
        return index
      }
    }
    const existing = this.stringMap.get(char)
    if (existing !== undefined) return existing
    const index = this.strings.length
    this.strings.push(char)
    this.stringMap.set(char, index)
    return index
  }
}
```

ASCII 字符走数组直接索引（O(1)），非 ASCII 字符走 Map 查找。池在帧间共享，ID 跨 screen buffer 有效，使得 blit 操作可以直接拷贝整数而无需重新 intern。

### 3.8.5 StylePool -- 样式 ID 的 bit 0 技巧

```typescript
intern(styles: AnsiCode[]): number {
  const key = styles.length === 0 ? '' : styles.map(s => s.code).join('\0')
  let id = this.ids.get(key)
  if (id === undefined) {
    const rawId = this.styles.length
    this.styles.push(styles.length === 0 ? [] : styles)
    id = (rawId << 1) | (styles.length > 0 && hasVisibleSpaceEffect(styles) ? 1 : 0)
    this.ids.set(key, id)
  }
  return id
}
```

styleId 的 bit 0 编码该样式是否对空格字符产生可见效果（背景色、反转、下划线等）。这让渲染器在 diff 时能用单次位运算跳过不可见的空格 cell——如果 charId=0（空格）且 `styleId & 1 === 0`（纯前景样式），该 cell 视觉上等价于空白，可以用光标移动替代输出。

### 3.8.6 Screen 创建与双缓冲复用

```typescript
export function createScreen(width, height, styles, charPool, hyperlinkPool): Screen {
  const size = width * height
  const buf = new ArrayBuffer(size << 3)  // 8 bytes per cell
  const cells = new Int32Array(buf)
  const cells64 = new BigInt64Array(buf)  // 同一 buffer 的 64 位视图

  return {
    width, height,
    cells, cells64,
    charPool, hyperlinkPool,
    emptyStyleId: styles.none,
    damage: undefined,
    noSelect: new Uint8Array(size),
    softWrap: new Int32Array(height),
  }
}
```

`cells` 和 `cells64` 是同一 `ArrayBuffer` 的两个视图。`cells`（Int32Array）用于逐字段读写；`cells64`（BigInt64Array）用于批量清零（`cells64.fill(0n)`，一次 fill 操作清除所有 cell，比逐 Int32 清零快 2 倍）。

`resetScreen` 函数用于帧间复用：

```typescript
export function resetScreen(screen, width, height): void {
  const size = width * height
  if (screen.cells64.length < size) {
    // 只在需要时扩容，避免重新分配
    const buf = new ArrayBuffer(size << 3)
    screen.cells = new Int32Array(buf)
    screen.cells64 = new BigInt64Array(buf)
  }
  screen.cells64.fill(EMPTY_CELL_VALUE, 0, size)
  screen.noSelect.fill(0, 0, size)
  screen.softWrap.fill(0, 0, height)
  screen.damage = undefined
}
```

只增不减的分配策略——如果终端缩小了，buffer 不会收缩，只是使用更小的范围。扩大时才重新分配。

### 3.8.7 Damage Tracking

Screen 维护一个 `damage: Rectangle` 跟踪本帧写入的范围：

```typescript
// setCellAt 内部
const damage = screen.damage
if (damage) {
  // 扩展已有范围
  if (minX < damage.x) { damage.width += damage.x - minX; damage.x = minX }
  else if (x >= right) { damage.width = x - damage.x + 1 }
  // ...
} else {
  screen.damage = { x: minX, y, width: x - minX + 1, height: 1 }
}
```

diff 引擎只比较 damage 区域内的 cell，跳过未被写入的区域。稳态下（只有时钟/spinner 更新），damage 可能只有几行，diff 开销接近 O(1)。

## 3.9 核心组件

### 3.9.1 Box

`Box` 是 Ink 的基础布局组件，对应 `<div style="display: flex">`。经过 React Compiler 转换后，它使用 `_c()` 缓存来消除不必要的重新渲染。

核心 JSX 产出是一个 `ink-box` 宿主元素：

```tsx
<ink-box
  ref={ref}
  style={{
    flexDirection, flexGrow, flexShrink, flexWrap,
    ...style
  }}
  tabIndex={tabIndex}
  autoFocus={autoFocus}
  onClick={onClick}
  onKeyDown={onKeyDown}
  onKeyDownCapture={onKeyDownCapture}
  onFocus={onFocus}
  onFocusCapture={onFocusCapture}
  onBlur={onBlur}
  onBlurCapture={onBlurCapture}
  onMouseEnter={onMouseEnter}
  onMouseLeave={onMouseLeave}
>
  {children}
</ink-box>
```

默认值：`flexDirection: 'row'`、`flexGrow: 0`、`flexShrink: 1`、`flexWrap: 'nowrap'`。这意味着 Box 默认水平排列子元素，不增长但可以收缩。

### 3.9.2 Text

`Text` 组件渲染样式化文本，它产出两层宿主元素：

1. 外层 `ink-text`：拥有 Yoga 节点和 `measureFunc`，参与布局。
2. 内层 `ink-virtual-text`（嵌套时降级）：不创建 Yoga 节点，只是样式和文本内容的容器。

样式通过 `textStyles` prop 传递给 `ink-text`：

```typescript
const memoizedStylesForWrap: Record<NonNullable<Styles['textWrap']>, Styles> = {
  wrap: {
    flexGrow: 0,
    flexShrink: 1,
    flexDirection: 'row',
    textWrap: 'wrap'
  },
  // ... 其他 wrap 模式
}
```

支持的文本样式属性：`color`、`backgroundColor`、`bold`/`dim`（互斥）、`italic`、`underline`、`strikethrough`、`inverse`。bold 和 dim 在终端中互斥，TypeScript 类型通过联合类型强制二选一：

```typescript
type WeightProps = {
  bold?: never; dim?: never;
} | {
  bold: boolean; dim?: never;
} | {
  dim: boolean; bold?: never;
}
```

### 3.9.3 ScrollBox

`ScrollBox` 是一个带虚拟滚动的容器。它的 imperative API 完全绕过 React：

```typescript
function scrollMutated(el: DOMElement): void {
  markScrollActivity();
  markDirty(el);
  markCommitStart();
  notify();
  if (renderQueuedRef.current) return;
  renderQueuedRef.current = true;
  queueMicrotask(() => {
    renderQueuedRef.current = false;
    scheduleRenderFrom(el);
  });
}
```

`scrollTo`、`scrollBy` 直接修改 DOM 节点的 `scrollTop`，然后通过 `markDirty` + `scheduleRenderFrom` 触发渲染。`queueMicrotask` 合并同一输入批次中的多次 `scrollBy`（比如 `discreteUpdates` 内的多个滚轮事件），避免 throttle 的 leading edge 在第一个事件就触发渲染。

`pendingScrollDelta` 是分帧排空的滚动增量——快速翻滚不会一次跳到目标位置，而是以 `SCROLL_MAX_PER_FRAME` 的速率逐帧排空，产生平滑的滚动动画效果。方向反转自然抵消（纯累加器，无目标跟踪）。

## 3.10 小结

Claude Code 的终端 UI 渲染引擎是一个高度优化的系统：

- **Reconciler** 适配 React 19，集成 W3C 事件模型和优先级调度。
- **DOM 模型** 极简化，只保留终端渲染所需字段，事件处理器独立存储避免无效 dirty 标记。
- **Yoga 适配层** 通过 LayoutNode 接口隔离布局引擎具体实现。
- **帧调度** 使用 16ms throttle + microtask 双层机制，平衡响应性和吞吐量。
- **Screen Buffer** 使用 packed Int32Array 存储 cell 数据，通过字符串池化、样式池化和 bit-packing 技术消除 GC 压力。
- **Damage Tracking** 将 diff 范围限制在实际变化的区域，稳态下接近零开销。
