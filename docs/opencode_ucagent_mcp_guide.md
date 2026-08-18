# UCAgent + 后端 CLI 使用指南

本文记录如何让 UCAgent 作为主控，通过 `--backend` 参数调用后端 CLI
（opencode / Codex / Claude Code）执行各验证阶段任务。后端 CLI 通过 MCP
连回 UCAgent，调用 `RoleInfo`、`CurrentTips`、`Complete`、`Exit` 等工具
推进阶段。

> **架构说明**：UCAgent 是主控（controller），后端 CLI 是执行后端（backend）。
> UCAgent 启动自己的 MCP Server，生成 MCP 配置文件让后端连回，然后每个阶段
> 调用后端 CLI 执行任务。用户不直接操作后端 CLI。

```text
UCAgent workflow (主控)
  → 启动 MCP Server (127.0.0.1:<port>)
  → 生成后端 MCP 配置 (opencode.json / .codex/config.toml / .mcp.json)
  → 每个阶段调用后端 CLI (opencode run / codex exec / claude -p)
  → 后端通过 MCP 回调 UCAgent 工具
  → RoleInfo → CurrentTips → 执行任务 → SetCurrentStageJournal → Complete → Exit
```

## 1. 前置条件

- UCAgent 源码和虚拟环境：`/Users/zzy/Workspace/ucagent/.venv`
- 至少一个后端 CLI 已安装（opencode / codex / claude）
- 工作目录：`/Users/zzy/Workspace/ucagent`

```bash
cd /Users/zzy/Workspace/ucagent
source .venv/bin/activate
ucagent --version
```

## 2. 后端配置

UCAgent 的后端配置在 `~/.ucagent/setting.yaml` 的 `backend` 段。每个后端
定义 CLI 命令模板和 MCP 配置文件生成方式。

### 2.1 可用后端

| 后端 | CLI 命令 | MCP 配置文件 | 配置生成方式 |
|------|---------|------------|------------|
| `codex` | `codex exec` | `.codex/config.toml` | `pre_bash_cmd` 手写 |
| `claude` | `claude -p` | `.mcp.json` | `pre_bash_cmd` 复制+sed |
| `opencode` | `opencode run` | `opencode.json` | `render_files` 模板渲染 |

### 2.2 Codex 后端（已验证）

`~/.ucagent/setting.yaml` 中：

```yaml
backend:
  codex:
    clss: ucagent.abackend.UCAgentCmdLineBackend
    args:
      cli_cmd_new: "codex exec {UC_ENV_CMD_BACKEND_EX_ARGS} --skip-git-repo-check --dangerously-bypass-approvals-and-sandbox < {MSG_FILE}"
      cli_cmd_ctx: "codex exec {UC_ENV_CMD_BACKEND_EX_ARGS} --skip-git-repo-check --dangerously-bypass-approvals-and-sandbox < {MSG_FILE}"
      pre_bash_cmd:
        - "mkdir -p {CWD}/.codex/"
        - "echo \"\n[mcp_servers.ucagent]\nurl = 'http://127.0.0.1:{PORT}/mcp'\ntool_timeout_sec=600000000000\n\" > {CWD}/.codex/config.toml"
```

### 2.3 Claude 后端（已验证）

```yaml
backend:
  claude:
    clss: ucagent.abackend.UCAgentCmdLineBackend
    args:
      cli_cmd_new: "claude {UC_ENV_CMD_BACKEND_EX_ARGS} --dangerously-skip-permissions -p < {MSG_FILE}"
      cli_cmd_ctx: "claude {UC_ENV_CMD_BACKEND_EX_ARGS} --dangerously-skip-permissions -c -p < {MSG_FILE}"
      pre_bash_cmd:
        - "mkdir -p {CWD}/.claude/"
        - "cp ~/.claude/.mcp.json {CWD}/.mcp.json"
        - "sed -i '' 's/5000\\/mcp/{PORT}\\/mcp/' {CWD}/.mcp.json"
```

