# MemPalace WSL2 实际安装状态 & 踩坑记录

> 记录日期：2026-05-13
> 参考：[windows-mempalace-global-install-runbook.md](./windows-mempalace-global-install-runbook.md)

---

## 当前机器状态（已验证可用）

### Binary 位置

| 平台 | 路径 |
|------|------|
| Windows | `C:\Users\DEVTrump\.local\bin\mempalace-mcp.exe` |
| WSL2 Linux | `/home/devtrump/.local/bin/mempalace-mcp` |
| WSL2 Linux | `/home/devtrump/.local/bin/mempalace` |

### Palace 路径

所有工具共用同一个 palace：

```
C:\Users\DEVTrump\.mempalace\palace   （Windows 路径）
/mnt/c/Users/DEVTrump/.mempalace/palace  （WSL2 访问路径）
```

### Mempalace config.json

两个用户都要有，否则 hook 会存到错误位置：

```
/home/devtrump/.mempalace/config.json  ← devtrump 用户
/root/.mempalace/config.json           ← root 用户（Claude Code 以 root 跑）
```

两个文件内容相同：

```json
{
  "palace_path": "/mnt/c/Users/DEVTrump/.mempalace/palace",
  "collection_name": "mempalace_drawers",
  "entity_languages": ["en", "zh"]
}
```

---

## 各工具 MCP 配置

### Claude Code（VS Code 扩展，WSL2 root 用户）

文件：`/root/.claude/claude.json`

```json
{
  "mcpServers": {
    "mempalace": {
      "command": "/home/devtrump/.local/bin/mempalace-mcp"
    }
  }
}
```

**注意：** Claude Code 在 WSL2 里以 root 运行，必须改 `/root/.claude/claude.json`，改 `/home/devtrump/.claude/claude.json` 没用。

### Codex

文件：`C:\Users\DEVTrump\.codex\config.toml`

```toml
[mcp_servers.mempalace]
command = "C:/Users/DEVTrump/.local/bin/mempalace-mcp.exe"
args = ["--palace", "C:/Users/DEVTrump/.mempalace/palace"]
```

### VS Code Copilot（GitHub Copilot MCP）

文件：`C:\Users\DEVTrump\AppData\Roaming\Code\User\mcp.json`

```json
{
  "servers": {
    "mempalace": {
      "command": "C:\\Users\\DEVTrump\\.local\\bin\\mempalace-mcp.exe",
      "args": ["--palace", "C:\\Users\\DEVTrump\\.mempalace\\palace"]
    }
  }
}
```

### Claude 桌面 App（CHAT 标签页的 local agent mode）

文件：`C:\Users\DEVTrump\AppData\Local\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "mempalace": {
      "command": "C:\\Users\\DEVTrump\\.local\\bin\\mempalace-mcp.exe",
      "args": ["--palace", "C:\\Users\\DEVTrump\\.mempalace\\palace"]
    }
  }
}
```

---

## 自动存档钩子（Stop / PreCompact）

文件：`/root/.claude/settings.json`，在 `hooks` 字段里：

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{}' | /home/devtrump/.local/bin/mempalace hook run --hook stop --harness claude-code"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{}' | /home/devtrump/.local/bin/mempalace hook run --hook precompact --harness claude-code"
          }
        ]
      }
    ]
  }
}
```

Stop 钩子在每轮 AI 回答结束后触发，自动把内容存进 palace。

---

## 手动 mine 历史会话

```bash
# mine WSL2 Claude Code 会话（root 用户）
/home/devtrump/.local/bin/mempalace mine /root/.claude/sessions/ \
  --mode convos --wing claude-code-wsl --agent claude --limit 0

# mine Windows Claude Code 会话
mempalace mine %USERPROFILE%\.claude\projects --mode convos --wing claude-code --agent claude --limit 0

# mine Codex 会话
mempalace mine %USERPROFILE%\.codex\sessions --mode convos --wing codex --agent codex --limit 0
```

---

## 关键踩坑

1. **MCP 面板不显示本地 MCP** — Claude Code 的 MCP servers UI 面板只显示 claude.ai 账号绑定的云端 MCP，本地 MCP 不会出现在面板里，但实际已加载。用 `claude mcp list` 验证。

2. **root vs devtrump 用户** — Claude Code VS Code 扩展在 WSL2 里以 root 运行，配置必须在 `/root/.claude/`，不是 `/home/devtrump/.claude/`。

3. **mempalace config.json 必须两个用户都有** — 否则 root 用户的 hook 会找不到正确的 palace 路径。

4. **钩子不会自动注册** — 插件 Connected 不等于钩子生效。必须手动在 `settings.json` 里配 hooks 字段。

5. **模型不点名不一定会自查 mempalace** — 别指望模型自动识别，重要规则写进 CLAUDE.md 更可靠。
