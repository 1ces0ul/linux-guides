# 我对 Rijan 的改造：从简约窗口管理器到日用完全体

> Rijan 是 [Isaac Freund](https://codeberg.org/ifreund)（River compositor 作者）用 Janet 语言编写的窗口管理器，运行在 River compositor 之上。它的设计哲学和 River 一脉相承：简洁、正确、可编程。
>
> 这篇文章记录我基于 Rijan 原版的改造过程。所有改动都建立在 Isaac 优秀的架构之上——他用不到 700 行 Janet 代码就写出了一个完整的平铺窗口管理器，这种代码密度和设计品味令人敬佩。
>
> 原始仓库：https://codeberg.org/ifreund/rijan
> 我的改动版：https://github.com/1ces0ul/config-river.0.4.0-dev

## 目录

- [Rijan 是什么](#rijan-是什么)
- [原版做了什么](#原版做了什么)
- [我改了什么](#我改了什么)
- [逐项详解](#逐项详解)
- [改动对照表](#改动对照表)
- [感想](#感想)

---

## Rijan 是什么

River 是一个 Wayland compositor（合成器），负责显示、输入、协议等底层工作。但 River 本身不管窗口怎么排列、怎么切换——这些交给外部的窗口管理器。

Rijan 就是这个窗口管理器。它通过 River 的 `river-window-management` 协议与 compositor 通信，用 Janet 语言编写，天然支持运行时热修改（通过内置的 netrepl 服务器）。

这种 compositor + WM 分离的架构，在 Wayland 世界里是相当前卫的设计。大多数 Wayland 合成器（Sway、Hyprland）都是把窗口管理逻辑内建的，River + Rijan 的分离让用户可以用脚本语言完全重写窗口管理行为，而不用碰 C/Zig。

## 原版做了什么

Isaac 的原版 Rijan 是一个极简但完整的平铺窗口管理器，约 650 行代码，包含：

- **单一布局**：左侧 master + 右侧 stack，经典 dwm 风格
- **浮动窗口**：支持鼠标拖动和缩放
- **全屏**：基本的全屏切换
- **多显示器**：tag 跟随 output 的模型
- **子窗口**：通过 parent 事件识别，自动浮动
- **背景管理**：用 single-pixel-buffer 渲染纯色背景
- **REPL**：运行时可通过 Unix socket 连接调试

原版的窗口管理逻辑非常简洁。`window/manage` 函数只有 20 行，`seat/manage` 不到 50 行。布局函数 `wm/layout` 用函数式管道写成，简洁到有些优雅。

但作为日用，它还缺不少东西。

## 我改了什么

我的改动大致分为六个方面：

### 1. 三种布局引擎

原版只有一种布局：左 master 右 stack。我加了三种布局，可以按 output 独立切换：

- **master-stack**（大幅增强）：支持四个方向（左/右/上/下），可配置 nmaster 数量
- **scroller**：类似 PaperWM / niri 的滚动布局，聚焦窗口居中，两侧窗口延伸，超出屏幕自动隐藏
- **grid**：网格布局，自动计算行列，最后一行居中对齐

对应新增的 action：
```janet
(action/set-layout :tile)
(action/set-layout :scroller)
(action/set-layout :grid)
(action/set-tile-location :left)   # master 在左/右/上/下
(action/switch-to-previous-layout) # 快速切回上一个布局
```

### 2. 窗口尺寸约束（clamp-to-hints）

原版的布局不考虑窗口的 min/max 尺寸提示。有些应用（比如终端模拟器）有固定的字符网格，强制拉伸会导致内容错位。

我加了 `clamp-to-hints` 函数：布局先分配空间（layout box），然后根据窗口的尺寸提示约束实际大小，居中放置，并用 `set-clip-box` 裁剪溢出部分。

### 窗口装饰与边框绘制修复（decoration-hint）

**问题：** 所有窗口都没有 River 绘制的服务端边框 (SSD)。

**根因：** `river_layout_manager_v1` 协议中的 `decoration_hint` 枚举值在 Janet 的 Wayland 绑定层中会被自动转换为 keyword：

| 协议整数 | Janet keyword | 含义 | 典型窗口 |
|:---------|:--------------|:-----|:---------|
| `0` | `:only-supports-csd` | 只支持 CSD | Telegram, Chrome, 无 MOTIF 的 X11 应用 |
| `1` | `:prefers-csd` | 偏好 CSD | WeChat (MOTIF `no_border`) |
| `2` | `:prefers-ssd` | 偏好 SSD | GTK4 应用, 有 MOTIF 装饰的 X11 应用 |
| `3` | `:no-preference` | 无偏好 | 仅 XDG toplevel |

然而代码中用整数 `(= hint 0)` 与 keyword 比较，永远为 false，导致边框逻辑全部失效。

**修复：** 将所有整数比较改为 keyword 比较，默认回退值从 `3` 改为 `:no-preference`。

**修复后的边框绘制决策：**

| 窗口类型 | 画边框？ | 装饰模式 |
|:---------|:---------|:---------|
| 子窗口 (`managed-as-child`) | ✗ 不画 | 无 |
| 独立顶层 + focused (hint 2/3) | ✓ 画 (focused 颜色) | `use-ssd` |
| 独立顶层 + unfocused (hint 2/3) | ✓ 画 (normal 颜色) | `use-ssd` |
| CSD 窗口 (hint 0/1) | ✗ 不画 | `use-csd` |
| 全屏窗口 | ✓ 画* | `use-ssd` |

\*协议规定全屏时边框不显示。

### Rijan 窗口识别与绘制流程概览

```
River 事件循环
│
├── [:window obj] → window/create
│   创建窗口, :new=true, 注册事件(app-id, parent, decoration-hint...)
│
├── [:manage-start] → wm/manage
│   ├── 清理已关闭窗口
│   ├── window/manage (仅 :new)
│   │   ├── parent-event? YES → 子窗口 (float, 不调用 use-ssd/csd)
│   │   └── parent-event? NO  → 独立窗口
│   │       ├── decoration-hint → use-csd 或 use-ssd
│   │       ├── is-partially-fixed? → float/tile
│   │       └── 分配 tag
│   ├── seat/manage (焦点)
│   ├── wm/layout (仅 tiled 窗口)
│   │   ├── :tile → master-stack
│   │   ├── :scroller → scroller
│   │   └── :grid → grid
│   └── wm/show-hide
│
└── [:render-start] → wm/render
    └── window/render
        ├── 初始定位 (parent居中/output居中/layout)
        ├── Z序 (子窗口在parent上方)
        ├── Border (managed-as-child→无, CSD→无, SSD→有)
        └── Clip box (tiled裁剪)
```

**border-width (默认 2px) 参与的计算：**
`set-position`(+bw), `propose-dimensions`(-2×bw), `center-on-output`(-2×bw), 子窗口定位(+bw), `clamp-to-hints`, `clip-box`, `float-move/resize/snap`

> 注意：子窗口在 manage 阶段直接调用 `(:propose-dimensions ...)` 不经过 `window/propose-dimensions` 包装，不减 border-width。

### 3. 子窗口/弹窗处理的全面重写

这是改动量最大的部分。原版对子窗口的处理很简单：有 parent 就浮动，tag 跟随 parent。但实际使用中会遇到很多边界情况：

**问题一：XWayland 的 client leader**

XWayland 应用的 `WM_TRANSIENT_FOR` 有时指向 client leader 而不是真正的父窗口，导致 `window :parent` 为 nil。我加了 `parent-event-received` 标志来区分「没收到 parent 事件」和「收到了但 parent 为 nil」，后者按子窗口处理。

**问题二：弹窗定位**

原版把所有子窗口定位到 output 中心。我改成了：
- 有真实 parent → 居中在 parent 内容区
- 有 client leader（parent 为 nil）→ 通过 `find-logical-parent` 按 app-id 匹配逻辑父窗口，居中在它上面
- 都没有 → 居中在 output

**问题三：焦点抢夺**

弹窗不应该抢走焦点（除非当前没有焦点窗口）。加了 `managed-as-child` 检查。

**问题四：焦点回退**

子窗口关闭后焦点应该回到父窗口。加了 `focus-return-to` 机制，同时处理了父子同时关闭的边界情况（比如微信登录窗口闪退）。

**问题五：半固定尺寸窗口**

有些应用（如 Moments）宽度固定但高度可变，不适合平铺。加了 `window/is-partially-fixed` 检测，自动设为浮动。

### 4. 焦点系统重构

原版的焦点逻辑全部写在 `seat/manage` 里，一个大函数。我拆分成了独立的子函数：

- `seat/resolve-focus-target` — 纯逻辑判断，返回目标窗口
- `seat/apply-focus` — 执行焦点切换
- `seat/clear-focus` — 清除焦点
- `seat/cleanup-stale-refs` — 清理已关闭窗口的引用
- `seat/resolve-focus` — 综合判断（focus-return-to > 新窗口 > 交互）
- `seat/execute-action` — 执行键绑定动作

还修复了原版的 `non-exclusive` 层交互问题：点击 waybar 后焦点会卡在 layer surface 上，需要重新 assert 焦点到当前窗口。

### 5. 大量新增 action

原版提供的 action 比较基础。我新增了：

| Action | 功能 |
|--------|------|
| `action/swap` | 交换聚焦窗口和相邻窗口的位置 |
| `action/send-to-output` | 把窗口发送到另一个显示器 |
| `action/focus-output` | 增强版，支持 next/prev 双向 |
| `action/sticky` | 窗口置顶（所有 tag 可见） |
| `action/retile` | 重置所有用户浮动窗口回平铺 |
| `action/set-main-ratio` | 动态调整 master 区域比例 |
| `action/set-nmaster` | 动态调整 master 窗口数量 |
| `action/float-move` | 键盘移动浮动窗口（带边界碰撞） |
| `action/float-resize` | 键盘缩放浮动窗口（居中、尊重 hints） |
| `action/float-snap` | 浮动窗口吸附到屏幕边缘 |
| `action/set-layout` | 切换布局类型 |
| `action/set-tile-location` | 设置 master-stack 方向 |
| `action/switch-to-previous-layout` | 切回上一个布局 |

### 6. 容错与稳定性

原版没有错误保护。如果某个窗口的事件处理抛异常（常见于 XWayland 应用半初始化状态），整个 manage 循环就死了，所有键绑定都不响应。

我用 `protect` 包裹了 `wm/manage` 和 `wm/render` 的核心逻辑，同时确保 `manage-finish` / `render-finish` 始终被调用。单个窗口出错不会影响整个 WM。

## 改动对照表

| 模块 | 原版 | 我的版本 |
|------|------|---------|
| 布局 | 1 种（左 master） | 3 种 + 4 方向 + 按 output 切换 |
| 尺寸约束 | 无 | clamp-to-hints + clip-box |
| 子窗口识别 | parent 事件 | parent + client leader + app-id 匹配 |
| 子窗口定位 | output 中心 | 智能居中（parent > logical parent > output） |
| 焦点回退 | 无 | focus-return-to 机制 |
| 焦点结构 | 单函数 | 6 个独立子函数 |
| Sticky 窗口 | 无 | 支持 |
| 浮动操作 | 仅鼠标 | 鼠标 + 键盘移动/缩放/吸附 |
| 窗口交换 | 无 | swap next/prev |
| 跨显示器 | 单向 focus-output | 双向 focus/send-to-output |
| nmaster | 固定 1 | 可配置，运行时动态调整 |
| 容错 | 无 | protect 包裹核心循环 |
| 背景管理 | single-pixel-buffer | 移除（交给 swaybg 等外部工具） |
| 代码量 | ~650 行 | ~1100 行 |

## 感想

改 Rijan 的过程让我对 Wayland 窗口管理的本质有了更深的理解。

Isaac 的原版代码有一种「刚好够用」的美感——每一行都有存在的理由，没有一行是多余的。这种克制在开源世界里很少见。他没有试图做一个功能齐全的 WM，而是做了一个正确的、可扩展的骨架。

我的改动某种程度上「弄脏」了这种简洁，但这些都是日用中真实遇到的问题。微信的弹窗行为、XWayland 的 client leader、半固定尺寸窗口——这些边界情况不会出现在设计文档里，只有真正用了才会撞上。

River + Rijan 的架构证明了一件事：Wayland 的窗口管理可以是完全可编程的，而且不需要用 C 或 Zig。用一个脚本语言就能写出一个功能完整的 WM，这在 X11 时代是不可想象的。

感谢 Isaac Freund 创造了 River 和 Rijan。站在巨人的肩膀上，折腾才有意义。

---

## 链接

- [River compositor](https://codeberg.org/river/river)
- [Rijan 原版](https://codeberg.org/ifreund/rijan)
- [我的改动版](https://github.com/1ces0ul/config-river.0.4.0-dev)
- [Janet 语言](https://janet-lang.org/)
