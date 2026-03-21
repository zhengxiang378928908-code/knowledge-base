# Warp 终端教程与常用指令大全

## Warp 简介

Warp 是一款基于 Rust 构建的现代化终端工具，专为开发者设计。提供 AI 辅助、Block 编辑、智能补全、Workflows 等功能，大幅提升命令行操作效率。

---

## 核心功能

### 1. Block 编辑模式

每条命令及输出为独立 Block，可单独复制、分享。

- 用 `Cmd + C` 复制选中 Block 的输出
- 用 `Cmd + Shift + C` 复制整个 Block（含命令和输出）
- 点击 Block 右上角分享按钮生成共享链接

### 2. Warp AI（内置 AI 助手）

- 按 `Ctrl + Space` 打开 AI 面板
- 输入自然语言描述需求，自动生成对应命令
- 支持 Explain 模式：选中命令后让 AI 解释含义
- 支持上下文感知，可基于当前终端输出对话

### 3. Workflows（工作流）

- 按 `Ctrl + Shift + R` 打开 Workflows 面板
- 搜索、收藏和运行常见命令模板
- 支持自定义工作流并在团队间共享

### 4. 智能补全

- 输入时自动弹出子命令、参数和文件路径补全
- 按 `Tab` 接受补全建议

---

## 常用快捷键

| 快捷键 | 功能 |
|--------|------|
| `Cmd + D` | 水平分屏 |
| `Cmd + Shift + D` | 垂直分屏 |
| `Cmd + T` | 新建标签页 |
| `Cmd + W` | 关闭当前面板 |
| `Cmd + P` | 命令面板（Command Palette） |
| `Ctrl + Space` | 打开 Warp AI |
| `Ctrl + Shift + R` | 打开 Workflows |
| `Ctrl + R` | 搜索历史命令 |
| `Cmd + Shift + C` | 复制当前 Block |
| `Cmd + K` | 清屏 |
| `Cmd + ,` | 打开设置 |
| `Option + Up/Down` | 在 Block 之间导航 |

---

## 输入编辑技巧

| 操作 | 快捷键 |
|------|--------|
| 多光标编辑 | `Option + Click` |
| 选择整行 | `Cmd + L` |
| 多行输入 | `Shift + Enter` |
| 撤销 / 重做 | `Cmd + Z` / `Cmd + Shift + Z` |

---

## Warp Drive（团队协作）

- **共享 Workflows**：团队共享命令模板
- **共享 Notebooks**：将终端输出保存为可协作文档
- **共享环境变量**：在团队间同步 .env 配置
- 使用 `Cmd + P` 搜索 "Warp Drive" 进入

---

## 自定义配置

### 主题

- 设置 > Appearance > 选择内置主题
- 自定义主题位于 `~/.warp/themes/`（YAML 格式）

### 自定义 Workflow

在 `~/.warp/workflows/` 下创建 YAML 文件：

```yaml
name: Deploy to Production
command: git pull origin main && npm run build && npm run deploy
tags:
  - deploy
description: Pull latest code, build, and deploy
arguments:
  - name: branch
    description: Branch to deploy
    default_value: main
```

---

## 常用终端命令速查

### 文件与目录

- `ls -la` 列出文件详情含隐藏文件
- `cd ~/projects` 切换目录
- `mkdir -p src/components` 递归创建目录
- `cp -r src/ backup/` 复制目录
- `mv old.txt new.txt` 移动/重命名
- `rm -rf node_modules/` 删除文件或目录
- `find . -name "*.ts"` 查找文件
- `tree -L 2` 树状显示目录

### Git 常用操作

- `git status` 查看工作区状态
- `git add .` 暂存所有更改
- `git commit` 提交更改
- `git pull origin main` 拉取远程更新
- `git push` 推送到远程
- `git branch -a` 查看所有分支
- `git checkout -b feature` 创建并切换分支
- `git log --oneline` 查看提交历史
- `git stash` 暂存工作区更改

### 进程与系统

- `ps aux` 查看所有进程
- `top` / `htop` 系统资源监控
- `lsof -i :3000` 查看端口占用
- `df -h` 查看磁盘空间
- `du -sh *` 查看目录大小

---

## 实用技巧

### 管道与重定向

- `cmd1 | cmd2` 将 cmd1 输出传给 cmd2
- `cmd > file` 输出写入文件（覆盖）
- `cmd >> file` 输出追加到文件
- `cmd1 && cmd2` cmd1 成功后执行 cmd2
- `cmd1 || cmd2` cmd1 失败后执行 cmd2

### Warp 特色技巧

1. **右键菜单**：在输出上右键可快速复制、搜索或用 AI 解释
2. **拖拽文件**：直接将文件拖入输入框自动填入路径
3. **书签命令**：hover 命令后点击书签图标可收藏
4. **Notebook 模式**：`Cmd + P` 搜索 Open Notebook 进入富文本模式
5. **SSH 集成**：连接远程服务器后 AI 和补全功能依然可用

> 💡 **提示**：Warp 持续更新中，更多功能可访问官方文档 [docs.warp.dev](http://docs.warp.dev) 获取最新信息。