### 2.4 opencode 后端

UCAgent 默认配置（`ucagent/setting.yaml`）中已内置 opencode 后端模板：

```yaml
backend:
  opencode:
    clss: ucagent.abackend.UCAgentCmdLineBackend
    args:
      cli_cmd_new: "opencode run {UC_ENV_CMD_BACKEND_EX_ARGS} {UC_ENV_CMD_BACKEND_EX_ARGS_N} < {MSG_FILE}"
      cli_cmd_ctx: "opencode run {UC_ENV_CMD_BACKEND_EX_ARGS} {UC_ENV_CMD_BACKEND_EX_ARGS_C} -c < {MSG_FILE}"
      render_files:
        "{ASSETS}/mcp_opencode.json": "{CWD}/opencode.json"
      cfg_bash_enable: false
```

`render_files` 会将 `ucagent/assets/mcp_opencode.json` 模板渲染到工作目录的
`opencode.json`，其中 `{{PORT}}` 替换为实际 MCP 端口。模板内容：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "unitytest": {
      "type": "remote",
      "url": "http://127.0.0.1:{{PORT}}/mcp",
      "enabled": true,
      "timeout": 600000000
    }
  }
}
```

如需使用 opencode 后端，将上述配置添加到 `~/.ucagent/setting.yaml` 的
`backend` 段即可。

## 3. 关键启动参数

| 参数 / 环境变量 | 作用 |
|------|------|
| `--backend <name>` | 指定后端 CLI（codex / claude / opencode） |
| `UCAGENT_CMDLINE_START_MCP=1` | 让 UCAgent 在拉起后端前先启动 MCP Server |
| `--mcp-server-port 5002` | MCP 端口（5000 常被 macOS ControlCenter 占用） |
| `--mcp-server-no-file-tools` | 不暴露 UCAgent 文件操作工具（后端自己处理文件） |
| `--exit-on-completion` | 全部阶段完成后自动退出 |
| `--no-embed-tools` | 禁用 UCAgent 内嵌工具（由后端 CLI 处理） |
| `--force-stage-index N` | 从第 N 个阶段开始（用于单阶段重跑） |
| `-s` | 流式输出到控制台 |

后端 CLI 的额外参数通过环境变量传递：

```bash
# Codex: 指定模型
export UC_ENV_CMD_BACKEND_EX_ARGS="-m gpt-5.4-mini --ephemeral"

# opencode: 通常留空
export UC_ENV_CMD_BACKEND_EX_ARGS=""
```

## 4. 运行

### 4.1 完整运行（所有阶段）

以 Codex 后端为例：

```bash
cd /Users/zzy/Workspace/ucagent
source .venv/bin/activate

UCAGENT_CMDLINE_START_MCP=1 \
UC_ENV_CMD_BACKEND_EX_ARGS="-m gpt-5.4-mini --ephemeral" \
ucagent <workspace> <dut> \
  --config <config.yaml> \
  --backend codex \
  --exit-on-completion \
  --mcp-server-no-file-tools \
  --mcp-server-host 127.0.0.1 \
  --mcp-server-port 5002 \
  --no-embed-tools \
  -s
```

### 4.2 单阶段运行

使用 `--force-stage-index` 从指定阶段开始，配合 `AGENTS.md` 中的指示
（Complete 后立即 Exit），实现单阶段重跑：

```bash
UCAGENT_CMDLINE_START_MCP=1 \
UC_ENV_CMD_BACKEND_EX_ARGS="-m gpt-5.4-mini --ephemeral" \
ucagent <workspace> <dut> \
  --config <config.yaml> \
  --backend codex \
  --exit-on-completion \
  --mcp-server-no-file-tools \
  --mcp-server-host 127.0.0.1 \
  --mcp-server-port 5002 \
  --force-stage-index <N> \
  --no-embed-tools \
  -s
