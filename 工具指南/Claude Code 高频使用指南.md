# Claude Code 高频使用指南

## 一、启动方式

```bash
# 交互式会话（最常用）
claude

# 带初始 prompt 启动
claude "帮我分析这个项目结构"

# 非交互单次查询（适合脚本集成）
claude -p "解释 src/auth.ts 的逻辑"

# 管道输入
cat error.log | claude -p "分析这些报错"
```

## 二、会话管理

```bash
# 继续上次会话
claude -c  # 或 --continue

# 选择历史会话恢复
claude -r  # 或 --resume（进入交互选择）

# 恢复指定 session
claude -r "session-id-abc123"

# Fork 一个新会话（不覆盖原始记录）
claude --resume abc123 --fork-session
```

## 三、键盘快捷键

| 快捷键 | 功能 |
|--------|------|
| `↑ / ↓` | 翻历史命令（当前 session 内） |
| `Ctrl+R` | 反向搜索历史命令 |
| `Shift+Enter` | 多行输入（部分终端需运行 `/terminal-setup` 开启） |
| `Esc` | 停止当前生成 |
| `Tab` | 接受智能提示 |
| `?` | 查看当前环境所有可用快捷键 |

## 四、内置 Slash 命令

```
/help           查看所有可用命令
/clear          清空当前上下文（开新任务必做）
/compact        压缩历史对话，释放 token
/context        查看当前 token/context 使用情况
/config         配置设置项
/init           分析项目，生成 CLAUDE.md
/resume         恢复历史会话（`/continue` 是别名）
/terminal-setup 为当前终端安装快捷键绑定
/desktop        将 session 转移到 Desktop App（查看可视化 diff）
```

## 五、模型选择

```bash
claude --model opus     # 复杂任务
claude --model sonnet   # 日常任务，默认
claude --model haiku    # 快速简单任务，省费用

# 指定完整模型名
claude --model claude-sonnet-4-6
```

## 六、权限控制

```bash
# Plan 模式：只规划不执行
claude --permission-mode plan

# 自动跳过权限确认（仅在沙箱/VM 中用）
claude --dangerously-skip-permissions

# 临时限制可用工具
claude -p --tools "Bash,Edit,Read" "query"

# 禁用所有工具（纯问答）
claude -p --tools "" "query"
```

## 七、CLAUDE.md 项目记忆

```
.claude/CLAUDE.md          ← 项目级（随 Git 共享）
~/.claude/CLAUDE.md        ← 个人全局（所有项目生效）
```

用 `/init` 自动生成初始版本，然后手动完善。

## 八、自定义 Slash 命令 / Skills

```
.claude/commands/review.md          ← 旧写法，仍支持
.claude/skills/review/SKILL.md     ← 新写法，功能更强
```

创建 commit 命令示例：

```markdown
<!-- .claude/commands/commit.md -->
---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
description: 自动生成符合规范的 git commit
---

## 当前变更
- 状态: !`git status`
- Diff: !`git diff HEAD`
- 分支: !`git branch --show-current`

根据以上变更生成一条 Conventional Commits 格式的提交信息并提交。
```

## 九、系统 Prompt 扩展

```bash
# 追加指令（推荐，保留默认能力）
claude --append-system-prompt "所有代码必须用 TypeScript 并加 JSDoc 注释"

# 完全替换 system prompt（谨慎使用）
claude --system-prompt "你是一个只写 Python 的专家"

# 从文件加载
claude -p --system-prompt-file ./prompts/security-review.txt "审查这段代码"
```

## 十、跨环境无缝切换

```
/desktop                  → 将当前 session 转到 Desktop App
claude --remote "任务描述" → 在 claude.ai 新建 Web 会话
claude --teleport         → 把 Web 会话接回本地终端
```

## 十一、实用 CLI 组合技

```bash
# 监控日志，异常时通知
tail -f app.log | claude -p "发现异常立刻告诉我"

# PR diff 安全审查
gh pr diff 42 | claude -p \
  "审查这份 PR diff，输出结构化安全结论" \
  --append-system-prompt "你是安全工程师，重点检查漏洞" \
  --output-format json

# 多轮连续分析
claude -p "审查这个项目的性能问题"
claude -c -p "重点看数据库查询"
claude -c -p "汇总所有问题"

# 限制最大执行轮次
claude -p --max-turns 5 "重构这个模块"
```

---

## 核心三条原则

1. **新任务前 `/clear`** — 保持 context 干净
2. **维护好 `CLAUDE.md`** — 让项目规范持久化
3. **善用 `--permission-mode plan`** — 先看计划再执行，避免误操作

> 文档来源：[Claude Code 官方文档](https://code.claude.com/docs)，整理时间：2026-03
