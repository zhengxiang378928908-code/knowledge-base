# OpenClaw 安装与配置完整指南

## 1. OpenClaw 简介

OpenClaw 是一个本地运行的 AI Agent 框架，可以连接：

- GPT
- Claude
- MiniMax
- DeepSeek
- Telegram / Discord / Slack

它通过 **Gateway 服务**统一管理模型和工具。

系统结构：

```
Browser UI → Control UI → Gateway (WebSocket) → Agents / Models / Skills
```

---

## 2. 系统要求

| 项目 | 最低要求 |
|------|----------|
| Node.js | >= 22 |
| npm | >= 10 |
| RAM | >= 4GB |

---

## 3. 安装 OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装完成后检查版本：

```bash
openclaw --version
```

---

## 4. 初始化配置

```bash
openclaw configure
# 或
openclaw onboard
```

向导会让你选择 AI provider、输入 API key、选择模型、设置 gateway 和 skills。

---

## 5. 配置目录

默认：`~/.openclaw`

```
.openclaw
 ├ openclaw.json
 ├ workspace
 ├ sessions
 ├ logs
 └ plugins
```

---

## 6. Gateway 服务

核心组件，默认端口：`127.0.0.1:18789`

```bash
openclaw gateway
```

浏览器打开：`http://127.0.0.1:18789`

---

## 7. 获取 Gateway Token

```bash
openclaw config get gateway.auth.token
```

重新生成：

```bash
openclaw doctor --generate-gateway-token
```

---

## 8. 配置示例

`~/.openclaw/openclaw.json`：

```json
{
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "YOUR_TOKEN"
    }
  }
}
```

---

## 9. 验证安装

```bash
openclaw status
openclaw gateway status
openclaw doctor
```

---

## 10. 常见问题

**Gateway unreachable**
→ `openclaw gateway restart`，检查 `lsof -i :18789`

**unauthorized / token 错误**
→ `openclaw config get gateway.auth.token`，重新填入

**openclaw: command not found**
→ `export PATH="$HOME/.openclaw/bin:$PATH"` 加入 `~/.zshrc`

**端口 18789 被占用**
→ `lsof -i :18789` 找到 PID → `kill -9 <PID>`

---

## 11. 安全建议

不要将 gateway 暴露到公网，始终绑定 `127.0.0.1` 并启用 token 认证。

---

## 12. 推荐模型配置

| 角色 | 模型 |
|------|------|
| Primary | MiniMax-M2.5 |
| Fallback 1 | DeepSeek |
| Fallback 2 | Claude |
| Fallback 3 | GPT |

> 文档版本：2026 | 系统：Windows / macOS / Linux