```

本项目提供的辅助脚本 `scripts/run_ucagent_stage.sh` 封装了单阶段运行：

```bash
scripts/run_ucagent_stage.sh <stage_index>
```

### 4.3 预期日志

成功运行时日志应包含：

```text
FastMCP server started at 127.0.0.1:5002
Uvicorn running on http://127.0.0.1:5002
```

随后每个阶段：

1. UCAgent 生成后端 MCP 配置文件
2. 调用后端 CLI 执行任务
3. 后端连接 UCAgent MCP，调用工具链
4. UCAgent 推进到下一阶段

```text
mcp: ucagent/RoleInfo completed
mcp: ucagent/CurrentTips completed
mcp: ucagent/ReadTextFile completed
mcp: ucagent/SetCurrentStageJournal completed
mcp: ucagent/Complete completed
```

最终 `Exit` 后 UCAgent 退出。

## 5. UCAgent MCP 工具一览

后端 CLI 通过 MCP 调用以下 UCAgent 工具：

| 工具 | 作用 |
|------|------|
| `RoleInfo` | 获取当前 UCAgent 角色信息和基本指导 |
| `CurrentTips` | 获取当前阶段的详细任务指导 |
| `Detail` | 获取 Mission 详情和当前进度 |
| `Status` | 获取 Mission 摘要和阶段状态 |
| `ReadTextFile` | 读取文件（让 UCAgent 知晓你读了哪些文件） |
| `RunTestCases` | 运行测试用例 |
| `Check` | 检查当前阶段是否完成（不推进阶段） |
| `Complete` | 检查并推进到下一阶段 |
| `GoToStage` | 跳转到指定阶段 |
| `Exit` | 退出当前任务 |
| `SetCurrentStageJournal` | 记录当前阶段日志 |
| `AllStageJournal` | 获取所有阶段日志 |
| `StageJournal` | 获取指定阶段日志 |

## 6. 常见问题

### 6.1 端口 5000 被占用

macOS ControlCenter 默认监听 5000。改用 5002：

```bash
lsof -nP -iTCP:5002 -sTCP:LISTEN
```

### 6.2 后端找不到 MCP 工具

确认 `UCAGENT_CMDLINE_START_MCP=1` 已设置，且日志中出现了
`FastMCP server started`。如果 MCP Server 没有在后端启动前起来，后端
无法连接 UCAgent 工具。

### 6.3 `File(s) <DUT>/__init__.py do not exist`

UCAgent 要求 workspace 下存在 `<DUT>/<DUT>/__init__.py`（即使为空）：

```bash
mkdir -p <workspace>/<DUT>/<DUT>
touch <workspace>/<DUT>/<DUT>/__init__.py
```

### 6.4 opencode.json / config.toml 没有生成

- opencode 后端：确认 `ucagent/assets/mcp_opencode.json` 模板存在
- codex 后端：`pre_bash_cmd` 中的 `echo` 命令负责生成，检查 `{CWD}` 和
  `{PORT}` 占位符是否正确替换
- claude 后端：确认 `~/.claude/.mcp.json` 存在（`pre_bash_cmd` 会复制它）

### 6.5 后端 CLI 报错 / 非零退出

UCAgent 会捕获后端退出码。如果后端频繁报错，检查：

- 后端 CLI 自身模型配置
- 网络连通性（模型 API 可达）
- 先手动运行 `codex exec -p "hello"` / `opencode run -p "hello"` /
  `claude -p "hello"` 确认基本功能正常

## 7. 参考链接

- UCAgent MCP 集成官方文档：<https://ucagent.open-verify.cc/content/02_usage/00_mcp/>
- UCAgent 仓库：<https://github.com/XS-MLVP/UCAgent>
- opencode 配置 schema：<https://opencode.ai/config.json>
- Codex 后端已验证记录：`instruction.md`
- UCAgent 默认后端配置：`ucagent/setting.yaml`
