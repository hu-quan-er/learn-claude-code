# 06 - Ink 终端渲染引擎深度剖析

---

## 一、Frame 双缓冲机制

### 1.1 Frame 结构

```typescript
type Frame = {
  readonly screen: Screen           // 屏幕缓冲（单元格矩阵）
  readonly viewport: Size           // 终端视口尺寸
  readonly cursor: Cursor           // 光标位置与可见性
  readonly scrollHint?: ScrollHint  // DECSTBM 滚动优化提示
  readonly scrollDrainPending?: boolean
}
```

### 1.2 双缓冲实现

Ink 主类维护两个 Frame 实例：
- **前帧 (frontFrame)** — 当前正在显示的帧
- **后帧 (backFrame)** — 下一帧在此构建

每次 render 循环中交换两个缓冲区。`prevScreen` 保留上一帧的 Screen 对象，用于 diff 计算和 blit 优化。

### 1.3 Screen 存储结构

Screen 对象使用 TypedArray 存储，每个单元格编码：

```typescript
type Cell = {
  char: string       // 内部化的字符 ID
  styleId: number    // 编码样式 + "空格可见性"标志位
  width: CellWidth   // Narrow | Wide | SpacerHead | SpacerTail
  hyperlink: number  // 内部化的超链接 ID
}
```

**StyleId 第 0 位**：编码空格是否有视觉效果（背景/反色/下划线），支持快速跳过不可见空格。

### 1.4 DECSTBM 滚动优化

对于支持 DECSET 1006 的终端：

```typescript
scrollHint = {
  top: number     // 滚动区域顶部行
  bottom: number  // 滚动区域底部行
  delta: number   // 移动行数（正=向上）
}
```

工作流程：
1. `render-node-to-output` 在 ScrollBox 渲染时计算 scrollHint
2. `log-update` 检测 hint 有效性（区域在视口内）
3. 若有效：在 prevScreen 上执行 `shiftRows` 模拟，然后生成单一 DECSTBM+SU/SD+reset 补丁
4. 对比完整 diff，节省大量带宽 — 从渲染整个滚动后内容变为一条 CSI 序列

---

## 二、Yoga Flexbox 布局引擎集成

### 2.1 YogaLayoutNode 适配器

```typescript
class YogaLayoutNode implements LayoutNode {
  readonly yoga: YogaNode

  insertChild(child: LayoutNode, index: number)
  removeChild(child: LayoutNode)
  calculateLayout(width?: number): void  // Direction.LTR only
  setMeasureFunc(fn: LayoutMeasureFunc): void
  getComputedLeft/Top/Width/Height(): number
  getComputedBorder/Padding(edge): number
}
```

### 2.2 文本测量与换行

**两个测量函数**：

1. **ink-text 节点**：
```typescript
measureTextNode(node, width, widthMode) {
  const text = expandTabs(node.nodeValue)

  // Fast path：文本不需换行
  if (stringWidth(text) <= width) return

  // 嵌入式换行处理
  if (text.includes('\n') && widthMode === LayoutMeasureMode.Undefined) {
    const effectiveWidth = Math.max(width, stringWidth(text))
    return measureText(wrapText(text, effectiveWidth, textWrap))
  }

  // Normal path：应用 wordWrap 或 overflowWrap
  return measureText(wrapText(text, width, node.style.textWrap))
}
```

2. **ink-raw-ansi 节点**：预渲染 ANSI 字符串的直接查表，无字符串宽度计算开销

### 2.3 样式应用与脏标记传播

```typescript
function applyStyles(yogaNode, styles, previousStyles) {
  const diff = styleDiff(previousStyles, styles)
  for (const [key, value] of Object.entries(diff)) {
    // 只有变化的属性才调用 Yoga setter
    yogaNode.setWidth(value) / setFlexDirection(value) / ...
  }
  yogaNode.markDirty()
}

// 脏标记从修改节点向上传播到 root
function markDirty(node?: DOMNode) {
  let current = node
  while (current) {
    current.dirty = true
    if (current.yogaNode) current.yogaNode.markDirty()
    current = current.parentNode
  }
}
```

---

## 三、React Reconciler 自定义实现

### 3.1 Reconciler 配置

