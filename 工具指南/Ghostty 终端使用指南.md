# Ghostty 终端使用指南

一份简洁实用的 Ghostty 终端模拟器使用手册，涵盖安装、快捷键、配置和日常技巧。

---

## 什么是 Ghostty

Ghostty 是一款用 Zig 编写的高性能 GPU 加速终端模拟器，注重速度、功能和原生体验。macOS 上提供 `.app`，Linux 支持多种包管理器
---

## 安装

### macOS

从官网下载 `.dmg` 或使用 Homebrew：

```bash
brew install --cask ghostty
```

### Linux

参考官方文档选择对应发行版的安装方式（apt / pacman / nix 等）。

---

## 核心快捷键

| 操作                 | 快捷键                   | 说明                       |
| ------------------ | --------------------- | ------------------------ |
| **新窗口**            | `Cmd + N`             | 打开一个全新的 Ghostty 窗口       |
| **新标签页**           | `Cmd + T`             | 在当前窗口内新建标签页              |
| **水平分屏**           | `Cmd + D`             | 左右分割当前面板                 |
| **垂直分屏**           | `Cmd + Shift + D`     | 上下分割当前面板                 |
| **切换分屏**           | `Cmd + [` / `Cmd + ]` | 在分屏面板间切换焦点               |
| **关闭面板/标签**        | `Cmd + W`             | 关闭当前分屏面板或标签页             |
| **切换标签页**          | `Cmd + 1-9`           | 跳转到第 N 个标签页              |
| **放大字体**           | `Cmd + =`             | 增大字体大小                   |
| **缩小字体**           | `Cmd + -`             | 减小字体大小                   |
| **重置字体**           | `Cmd + 0`             | 恢复默认字体大小                 |
| **全屏**             | `Cmd + Enter`         | 切换全屏模式                   |
| **搜索**             | `Cmd + F`             | 在终端输出中搜索文本               |
| **Quick Terminal** | `Global hotkey`       | 可配置的全局快速唤出终端             |
| **到行首**            | `Ctrl + A`            | 在当前命令行中把光标移动到行首          |
| **到行尾**            | `Ctrl + E`            | 在当前命令行中把光标移动到行尾          |
| **删除前面内容**         | `Ctrl + U`            | 删除光标前的全部输入内容             |
| **删除后面内容**         | `Ctrl + K`            | 删除光标后的全部输入内容             |
| **删除前一个单词**        | `Ctrl + W`            | 删除光标前的一个单词，适合快速改命令参数     |
| **粘贴刚删除内容**        | `Ctrl + Y`            | 粘贴最近一次通过 shell 快捷键删除的内容  |
| **清屏**             | `Ctrl + L`            | 清空当前终端显示内容，等价于执行 `clear` |
| **搜索历史命令**         | `Ctrl + R`            | 反向搜索历史命令，适合快速复用旧命令       |

---

## 配置文件

配置文件路径：`~/.config/ghostty/config`

如果文件不存在则手动创建：

```bash
mkdir -p ~/.config/ghostty && touch ~/.config/ghostty/config
```

配置使用 `key = value` 格式，每行一个设置。

---

## 常用配置项

### 字体

```
font-family = "JetBrains Mono"
font-size = 14
font-thicken = true
```

### 主题 / 配色

Ghostty 内置多种主题，可直接使用名称：

```
theme = GruvboxDark
```

查看所有可用主题：

```bash
ghostty +list-themes
```

也可以自定义颜色：

```
background = 06132b
foreground = d9e2ff
cursor-color = ffb786
```

### 窗口

```
window-padding-x = 10
window-padding-y = 10
window-decoration = true
macos-titlebar-style = hidden
```

### 光标

```
cursor-style = block
cursor-style-blink = false
```

### 透明度与模糊

```
background-opacity = 0.9
background-blur = 20
```

### Quick Terminal（全局唤出）

```
keybind = global:cmd+grave_accent=toggle_quick_terminal
quick-terminal-position = top
quick-terminal-screen = main
quick-terminal-animation-duration = 0.15
```

### Shell 集成

```
shell-integration = zsh
shell-integration-features = cursor,sudo,title
```

---

## 实用技巧

> 💡 **快速重载配置**：修改配置文件后，Ghostty 会自动检测并热重载，无需重启。

> 🔍 **查看当前配置**：在终端中运行 `ghostty +show-config` 可以输出当前生效的完整配置。

> 📋 **查看所有可用配置项**：运行 `ghostty +help` 查看完整配置项文档。

> 🎨 **自定义 Shader**：Ghostty 支持自定义 fragment shader，可以实现 CRT 效果等视觉效果。配置项为 `custom-shader`。

---

## 与其他终端对比

| 特性 | Ghostty | iTerm2 | Alacritty | Warp |
|------|---------|--------|-----------|------|
| GPU 加速 | ✅ | 部分 | ✅ | ✅ |
| 原生 UI | ✅ | ✅ | ❌ | ✅ |
| 分屏 | ✅ | ✅ | ❌ | ✅ |
| 配置热重载 | ✅ | ✅ | ✅ | ✅ |
| 开源 | ✅ | ✅ | ✅ | ❌ |
| 跨平台 | macOS + Linux | macOS only | 全平台 | macOS + Linux |

---

## 常见问题

**配置文件修改后没有生效？**
确认文件路径为 `~/.config/ghostty/config`（注意没有文件扩展名）。检查语法是否正确，每行一个 `key = value`。

**字体显示异常或缺字？**
确保系统已安装对应字体。运行 `fc-list | grep "字体名"` 确认。Ghostty 要求字体名精确匹配。

**如何恢复默认配置？**
删除或重命名配置文件即可：`mv ~/.config/ghostty/config ~/.config/ghostty/config.bak`

**Quick Terminal 没有反应？**
需要在配置中设置全局快捷键（`keybind = global:...`），并确保 Ghostty 在后台运行且获得了辅助功能权限（系统设置 → 隐私与安全 → 辅助功能）。
