# Windows MemPalace 全局安装与验收 Runbook

目标：在 Windows 笔记本上把 MemPalace 装成本机所有 CLI agent 的长期记忆底座。你交付前必须自己完成安装、修坑、导入、验证；用户只负责直接用。

覆盖入口：

- Codex
- Claude Code
- OpenClaw
- OpenCode
- Hermes agent
- 其它支持 stdio MCP 的本机 agent

核心原则：

- 全机共用一个 palace：`%USERPROFILE%\.mempalace\palace`
- 优先 stdio MCP：本机进程直连，最稳，不依赖公网、OAuth、反代、ChatGPT connector UI
- 只用官方源：GitHub / PyPI / `mempalaceofficial.com`
- 安装后必须验收功能完备可用，不要只写配置就结束

## 0. 不要碰的东西

MemPalace README 明确警告：只信这些来源：

- `https://github.com/MemPalace/mempalace`
- PyPI package `mempalace`
- `https://mempalaceofficial.com`

不要安装 `mempalace.tech` 或其它第三方域名的脚本/二进制。

## 1. 基础检查

用 PowerShell 执行：

```powershell
python --version
git --version
uv --version
codex --version
claude --version
openclaw --version
```

如果缺 `uv`：

```powershell
winget install --id astral-sh.uv -e
```

如果缺 Git：

```powershell
winget install --id Git.Git -e
```

安装后重开 PowerShell，再继续。

## 2. 拉源码并可编辑安装

```powershell
mkdir "$env:USERPROFILE\gh-projects" -Force
cd "$env:USERPROFILE\gh-projects"
git clone https://github.com/MemPalace/mempalace.git
cd mempalace
uv tool install --editable .
```

确认命令：

```powershell
where mempalace
where mempalace-mcp
```

记下 `mempalace-mcp.exe` 绝对路径。后面所有 agent MCP 配置都优先用绝对路径，不要依赖 PATH。

## 3. 修 Codex marketplace 兼容坑

检查：

```text
%USERPROFILE%\gh-projects\mempalace\.agents\plugins\marketplace.json
```

如果有：

```json
"authentication": "NONE"
```

改成：

```json
"authentication": "ON_USE"
```

原因：当前 Codex 可能拒绝 `NONE`，错误类似：

```text
unknown variant `NONE`, expected `ON_INSTALL` or `ON_USE`
```

## 4. 初始化和首次索引

```powershell
cd "$env:USERPROFILE\gh-projects\mempalace"
mempalace init "$PWD" --yes --no-llm
mempalace mine "$PWD" --limit 20 --agent installer
```

首次 `mine` 会下载 Chroma 本地 embedding 模型。必须允许网络。下载成功后，后续一般不再需要重复下载。

如果失败并出现 DNS/连接错误，例如：

```text
ConnectError: [Errno 8] nodename nor servname provided
```

处理：放行网络后重跑同一条 `mempalace mine ...`。

## 5. 接 Claude Code

```powershell
claude plugin marketplace add "$env:USERPROFILE\gh-projects\mempalace"
claude plugin install --scope user mempalace@mempalace
claude plugin list
claude mcp list
```

必须看到：

```text
mempalace@mempalace enabled
plugin:mempalace:mempalace ... Connected
```

如果 Claude 插件使用裸 `mempalace-mcp` 但健康检查连不上，把 Claude 生成的 MCP 配置改成 `where mempalace-mcp` 查到的绝对路径。

## 6. 接 Codex

注册 marketplace：

```powershell
codex plugin marketplace add "$env:USERPROFILE\gh-projects\mempalace"
```

编辑：

```text
%USERPROFILE%\.codex\config.toml
```

加入或确认：

```toml
[plugins."mempalace@mempalace"]
enabled = true

[mcp_servers.mempalace]
command = "C:\\Users\\你的用户名\\.local\\bin\\mempalace-mcp.exe"
```

把路径替换成 `where mempalace-mcp` 的真实结果。TOML 里 Windows 反斜杠要双写，或者改用 `/`。

验证：

```powershell
codex mcp list
```

必须看到 `mempalace` enabled。

## 7. 接 OpenClaw

优先用绝对路径：

```powershell
openclaw mcp set mempalace '{"command":"C:\\Users\\你的用户名\\.local\\bin\\mempalace-mcp.exe"}'
openclaw mcp list
```

如果 PowerShell JSON 转义麻烦，直接编辑 OpenClaw 配置，加入：

```json
{
  "mcp": {
    "servers": {
      "mempalace": {
        "command": "C:\\Users\\你的用户名\\.local\\bin\\mempalace-mcp.exe"
      }
    }
  }
}
```

验证 `openclaw mcp list` 必须列出 `mempalace`。

## 8. 接 OpenCode / Hermes / 其它本机 agent

原则：只要支持 MCP stdio，就加同一个 server。

通用配置形态：

```json
{
  "mcpServers": {
    "mempalace": {
      "command": "C:\\Users\\你的用户名\\.local\\bin\\mempalace-mcp.exe"
    }
  }
}
```

必须基于每个 agent 的实际配置文件位置落地。不要只写“建议配置”；要找到配置文件、写入、启动/列表验证。

