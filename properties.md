# 扫雷 - 网页版 (Minesweeper Web Edition)

## 项目概述

纯 HTML/CSS/JS 单文件扫雷游戏，零依赖，跨平台适配（桌面 + 手机 + 平板），直接在浏览器中打开 `index.html` 即可运行。

---

## 文件结构

```
mines/
└── index.html   # 单文件，包含内联 CSS 和 JS
```

---

## 技术栈

- **HTML5**：单页面，无框架
- **CSS3**：CSS Grid 布局，CSS 变量（`--cell-size`、`--cell-font`），`@keyframes` 动画，`clamp()` 响应式字号
- **JavaScript（ES6）**：所有逻辑封装在一个 IIFE `(function() { ... })()` 中，全局仅暴露 3 个函数（`newGame`、`setDifficulty`、`toggleMode`）给 HTML 的 `onclick` 属性

---

## 难度系统与自适应棋盘

### 原始难度定义

| 难度 | 原始行数 | 原始列数 | 原始地雷数 | 雷密度 |
|------|---------|---------|----------|--------|
| easy (简单) | 9 | 9 | 10 | ~12.3% |
| medium (中等) | 16 | 16 | 40 | ~15.6% |
| hard (困难) | 16 | 30 | 99 | ~20.6% |

### 自适应算法 (`adaptCfg`)

不直接使用原始尺寸，而是根据屏幕可用空间动态计算实际棋盘大小：

```
常量：
  TARGET_CELL = 38px   # 手指舒适点击目标尺寸
  MIN_CELL    = 30px   # 绝对最小格子尺寸
  CHROME_W    = 40px   # 界面横向装饰占用(game padding + border)
  CHROME_H    = 160px  # 界面纵向装饰占用(header + 难度栏 + hint + padding)

计算步骤：
  1. availW = window.innerWidth  - CHROME_W
     availH = window.innerHeight - CHROME_H

  2. cols = min(原始列数, floor(availW / TARGET_CELL))   ← 行列各自独立适配
     rows = min(原始行数, floor(availH / TARGET_CELL))   ← 不超过原始值

  3. density = 原始地雷数 / (原始列数 × 原始行数)
     mines   = max(1, floor(cols × rows × density))

  4. cellW  = floor(availW / cols)
     cellH  = floor(availH / rows)
     cellSz = min(cellW, cellH, TARGET_CELL)
     if cellSz < MIN_CELL → cellSz = MIN_CELL

返回：{ rows, cols, mines, cellSize }
```

- **关键特性**：行列各自独立适配，不会因为宽度受限而浪费高度空间（例如手机竖屏 390×844，宽度方向 9 列，高度方向可容纳 17 行）
- 棋盘不会超过原始难度设计的规模
- 地雷按原始雷密度等比缩放
- 难度按钮文字动态显示实际尺寸（如 "困难 10×6"），由 `updateLabels()` 在页面加载和窗口 resize 时刷新

---

## 界面布局

```
┌──────────────────────────────────────┐
│  [雷数计数器] [🚩/⛏️切换] [🙂] [计时器] │  ← .header (flex, gap: 8px)
│  [简单 n×m] [中等 n×m] [困难 n×m]     │  ← .difficulty-bar (flex, gap: 4px)
├──────────────────────────────────────┤
│           棋盘区域 (.board-wrap)       │  ← overflow-x: auto, border 3px
│   CSS Grid, 单元格宽度 = --cell-size  │
├──────────────────────────────────────┤
│     操作提示文字（桌面/移动切换）       │  ← .hint
└──────────────────────────────────────┘
```

### 组件样式规格

