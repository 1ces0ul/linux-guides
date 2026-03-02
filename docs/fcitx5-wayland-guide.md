# Fcitx5 在 Wayland 下的配置指南

> 适用于 wlroots 系合成器（River、Sway 等），其他合成器可参考。

## 目录

- [原理概述](#原理概述)
- [协议总览](#协议总览)
- [各桌面环境支持情况](#各桌面环境支持情况)
- [环境变量配置](#环境变量配置)
- [环境变量放在哪里](#环境变量放在哪里)
- [GTK 单独配置](#gtk-单独配置)
- [用 systemd 管理 Fcitx5](#用-systemd-管理-fcitx5)
- [Chromium / Electron 应用](#chromium--electron-应用)
- [已知问题](#已知问题)
- [常见问题](#常见问题)

---

## 原理概述

X11 下输入法有统一的 XIM 框架，所有应用走同一条路。Wayland 没有这种统一框架，输入法需要通过多种协议跟合成器通信，不同桌面环境、不同工具包支持的协议不同。

Fcitx5 在 Wayland 下有两种工作方式：

1. **text-input 协议路径**：应用 → 合成器 → Fcitx5（通过合成器中转）
2. **im-module 路径**：应用 → Fcitx5（直接通信，绕过合成器）

两种各有优劣，需要根据实际情况选择。

## 协议总览

### 应用 → 合成器（text-input 协议）

| 协议 | 支持的合成器 | 说明 |
|------|-------------|------|
| text-input-v1 | Weston, KWin 5.27+ | 最早的版本，Chromium 用的 |
| text-input-v2 | KWin 专属 | Qt 私有，只有 KDE 支持 |
| text-input-v3 | GNOME, Sway, River, 大多数现代合成器 | 当前主流标准 |

### 合成器 → 输入法（input-method 协议）

| 协议 | 支持的合成器 | 说明 |
|------|-------------|------|
| input-method-v1 | Weston | 老版本 |
| input-method-v2 | KWin, Sway, River | 当前主流，wlroots 系都用这个 |

### im-module（直接通信）

Fcitx5 提供 GTK 和 Qt 的 im-module 插件，直接嵌入应用进程内部与 Fcitx5 通信，不经过合成器。

## 各桌面环境支持情况

### KDE Plasma（最完善）

- 支持 text-input-v1/v2/v3 + input-method-v1/v2
- 不需要设 GTK_IM_MODULE 和 QT_IM_MODULE
- 在「系统设置 → 虚拟键盘」中选择 Fcitx 5 启动

### GNOME

- 支持 text-input-v3
- 通过 ibus D-Bus 协议对接输入法，Fcitx5 启动时自动替代 ibus
- 需要安装 [Kimpanel 扩展](https://extensions.gnome.org/extension/261/kimpanel/) 才能正确显示候选窗口
- Qt 应用需要设 `QT_IM_MODULE=fcitx`

### River / Sway（wlroots 系）

- 支持 text-input-v3 + input-method-v2
- input-method-v2 实现不完整，候选窗口定位可能有问题
- Qt 应用必须设 `QT_IM_MODULE=fcitx`（因为不支持 Qt 私有的 text-input-v2）
- River 支持 `wlr-foreign-toplevel-management`，Fcitx5 可识别当前窗口，实现按窗口记忆输入法状态

### Weston

- 只支持 text-input-v1 + input-method-v1
- 必须全部走 im-module：`GTK_IM_MODULE=fcitx`、`QT_IM_MODULE=fcitx`

## 环境变量配置

### wlroots 系（River / Sway）推荐配置

```ini
XMODIFIERS=@im=fcitx
QT_IM_MODULE=fcitx
QT_IM_MODULES=wayland;fcitx;ibus
# 不要设 GTK_IM_MODULE！让 GTK3/4 自动走 text-input-v3
```

### 各变量的作用

| 变量 | 作用 | 谁读 |
|------|------|------|
| `XMODIFIERS` | XWayland 下的 X11 应用用 | 所有 X11 应用 |
| `QT_IM_MODULE` | 指定 Qt 输入法模块 | Qt 5 和 Qt 6.6 以下 |
| `QT_IM_MODULES` | 指定 Qt 输入法 fallback 链 | Qt 6.7+，按顺序尝试 |
| `GTK_IM_MODULE` | 指定 GTK 输入法模块 | GTK 2/3/4 |

### 为什么不设 GTK_IM_MODULE

- 不设时，GTK3/4 的 Wayland 原生应用自动走 text-input-v3 协议
- 设成 `fcitx` 也能用，但 Wayland 原生应用会走 im-module 而非原生协议
- 设成 `wayland` 也不好，因为 GTK2 也会读到这个变量，导致 GTK2 应用崩

### QT_IM_MODULE 和 QT_IM_MODULES 的关系

```ini
QT_IM_MODULES=wayland;fcitx;ibus   # Qt 6.7+ 读这个，按顺序 fallback
QT_IM_MODULE=fcitx                   # Qt 5 / Qt 6.6 以下读这个
```

两个都设，新旧版本 Qt 通吃。`QT_IM_MODULES` 是 Qt 6.7 新增的变量，老版本不认。

### 为什么 Qt 不能像 GTK 一样不设变量

Qt 默认的 Wayland 输入法模块用的是 text-input-v2（KDE 私有协议），只有 KWin 支持。在非 KDE 环境下不设 `QT_IM_MODULE`，Qt 应用的输入法就没法用。

## 环境变量放在哪里

### 方式一：~/.config/environment.d/（推荐，systemd 环境）

```bash
mkdir -p ~/.config/environment.d

cat > ~/.config/environment.d/fcitx.conf << 'EOF'
XMODIFIERS=@im=fcitx
QT_IM_MODULE=fcitx
QT_IM_MODULES=wayland;fcitx;ibus
EOF
```

- 用户级别，不影响其他用户
- systemd 原生支持，session 启动时自动加载
- **注意：不要加引号**，systemd 会把引号当成值的一部分

### 方式二：~/.profile（从 tty 手动启动合成器时）

```bash
export XMODIFIERS=@im=fcitx
export QT_IM_MODULE=fcitx
export QT_IM_MODULES="wayland;fcitx;ibus"
```

- shell 脚本，分号需要加引号（`;` 是 shell 命令分隔符）
- 只在 login shell 启动时读取

### 方式三：/etc/environment（不推荐）

```ini
XMODIFIERS=@im=fcitx
QT_IM_MODULE=fcitx
QT_IM_MODULES="wayland;fcitx;ibus"
```

- 全局生效，影响所有用户
- 语法简单，不支持条件判断
- 引号加不加都行（PAM 解析器会自动去掉外层引号），建议加

### 引号规则总结

| 文件 | 分号要加引号吗 | 原因 |
|------|--------------|------|
| environment.d/*.conf | ❌ 不加 | systemd 解析，加了引号会变成值的一部分 |
| ~/.profile | ✅ 必须加 | shell 脚本，`;` 是命令分隔符 |
| /etc/environment | ➕ 建议加 | PAM 解析器会去掉引号，加上更安全 |

## GTK 单独配置

不设 `GTK_IM_MODULE` 全局变量的情况下，通过配置文件让 XWayland 下的 GTK 应用使用 fcitx：

```bash
# GTK2: ~/.gtkrc-2.0
gtk-im-module="fcitx"

# GTK3: ~/.config/gtk-3.0/settings.ini
[Settings]
gtk-im-module=fcitx

# GTK4: ~/.config/gtk-4.0/settings.ini
[Settings]
gtk-im-module=fcitx
```

如果使用 GNOME，还需要：

```bash
gsettings set org.gnome.settings-daemon.plugins.xsettings overrides "{'Gtk/IMModule':<'fcitx'>}"
```

## 启动 Fcitx5

### 推荐方式：在合成器 init 中直接启动

Fcitx5 本身就是 daemon（`-d` 参数 daemonize），它自己管理生命周期、处理信号、重连 wayland socket。不需要额外的进程管理器。

在合成器的 init 脚本中（如 River 的 init 或 Rijan 配置文件）：

```bash
# 先同步环境变量给 D-Bus 和 systemd
dbus-update-activation-environment --systemd WAYLAND_DISPLAY XDG_CURRENT_DESKTOP DISPLAY

# 启动 fcitx5（pgrep 防止重复启动，比如重载配置时）
pgrep -x fcitx5 > /dev/null || fcitx5 -d &
```

这是最简洁、最可靠的方式。当合成器退出时 wayland socket 断开，fcitx5 自然退出，不需要手动清理。

### 为什么不推荐 systemd 管理 Fcitx5

用 systemd user service 管理 fcitx5 看起来"正规"，但在 wlroots 系合成器（River、Sway 等）下会引入不必要的复杂度：

1. **需要手动激活 graphical-session.target** — KDE/GNOME 的 session manager 会自动做，但 River/Sway 不会。你必须在 init 里加 `systemctl --user start graphical-session.target`，否则 service 不会被拉起。

2. **需要 import-environment** — systemd user manager 看不到 `WAYLAND_DISPLAY`，必须手动导入，否则 fcitx5 找不到 wayland socket。

3. **三个活动部件**（service 文件 + import-environment + graphical-session.target），任何一个配错输入法就不起来，且排查困难。

相比之下，直接 `fcitx5 -d &` 一行搞定，没有额外依赖。

### 注意事项：XWayland 应用启动时序

如果你在 init 中同时 autostart XWayland 应用（如微信），应用可能在键盘布局和输入法完全就绪前启动，导致使用默认布局。解决方法：

- 给 autostart 的 XWayland 应用加延迟：`sleep 2 && wechat &`
- 或者等合成器初始化完成通知后再手动打开

## Chromium / Electron 应用

### 以 XWayland 模式运行（简单）

默认情况下 Chromium/Electron 可能走 XWayland，此时跟 X11 一样，确保 `GTK_IM_MODULE=fcitx` 即可。

### 以 Wayland 原生模式运行

```bash
# text-input-v1（推荐 wlroots 系合成器）
chromium --enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime

# text-input-v3
chromium --enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime --wayland-text-input-version=3

# Electron 应用（如 VS Code）
code --enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime
```

### 判断应用是否跑在 Wayland 原生模式

用 `xeyes` 测试：鼠标移到应用窗口上，如果眼睛跟着动就是 X11 窗口，不动就是 Wayland 原生。

## 已知问题

### 候选窗口定位/闪烁

Wayland 没有全局坐标系统，候选窗口可能位置不准或闪烁。wlroots 系的 input-method-v2 实现不完整，这个问题尤为明显。

### 输入法状态全局共享

使用 input-method-v2 协议时，Fcitx5 只能看到一个全局输入上下文。但如果合成器支持 `wlr-foreign-toplevel-management`（River/Sway 支持），Fcitx5 可以识别当前窗口，实现按窗口独立记忆输入法状态。

### XKB 键盘布局

Fcitx5 托管的键盘布局目前只在 KDE 和 GNOME 下正常工作。其他合成器需要手动确保合成器配置的布局与 Fcitx5 输入法组的布局一致。

## 常见问题

### Q: QtWebEngine 应用（qutebrowser 等）候选窗口不更新怎么办？

A: 这是 Fcitx5 的 Qt im-module 跟 QtWebEngine（Chromium 内核）对接的 bug。解决方案：

- 全局设 `QT_IM_MODULES=wayland;fcitx;ibus`（Qt 6.7+）
- 或单独给该应用设 `QT_IM_MODULE=wayland`

### Q: 设了 QT_IM_MODULE=wayland 后 XWayland 的 Qt 应用没输入法了？

A: 因为 XWayland 应用没有 Wayland 连接，找不到 wayland 输入法模块。用 `QT_IM_MODULES=wayland;fcitx;ibus` 设置 fallback 链，或者给 XWayland 应用单独设 `QT_IM_MODULE=fcitx`。

### Q: 我需要同时设 QT_IM_MODULE 和 QT_IM_MODULES 吗？

A: 是的。`QT_IM_MODULES`（带 s）是 Qt 6.7+ 才认的新变量，老版本 Qt 只读 `QT_IM_MODULE`。两个都设，新旧通吃。

---

## 参考

- [Fcitx5 官方 Wayland 文档](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland)
- [ArchWiki - Fcitx5](https://wiki.archlinux.org/title/Fcitx5)
- [Wayland text-input-v3 协议](https://wayland.app/protocols/text-input-unstable-v3)
- [Wayland input-method-v2 协议](https://wayland.app/protocols/input-method-unstable-v2)