```typescript
const reconciler = createReconciler({
  // 宿主上下文：追踪是否在 Text 内（禁止嵌套 Box）
  getRootHostContext: () => ({ isInsideText: false }),
  getChildHostContext(parent, type) {
    return { isInsideText: parent.isInsideText || type === 'ink-text' }
  },

  createInstance(type, props, root, hostContext) {
    // ink-text 在 Text 内 → ink-virtual-text（不需要 Yoga 节点）
    const actualType = hostContext.isInsideText && type === 'ink-text'
      ? 'ink-virtual-text'
      : type
    const node = createNode(actualType)
    for (const [key, value] of Object.entries(props)) {
      applyProp(node, key, value)
    }
    return node
  },

  // 关键回调：布局与绘制触发点
  resetAfterCommit(rootNode) {
    rootNode.onComputeLayout?.()  // 1. Yoga 布局
    rootNode.onRender?.()         // 2. 调度渲染
  }
})
```

### 3.2 事件优先级调度

```typescript
class Dispatcher {
  resolveEventPriority(): number {
    if (this.currentEvent) {
      return getEventPriority(this.currentEvent.type)
      // keydown/click/focus → DiscreteEventPriority（同步）
      // mousemove/scroll/resize → ContinuousEventPriority（可打断）
      // 其他 → DefaultEventPriority
    }
    return DefaultEventPriority
  }

  dispatch(target, event) {
    // 两阶段分派：捕获 → 目标 → 冒泡
    const listeners = collectListeners(target, event)
    processDispatchQueue(listeners, event)
  }
}
```

### 3.3 性能追踪集成

```typescript
type FrameEvent = {
  durationMs: number
  phases?: {
    renderer: number     // DOM → yoga → screen buffer
    diff: number         // screen diff → Patch[]
    optimize: number     // patch merge/dedupe
    write: number        // ANSI → stdout
    patches: number      // 优化前 patch 计数
    yoga: number         // yoga.calculateLayout()
    commit: number       // React reconcile
    yogaVisited: number  // layoutNode() 调用次数
    yogaMeasured: number // measureFunc 调用次数
    yogaCacheHits: number
    yogaLive: number     // 当前 Yoga 节点数
  }
}
```

---

## 四、鼠标与键盘事件系统

### 4.1 键盘输入解析（三层状态机）

**文件**: `src/ink/parse-keypress.ts`

```
第 1 层：Tokenizer（termio/tokenize.ts）
  → 输入字符串 → Token 序列 ({type:'sequence'|'text', value})

第 2 层：Paste 检测
  → PASTE_START → 收集 pasteBuffer → PASTE_END → createPasteKey

第 3 层：终端响应与按键解析
  → parseTerminalResponse() (DECRPM, DA1, DA2, OSC)
  → parseMouseEvent() (SGR mouse)
  → parseKeypress() (普通键/特殊键)
```

**支持的编码协议**：
- Kitty Keyboard Protocol (CSI u): `ESC[codepoint;modifier u`
- xterm modifyOtherKeys: `ESC[27;modifier;keycode u`
- SGR Mouse: `ESC[<button;col;row M/m`
- 传统序列: `ESC[A` (up) 等

**修饰符解码**：
```typescript
function decodeModifier(m: number) {
  const mod = m - 1
  return {
    shift: !!(mod & 1),
    meta:  !!(mod & 2),
    ctrl:  !!(mod & 4),
    super: !!(mod & 8)
  }
}
```

### 4.2 鼠标 Hit-Test 算法

**文件**: `src/ink/hit-test.ts`

```typescript
function hitTest(node: DOMElement, col: number, row: number): DOMElement | null {
  const rect = nodeCache.get(node)
  if (!rect || col < rect.x || col >= rect.x + rect.width) return null

  // 逆序遍历子节点（后绘制的在上方）
  for (let i = node.childNodes.length - 1; i >= 0; i--) {
    const hit = hitTest(node.childNodes[i], col, row)
    if (hit) return hit  // 最深的命中节点
  }
  return node
}
```

### 4.3 鼠标悬停追踪

```typescript
function dispatchHover(root, col, row, hovered: Set<DOMElement>) {
  const next = new Set<DOMElement>()
  let node = hitTest(root, col, row)

  // 上升链收集所有有 hover 处理器的祖先
  while (node) {
    if (node._eventHandlers?.onMouseEnter || node._eventHandlers?.onMouseLeave) {
      next.add(node)
    }
    node = node.parentNode
  }

  // 差分：离开/进入
  for (const old of hovered) {
    if (!next.has(old)) { hovered.delete(old); old._eventHandlers?.onMouseLeave?.() }
  }
  for (const n of next) {
    if (!hovered.has(n)) { hovered.add(n); n._eventHandlers?.onMouseEnter?.() }
  }
}
```