| 组件 | 尺寸 | 配色 |
|------|------|------|
| 计数器/计时器 | `clamp(20px, 5vw, 28px)`，黑底红字，monospace | `#222` 背景，`#f00` 文字 |
| 表情按钮 | `clamp(36px, 8vw, 42px)` | `#c0c0c0` 背景，3D 凸起边框 |
| 模式切换按钮 | 同表情按钮大小 | 挖掘模式 `#c8d8e8`，旗子模式 `#f0d0d0` |
| 难度按钮 | `clamp(11px, 2.5vw, 14px)`，`white-space: nowrap` | `#c0c0c0`，active 态 `#b0b0b0` |
| 提示文字 | `clamp(11px, 2.5vw, 13px)` | `#555` |

### 3D 边框效果

所有按钮和单元格使用经典的 Windows 风格 3D 边框：
- 默认：`border: 2px solid #fff`，右边和下边用 `#808080`（亮边 + 暗边 = 凸起效果）
- `:active` 或 `:pressed`：交换亮暗边方向（暗边 + 亮边 = 凹陷效果）

---

## 格子状态与渲染

### 数据模型

```js
board[r][c] = {
  mine: Boolean,    // 是否是地雷
  num:  Number,     // 周围地雷数 (0-8)
  rev:  Boolean,    // 是否已翻开
  flg:  Boolean     // 是否已插旗
}
```

### 视觉状态

| 状态 | CSS class | 显示内容 | 背景色 |
|------|-----------|---------|--------|
| 未翻开（默认） | `cell` | 空 | `#c0c0c0`，3D 凸起边框 |
| 已翻开（空） | `cell revealed` | 空 | `#b0b0b0`，凹陷边框 |
| 已翻开（数字） | `cell revealed n{N}` | 数字 1-8 | `#b0b0b0`，凹陷边框 |
| 已翻开（地雷） | `cell revealed mine-hit` | 💣 | `#f00 !important` |
| 已插旗 | `cell` | 🚩 | 默认背景 |
| 错误插旗 | `cell flag-wrong` | 🚩 | `#f88`（仅游戏结束显示） |

数字颜色映射：
- n1: `#00f` (蓝)
- n2: `#008000` (绿)
- n3: `#f00` (红)
- n4: `#000080` (深蓝)
- n5: `#800000` (暗红)
- n6: `#008080` (青)
- n7: `#000` (黑)
- n8: `#808080` (灰)

### 渲染函数 (`render`)

1. 清空 `#board` 的 `innerHTML`
2. 设置 `gridTemplateColumns/Rows` 为 `repeat(N, var(--cell-size))`
3. 用 `for (let r ...)` 循环（**必须用 `let`**，不能用 `var`，因为事件闭包需要块作用域绑定）
4. 每个格子创建 `div.cell`，存入 `dataset.r` 和 `dataset.c`
5. 绑定事件监听器：`click`、`dblclick`、`contextmenu`；如果是触摸设备，额外绑定 `touchstart`、`touchend`、`touchmove`（均 `passive: false`）
6. 调用 `delAll()` 初始化所有格子的视觉状态

### 更新函数

- `delOne(r, c)`：根据 `board[r][c]` 状态设置单个格子的 className 和 textContent。**重要：每次调用先 `el.className = 'cell'` 重置所有 class，然后按状态添加对应 class**
- `delAll()`：双重 for 循环调用 `delOne` 更新所有格子

---

## 游戏核心逻辑

### 新游戏 (`newGame`)

1. 调用 `adaptCfg(D[diff])` 获取自适应配置
2. 重置全局状态（`board`、`over`、`won`、`first`、`flags`、计时器）
3. 表情设为 🙂
4. 调用 `applySize()` 设置 CSS 变量 `--cell-size` 和 `--cell-font`
5. 初始化 `board` 二维数组（所有格子的 `mine`/`num`/`rev`/`flg` 均初始化为 false/0）
6. 调用 `render()` 渲染棋盘
7. **地雷尚未放置**——等首次点击时由 `placeMines` 放置，保证首次点击安全

### 布雷 (`placeMines(sr, sc)`)

参数 `sr, sc` 是首次点击的安全坐标（-1 表示无安全区，用于右击首次操作时先布雷但无安全区保证）：