如果某个 agent 不支持 MCP stdio，记录为“不支持直接接入”，不要硬搞 HTTP MCP 除非用户明确要求。HTTP MCP 会引入端口、反代、鉴权、后台服务和公网/本机访问问题，稳定性不如 stdio。

## 9. 导入历史会话

必须顺序执行，不要并发。MemPalace palace 有写锁，并发会报：

```text
palace ... is held by PID ...
```

存在才导入：

```powershell
mempalace mine "$env:USERPROFILE\.claude\projects" --mode convos --wing claude-code --agent claude --limit 0
mempalace mine "$env:USERPROFILE\.codex\sessions" --mode convos --wing codex --agent codex --limit 0
mempalace mine "$env:USERPROFILE\.codex\archived_sessions" --mode convos --wing codex-archived --agent codex --limit 0
mempalace mine "$env:USERPROFILE\.openclaw\agents\main\sessions" --mode convos --wing openclaw --agent openclaw --limit 0
```

再查 OpenCode / Hermes 的真实会话目录，按同样模式导入：

```powershell
mempalace mine "实际OpenCode会话目录" --mode convos --wing opencode --agent opencode --limit 0
mempalace mine "实际Hermes会话目录" --mode convos --wing hermes --agent hermes --limit 0
```

不要导入整个 home 目录。只导入明确的会话目录和需要记忆的项目目录，避免缓存、隐私文件、二进制垃圾进入 palace。

## 10. Hook 权限和自动保存

Windows 一般没有 Unix executable bit 问题，但仍要验证 hooks 能跑。

Claude/Codex 插件的 hooks 作用：

- `Stop`：会话/回合结束时保存
- `PreCompact`：上下文压缩前保存
- `SessionStart`：新会话启动时加载/初始化状态

这不是“用户不说话几秒就保存”。它由 agent 客户端生命周期触发。

必须验证：

```powershell
'{}' | mempalace hook run --hook session-start --harness codex
'{}' | mempalace hook run --hook stop --harness codex
'{}' | mempalace hook run --hook stop --harness claude-code
```

期望退出码为 0。输出 `{}` 可以接受。

## 11. 功能验收，必须全部跑

状态：

```powershell
mempalace status
```

必须显示 drawers，并能看到导入的 wings，例如 `claude-code`、`codex`、`openclaw`、`opencode`、`hermes`。

健康：

```powershell
mempalace repair-status
```

如果 HNSW metadata 暂未 flushed，状态可能是 `UNKNOWN`，但如果 search 正常，可以接受。若明确报坏且 search 不工作，先跑 repair 或重建索引，不能交付半坏状态。

搜索：

```powershell
mempalace search "选一句确定历史会话里出现过的关键词"
```

必须能找回历史内容。

MCP 直测：

```powershell
python -c "from mempalace.mcp_server import handle_request; import json; r=handle_request({'jsonrpc':'2.0','id':1,'method':'tools/call','params':{'name':'mempalace_status','arguments':{}}}); print(json.loads(r['result']['content'][0]['text'])['total_drawers'])"
```

如果系统 `python` 没有 MemPalace 依赖，用 `where mempalace` 找 uv tool 的 Python，或者以 Claude/Codex/OpenClaw 的 MCP 健康检查为准。但至少一个 MCP 真实调用必须通过。

Agent 侧验证：

```powershell
claude mcp list
codex mcp list
openclaw mcp list
```

OpenCode/Hermes 也要用各自命令确认 MCP server 被加载。不要跳过。

## 12. 验收标准

交付前必须满足：

- `mempalace` 和 `mempalace-mcp` 可执行
- 全局 palace 存在：`%USERPROFILE%\.mempalace\palace`
- 至少一个项目索引成功
- 历史会话目录已顺序导入
- `mempalace status` 有 drawers
- `mempalace search` 能找回历史内容
- Claude Code MCP connected
- Codex MCP enabled
- OpenClaw MCP listed
- OpenCode/Hermes：支持 MCP 的已接入并验证；不支持的明确记录原因
- Hooks CLI 自测退出码 0
- 所有 MCP command 使用绝对路径，避免 PATH 问题

## 13. 最终交付格式

安装完成后，给用户一个简短结果，不要甩待办：

```text
MemPalace 已装好并验收通过。

Palace: %USERPROFILE%\.mempalace\palace
Drawers: <数字>
已接入: Claude Code / Codex / OpenClaw / OpenCode / Hermes
已验证:
- mempalace status
- mempalace search
- Claude MCP health
- Codex MCP list
- OpenClaw MCP list
- OpenCode/Hermes MCP list
- hooks run

遗留限制:
- <只写真实限制，没有就写“无”>
```

## 14. 从 macOS 安装踩过的坑

这些坑已经在 Mac 上踩过，Windows 安装时不要重复：

- Codex marketplace `authentication: NONE` 需要改 `ON_USE`
- 首次 Chroma 模型下载失败通常是网络问题，不是 palace 坏
- `mempalace mine` 不要并发
- MCP 命令要用绝对路径
- Hook 脚本/命令要实际跑一次，不要假设可用
- HNSW quarantine 警告不一定是失败，后续 `status/search` 正常即可
- ChatGPT Web/App 不是本机 stdio MCP 入口，先不要纳入“本机全局 agent”验收