---

## 五、文本选择系统

### 5.1 选择状态

```typescript
type SelectionState = {
  active: boolean        // 选择是否激活
  startCell: CellCoord   // 起始单元格
  endCell: CellCoord     // 结束单元格
  reversed: boolean      // 是否反向（从下到上）
}
```

### 5.2 反向覆盖层（Inverse Overlay）

```typescript
function applyInverseToSelection(screen, stylePool, selection) {
  const [start, end] = normalizeSelection(selection.startCell, selection.endCell)

  for (let row = start.row; row <= end.row; row++) {
    const colStart = row === start.row ? start.col : 0
    const colEnd = row === end.row ? end.col + 1 : screen.width

    for (let col = colStart; col < colEnd; col++) {
      const cell = cellAtIndex(screen, row * screen.width + col)
      const newStyleId = stylePool.withInverse(cell.styleId)
      setCellStyleId(screen, col, row, newStyleId)
    }
  }
}
```

### 5.3 noSelect 位图

由 `<NoSelect>` 组件标记，复制时排除行号、diff sigils 等装饰字符。

---

## 六、滚动实现

### 6.1 滚动状态管理

```typescript
type ScrollState = {
  scrollTop: number            // 当前滚动偏移
  pendingScrollDelta: number   // 累计的待处理滚动
  scrollHeight: number         // 内容总高度
  scrollViewportHeight: number // 视口高度
  stickyScroll: boolean        // 内容增长时自动粘底
  scrollAnchor?: { el, offset }// scrollToElement 锚点
}
```

### 6.2 DECSTBM 快速路径

满足条件时（alt-screen + 滚动区域在视口内）：

```
Step 1: 检测滚动 → 计算 cumHeightShift
Step 2: prevScreen 上执行 shiftRows 模拟
Step 3: 构造单一 DECSTBM 补丁
        setScrollRegion(top, bottom) + csiScrollUp/Down(delta) + RESET
Step 4: 仅 diff 新进入的边缘行，复用 blit 内容
```

### 6.3 视口裁剪（Viewport Culling）

```typescript
// ScrollBox 内容高度 = 10000 行，视口 = 30 行
for (const child of content.childNodes) {
  const top = child.yogaNode.getComputedTop()
  const height = child.yogaNode.getComputedHeight()

  if (top + height <= scrollTop || top >= scrollTop + viewportHeight) {
    dropSubtreeCache(child)  // 离屏：跳过整个子树
    continue
  }
  renderNodeToOutput(child, ...)  // 可见：正常渲染
}
```

**O(visible) 性能**：即使有 5000 个离屏 children，也只渲染 <100 个可见行。

---

## 七、ANSI 差异渲染优化算法

### 7.1 增量 Diff 核心

```typescript
function diffEach(prevScreen, nextScreen, callback) {
  for (let row = 0; row < h; row++) {
    for (let col = 0; col < w; col++) {
      const prev = cellAtIndex(prevScreen, idx)
      const next = cellAtIndex(nextScreen, idx)
      if (prev.char !== next.char ||
          prev.styleId !== next.styleId ||
          prev.width !== next.width) {
        callback(col, row)  // 只处理变化的单元格
      }
    }
  }
}
```

### 7.2 补丁类型

```typescript
type Patch =
  | { type: 'stdout'; content: string }        // 原始 ANSI
  | { type: 'cursorMove'; x: number; y: number } // 相对移动
  | { type: 'cursorTo'; col: number }          // 行内移位
  | { type: 'styleStr'; str: string }          // 样式转换
  | { type: 'hyperlink'; uri: string }         // OSC 8
  | { type: 'clear'; count: number }           // 清除 N 行
  | { type: 'cursorHide' | 'cursorShow' }
```

### 7.3 优化器（单遍处理）

**文件**: `src/ink/optimizer.ts`

```
规则：
1. 删除空 patch（如空 stdout）
2. 合并连续 cursorMove: [x1,y1] + [x2,y2] → [x1+x2, y1+y2]
3. 覆盖连续 cursorTo（只保留最后一个）
4. 连接相邻 styleStr
5. 删除重复 hyperlink
6. 抵消 cursorHide/cursorShow 对
```