1. 构建安全区：以 `(sr, sc)` 为中心的 3×3 区域（含自身）
2. 构建候选池：所有不在安全区的坐标，打乱（Fisher-Yates 洗牌）
3. 取前 M 个坐标设置为地雷
4. 遍历所有非雷格子，计算周围 8 方向的地雷数量，存入 `num`

### 翻开 (`leftClick(r, c)`)

1. 如果游戏结束或格子已翻开/已插旗 → 直接返回
2. 如果是首次点击 → 调用 `placeMines(r, c)` 放置地雷，启动计时器
3. 如果踩雷 → 设置 `over=true`，表情变为 😵，翻开所有地雷，返回
4. 如果 `num === 0` → 调用 `revealEmpty(r, c)` 洪水填充
5. 否则 → 翻开该格
6. 调用 `delAll()` 和 `chkWin()`

### 洪水填充 (`revealEmpty(r, c)`)

基于栈的 BFS：
1. 起始坐标压栈
2. 循环弹栈：如果格子已翻开或已插旗 → `continue`
3. 翻开该格，如果 `num === 0`，将周围 8 格中未翻开且未插旗的坐标压栈
4. 栈空则结束

### 插旗 (`rightClick(r, c)`)

1. 如果游戏结束或格子已翻开 → 返回
2. 如果是首次点击 → 调用 `placeMines(-1, -1)`（无安全区布雷），启动计时器
3. 切换 `flg` 状态
4. 更新全局 `flags` 计数
5. 更新计数器和单格显示
6. 调用 `chkWin()`

### 快速翻开 / Chord (`dblClick(r, c)`)

适用于桌面双击或手机点击已翻开数字格：
1. 如果游戏结束、格子未翻开、或 `num === 0` → 返回
2. 统计周围 8 格的旗子数
3. 如果旗子数 ≠ 该格数字 → 返回（不满足 Chord 条件）
4. 翻开周围所有未翻开且未插旗的格子
5. 如果其中有地雷 → 游戏结束（踩雷）
6. 如果非零格子 → 直接翻开；如果零格 → 递归洪水填充
7. 调用 `chkWin()`

### 胜利判定 (`chkWin()`)

两种条件任意满足其一即获胜：
1. **所有非雷格子都已翻开**（无需全部插旗）
2. **所有地雷都被正确标记且无多余旗子**（即 `flags === M` 且每个雷都被标记，每个标记都对应一个雷）

获胜后：
- `won = true, over = true`
- 停止计时器
- 表情变为 😎
- 翻开所有未翻开的地雷
- 自动标记所有未标记的地雷
- 更新计数器
- 触发 `confetti()` 庆祝动画

---

## 移动端触摸交互

### 检测方式

```js
var IS_TOUCH = ('ontouchstart' in window) || (navigator.maxTouchPoints > 0);
```

只有检测到触摸设备时，才绑定 `touchstart/touchend/touchmove` 事件。

### 旗子模式切换

顶栏 `⛏️/🚩` 按钮（`toggleMode()`）切换 `flagMode` 全局变量：

| 模式 | 图标 | 背景色 | 点击行为 | 长按行为 |
|------|------|--------|---------|---------|
| 挖掘模式（默认） | ⛏️ | `#c8d8e8` | 翻开格子 | 插旗/取消 |
| 旗子模式 | 🚩 | `#f0d0d0` | 插旗/取消 | 翻开格子 |

- `cellClick()` 中检查 `flagMode` 决定调用 `leftClick` 还是 `rightClick`
- `cellRightAction()` 用于长按和系统 contextmenu 处理，根据 `flagMode` 执行反向操作

### 长按机制

**时间阈值**：`LP_MS = 400ms`

**touchstart (`tsStart`)**：
1. 记录触摸起始坐标 `(tsX, tsY)` 和长按目标格子 `lpCell`
2. 重置所有长按状态标志
3. 如果格子已翻开 → 不启动长按（直接返回）
4. 调用 `addRing()` 显示长按进度环
5. 启动 400ms 定时器，到期后：
   - 如果已被 Android `contextmenu` 抢先处理（`lpHandled`）→ 清除环并返回
   - 设置 `lpDone = true, skipClick = true`
   - 清除环，触发振动
   - 调用 `cellRightAction()`（根据模式执行反向操作）

**touchmove (`tsMove`)**：
- 如果手指移动超过 12px（任一方向）→ 取消长按，清除环，重置状态

**touchend (`tsEnd`)**：
- 清除定时器和环
- 如果长按已完成（`lpDone`）→ 调用 `e.preventDefault()` 阻止后续 click 事件，重置状态

**skipNextClick 标志**：
- 长按完成后设置 `skipClick = true`
- `cellClick()` 首行检查此标志，如果为 true 则直接返回（不执行任何操作）
- 目的：防止长按后手指抬起触发的 click 事件执行意外操作（如翻开刚标记的格子）

### 长按进度环

用 CSS class `cell.pressing` + `@keyframes press-grow` 实现：

```css
.cell.pressing {
  animation: press-grow 0.4s linear forwards;
  z-index: 10;
}
@keyframes press-grow {
  0%   { box-shadow: 0 0 0 0 rgba(74,144,217,0.5); }
  100% { box-shadow: 0 0 0 12px rgba(74,144,217,0.7); background: #c8d0e0; }
}
```

- 蓝色半透明 box-shadow 从 0 向外扩散到 12px（`rgba(74,144,217,0.7)`）
- 背景从 `#c0c0c0` 过渡到 `#c8d0e0`（微蓝）
- `z-index: 10` 确保光晕不被相邻格子裁剪
- 动画时长 400ms，与长按阈值一致
- 手指抬起或长按完成时，移除 `pressing` class，动画自然终止

### Android contextmenu 竞态处理

**问题**：Android 系统在长按时会额外触发 `contextmenu` 事件，导致两次触发 `onRightClick`。

**解决方案**：`contextmenu` 事件监听器与 `touchstart` 定时器之间有双向守卫：

```
contextmenu 事件处理：
  if (lpDone) return;                      // 定时器已处理，跳过
  clearTimeout(lpTimer);                    // 清除定时器
  lpHandled = true; lpDone = true;          // 标记已处理
  skipClick = true; clearRing();            // 防止后续 click
  cellRightAction(r, c);                    // 执行反向操作

定时器回调：
  if (lpHandled) { clearRing(); return; }   // contextmenu 已处理，跳过
  lpDone = true; ...                        // 正常执行
```

- `contextmenu` 先到 → 清除定时器，自行处理
- 定时器先到 → 设置 `lpDone`，`contextmenu` 事件到来时检测到 `lpDone` 直接返回

### 振动反馈

长按完成时，如果设备支持，调用 `navigator.vibrate(20)`（20ms 短震）。

### 触摸设备上点击已翻开数字格

在 `cellClick()` 中，如果 `IS_TOUCH && 格子已翻开 && num > 0`，自动调用 `dblClick()`（Chord），因为手机上没有真正的双击操作。

### 提示文字

- 桌面端：`<span class="d">左键翻开 | 右键插旗 | 双击数字快速翻开</span>`
- 移动端：`<span class="m">⛏点击翻开/长按插旗 | 🚩点击插旗/长按翻开 | 点数字快速翻开</span>`
- 页面加载时根据 `IS_TOUCH` 切换显示/隐藏

---

## 庆祝动画 (`confetti`)

获胜时触发：

1. 分 80 次，每次间隔 30ms，在 `#confetti` 容器中创建 `div.confetti-piece`
2. 每个粒子随机属性：
   - 形状：■ ● ▲ ★ ♦ 五选一
   - 颜色：10 种荧光色随机
   - 水平位置：0-100% 随机
   - 起始垂直位置：屏幕上方 -2% 到 -12%
   - 字号：10px-26px 随机
   - 动画时长：2s-4s 随机