---

## 八、对象池与内存优化

### 8.1 字符串内部化（CharPool）

```typescript
class CharPool {
  private ascii: Int32Array   // ASCII 快速路径：O(1) 查表，命中率 >90%
  private stringMap: Map<string, number>

  intern(char: string): number {
    if (char.length === 1 && char.charCodeAt(0) < 128) {
      const idx = this.ascii[char.charCodeAt(0)]
      if (idx !== -1) return idx
    }
    return this.stringMap.get(char) ?? this.addNew(char)
  }
}
```

**效果**：20 万单元格屏幕的字符内存从 2MB+ 降至 ~100KB。

### 8.2 StylePool 三级缓存

```typescript
class StylePool {
  private ids: Map<string, number>           // 样式字符串 → ID
  private transitionCache: Map<number, string> // (fromId, toId) → ANSI 转换
  private inverseCache: Map<number, number>    // baseId → withInverse(baseId)
  private currentMatchCache: Map<number, number>
}
```

首次计算 `(fromId, toId)` 的 transition 后，后续相同转换 `Map.get` O(1) 命中。选择/搜索高亮变化时缓存命中率 >99%。

### 8.3 Output charCache

```typescript
class Output {
  private charCache: Map<string, ClusteredChar[]>  // 行文本 → grapheme 簇化结果

  reset(width, height, screen) {
    if (this.charCache.size > 16384) this.charCache.clear()  // 防 OOM
  }
}
```

静态行（代码、提示、边框）缓存命中率 >80%，避免每帧的 tokenize + grapheme segmentation 开销。

### 8.4 节点缓存（WeakMap）

```typescript
const nodeCache = new WeakMap<DOMElement, CachedLayout>()

type CachedLayout = {
  x: number; y: number; width: number; height: number
  top?: number  // yoga-local 坐标，用于 ScrollBox 视口裁剪
}
```

WeakMap 优势：DOM 节点被垃圾回收时，缓存自动清理（零内存泄漏）。

### 8.5 Blit 与 Diff 配合

```typescript
// 无变化的节点从 prevScreen blit（直接内存复制）
if (!node.dirty && cached) {
  output.blit(prevScreen, cached.x, cached.y, cached.width, cached.height)
  // 跳过此节点及所有后代的重新渲染
}
```

---

## 九、完整渲染管道

```
React.render(app)
  ↓
reconciler.updateContainer()    [React Fiber work + commit]
  ↓ resetAfterCommit
rootNode.onComputeLayout()
  ↓
Yoga.calculateLayout()          [DOM → flex constraints → positions/sizes]
  ↓
rootNode.onRender()              [throttled @ 33ms / 30fps]
  ↓
createFrame()
  ├─ renderNodeToOutput(rootNode, output)
  │   ├─ ink-box: 背景、padding、border → 递归 children
  │   ├─ ink-text: 测量 + 写字符簇到 screen buffer
  │   ├─ ink-raw-ansi: 放置预渲染 ANSI 块
  │   ├─ ScrollBox: blit+shift+edge-render 或 full render
  │   └─ 缓存 rects 到 nodeCache
  │
  ├─ output.get() → screen cells
  │
  ├─ diff(prevScreen, nextScreen) → Patch[]
  │
  ├─ optimize(patches) → 合并/去重/抵消
  │
  └─ writeDiffToTerminal() → ANSI → stdout

(swap front/back frames, schedule next if needed)
```

---

## 十、性能特性总结

| 特性 | 实现 | 效果 |
|------|------|------|
| 双缓冲 | prevScreen + frontFrame/backFrame | 支持 diff、blit、DECSTBM |
| 布局引擎 | Yoga + 自定义文本测量 | 完整 Flexbox、自适应换行 |
| 事件系统 | 自定义 Dispatcher + 优先级调度 | 捕获/冒泡、同步/可打断 |
| 视口裁剪 | ScrollBox 只渲染可见节点 | O(visible) 性能 |
| 字符内部化 | ASCII O(1) 快速路径 | 内存降低 95%+ |
| 样式缓存 | 三级缓存（transition/inverse/match） | 命中率 >99% |
| ANSI 差异 | 增量 diff + patch optimizer | 800 行改 1 行只发 diff |
| 滚动优化 | DECSTBM 硬件滚动 + blit | <16ms 帧时间 |