3. CSS 动画 `confetti-fall`：从顶部旋转下落 720°，缩放到 0.3，透明度从 1 到 0
4. 每个粒子 4 秒后自动移除

---

## 响应式与自适应

### CSS 变量

```css
:root {
  --cell-size: 30px;  /* 初始值，会被 JS 动态覆盖 */
  --cell-font: 15px;  /* 字号 = cellSize × 0.5，最小 10px */
}
```

### 窗口 resize 处理

监听 `window.resize`：
1. 调用 `updateLabels()` 刷新难度按钮文字
2. 重新调用 `adaptCfg()` 计算新配置
3. 重置棋盘状态（雷区重新洗牌）
4. 重新渲染

### Viewport meta

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
```

`user-scalable=no` 防止双击缩放干扰游戏操作。`touch-action: manipulation` 在 `.cell` 上用于同样目的。

### 棋盘横向溢出

`.board-wrap` 设置 `overflow-x: auto` 和 `-webkit-overflow-scrolling: touch`，极小屏幕时可横向滚动。

---

## 全局 JS 变量清单

所有变量在 IIFE 内部，不污染全局命名空间。以下为内部变量：

| 变量 | 类型 | 说明 |
|------|------|------|
| `D` | object | 原始难度配置 |
| `IS_TOUCH` | bool | 是否为触摸设备 |
| `LP_MS` | number | 长按阈值 (400) |
| `TARGET_CELL` | number | 目标格子尺寸 (38) |
| `MIN_CELL` | number | 最小格子尺寸 (30) |
| `CHROME_W` | number | 横向装饰占用 (40) |
| `CHROME_H` | number | 纵向装饰占用 (160) |
| `diff` | string | 当前难度 ('easy'/'medium'/'hard') |
| `R, C, M` | number | 当前游戏的行数、列数、雷数 |
| `cellSz` | number | 当前格子尺寸 |
| `board` | array | 棋盘数据 (二维数组) |
| `over` | bool | 游戏是否结束 |
| `won` | bool | 是否获胜 |
| `first` | bool | 是否首次点击 |
| `timerId` | number | 计时器 interval ID |
| `sec` | number | 已过秒数 |
| `flags` | number | 已插旗数量 |
| `flagMode` | bool | 旗子模式开关 |
| `lpTimer` | number | 长按定时器 ID |
| `lpDone` | bool | 长按是否已完成 |
| `lpCell` | object | 当前长按的格子坐标 |
| `lpHandled` | bool | 长按是否被 contextmenu 抢先处理 |
| `tsX, tsY` | number | 触摸起始坐标 |
| `skipClick` | bool | 是否跳过下一次 click 事件 |
| `pressCell` | DOM | 当前显示进度环的格子元素 |

### 暴露到 `window` 的 3 个函数

（仅在 inline `onclick` 属性中使用）：

- `window.newGame`
- `window.setDifficulty`
- `window.toggleMode`

---

## 关键边角情况

1. **首次点击必安全**：地雷在首次点击后才布置，且以点击坐标为中心的 3×3 区域不会布雷
2. **右击首操作无安全区保护**：`placeMines(-1, -1)` 不建立安全区，地雷可能出现在任何位置
3. **旗子模式下点击已插旗格**：`rightClick` 会切换旗子状态（取消插旗），不会翻开
4. **游戏结束后所有交互锁死**：`leftClick`、`rightClick`、`dblClick` 首行均检查 `over` 标志
5. **Chord 错误导致踩雷**：`dblClick` 如果旗子标错导致翻开地雷，游戏结束，棋盘翻出所有地雷
6. **获胜后自动处理未标记雷**：胜利时如果存在未标记的地雷，自动翻开并标记它们，确保视觉完整
7. **`updateCell` 中旗子错误标记**：仅在 `over && !mine` 时才显示 `flag-wrong` 样式
8. **闭包变量**：`render()` 中循环必须用 `let` 而非 `var`，确保事件监听器闭包中 `r`、`c` 正确绑定
