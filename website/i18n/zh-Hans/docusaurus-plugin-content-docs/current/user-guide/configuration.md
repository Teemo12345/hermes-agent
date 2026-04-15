---
sidebar_position: 2
title: "配置"
description: "配置 Hermes Agent — config.yaml、提供商、模型、API 密钥等"
---

# 配置

所有设置都存储在 `~/.hermes/` 目录中，便于访问。

## 目录结构

```text
~/.hermes/
├── config.yaml     # 设置（模型、终端、TTS、压缩等）
├── .env            # API 密钥和机密信息
├── auth.json       # OAuth 提供商凭据（Nous Portal 等）
├── SOUL.md         # 主要智能体身份（系统提示中的插槽 #1）
├── memories/       # 持久化内存（MEMORY.md、USER.md）
├── skills/         # 智能体创建的技能（通过 skill_manage 工具管理）
├── cron/           # 定时任务
├── sessions/       # 网关会话
└── logs/           # 日志（errors.log、gateway.log — 机密信息自动脱敏）
```

## 管理配置

```bash
hermes config              # 查看当前配置
hermes config edit         # 在编辑器中打开 config.yaml
hermes config set KEY VAL  # 设置特定值
hermes config check        # 检查缺失选项（更新后）
hermes config migrate      # 交互式添加缺失选项

# 示例：
hermes config set model anthropic/claude-opus-4
hermes config set terminal.backend docker
hermes config set OPENROUTER_API_KEY sk-or-...  # 保存到 .env
```

:::tip
`hermes config set` 命令会自动将值路由到正确的文件 — API 密钥保存到 `.env`，其他所有内容保存到 `config.yaml`。
:::

## 配置优先级

设置按以下顺序解析（优先级从高到低）：

1. **CLI 参数** — 例如，`hermes chat --model anthropic/claude-sonnet-4`（每次调用覆盖）
2. **`~/.hermes/config.yaml`** — 所有非机密设置的主要配置文件
3. **`~/.hermes/.env`** — 环境变量的后备；**必需**用于机密信息（API 密钥、令牌、密码）
4. **内置默认值** — 当没有设置其他内容时的硬编码安全默认值

:::info 经验法则
机密信息（API 密钥、机器人令牌、密码）放在 `.env` 中。其他所有内容（模型、终端后端、压缩设置、内存限制、工具集）放在 `config.yaml` 中。当两者都设置时，`config.yaml` 对非机密设置具有优先权。
:::

## 环境变量替换

您可以在 `config.yaml` 中使用 `${VAR_NAME}` 语法引用环境变量：

```yaml
auxiliary:
  vision:
    api_key: ${GOOGLE_API_KEY}
    base_url: ${CUSTOM_VISION_URL}

delegation:
  api_key: ${DELEGATION_KEY}
```

单个值中的多个引用有效：`url: "${HOST}:${PORT}"`。如果引用的变量未设置，占位符将保持原样（`${UNDEFINED_VAR}` 保持不变）。仅支持 `${VAR}` 语法 — 裸 `$VAR` 不会被展开。

有关 AI 提供商设置（OpenRouter、Anthropic、Copilot、自定义端点、自托管 LLM、备用模型等），请参阅 [AI 提供商](/docs/integrations/providers)。

## 终端后端配置

Hermes 支持六种终端后端。每种决定了智能体的 shell 命令实际执行的位置 — 您的本地机器、Docker 容器、通过 SSH 的远程服务器、Modal 云沙盒、Daytona 工作区或 Singularity/Apptainer 容器。

```yaml
terminal:
  backend: local    # local | docker | ssh | modal | daytona | singularity
  cwd: "."          # 工作目录（"." = 本地当前目录，容器为 "/root"）
  timeout: 180      # 每个命令的超时时间（秒）
  env_passthrough: []  # 要转发到沙盒化执行的环境变量名称（终端 + execute_code）
  singularity_image: "docker://nikolaik/python-nodejs:python3.11-nodejs20"  # Singularity 后端的容器镜像
  modal_image: "nikolaik/python-nodejs:python3.11-nodejs20"                 # Modal 后端的容器镜像
  daytona_image: "nikolaik/python-nodejs:python3.11-nodejs20"               # Daytona 后端的容器镜像
```

对于云沙盒（如 Modal 和 Daytona），`container_persistent: true` 意味着 Hermes 将尝试在沙盒重新创建时保留文件系统状态。它不保证相同的实时沙盒、PID 空间或后台进程稍后仍在运行。

### 后端概览

| 后端 | 命令运行位置 | 隔离性 | 最适合 |
|---------|-------------------|-----------|----------|
| **local** | 直接在您的机器上 | 无 | 开发、个人使用 |
| **docker** | Docker 容器 | 完整（命名空间、cap-drop） | 安全沙盒化、CI/CD |
| **ssh** | 通过 SSH 的远程服务器 | 网络边界 | 远程开发、强大硬件 |
| **modal** | Modal 云沙盒 | 完整（云虚拟机） | 临时云计算、评估 |
| **daytona** | Daytona 工作区 | 完整（云容器） | 托管云开发环境 |
| **singularity** | Singularity/Apptainer 容器 | 命名空间（--containall） | HPC 集群、共享机器 |

### 本地后端

默认设置。命令直接在您的机器上运行，没有隔离。无需特殊设置。

```yaml
terminal:
  backend: local
```

:::warning
智能体具有与您的用户帐户相同的文件系统访问权限。使用 `hermes tools` 禁用您不想要的工具，或切换到 Docker 进行沙盒化。
:::

### Docker 后端

在具有安全加固的 Docker 容器内运行命令（所有能力被丢弃、无权限提升、PID 限制）。

```yaml
terminal:
  backend: docker
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: false  # 将启动目录挂载到 /workspace
  docker_forward_env:              # 要转发到容器的环境变量名称
    - "GITHUB_TOKEN"
  docker_volumes:                  # 主机目录挂载
    - "/home/user/projects:/workspace/projects"
    - "/home/user/data:/data:ro"   # :ro 表示只读

  # 资源限制
  container_cpu: 1                 # CPU 核心数（0 = 无限制）
  container_memory: 5120           # MB（0 = 无限制）
  container_disk: 51200            # MB（需要在 XFS+pquota 上使用 overlay2）
  container_persistent: true       # 跨会话持久化 /workspace 和 /root
```

**要求：** Docker Desktop 或 Docker Engine 已安装并运行。Hermes 探测 `$PATH` 加上常见的 macOS 安装位置（`/usr/local/bin/docker`、`/opt/homebrew/bin/docker`、Docker Desktop 应用程序包）。

**容器生命周期：** 每个会话启动一个长期存在的容器（`docker run -d ... sleep 2h`）。命令通过 `docker exec` 使用登录 shell 运行。清理时，容器被停止并移除。

**安全加固：**
- `--cap-drop ALL`，仅添加回 `DAC_OVERRIDE`、`CHOWN`、`FOWNER`
- `--security-opt no-new-privileges`
- `--pids-limit 256`
- 大小受限的 tmpfs：`/tmp`（512MB）、`/var/tmp`（256MB）、`/run`（64MB）

**凭据转发：** 列在 `docker_forward_env` 中的环境变量首先从您的 shell 环境解析，然后从 `~/.hermes/.env` 解析。技能也可以声明 `required_environment_variables`，这些变量会自动合并。

### SSH 后端

通过 SSH 在远程服务器上运行命令。使用 ControlMaster 进行连接复用（5 分钟空闲保持）。默认启用持久 shell — 状态（cwd、环境变量）在命令间保持。

```yaml
terminal:
  backend: ssh
  persistent_shell: true           # 保持长期存在的 bash 会话（默认：true）
```

**必需的环境变量：**

```bash
TERMINAL_SSH_HOST=my-server.example.com
TERMINAL_SSH_USER=ubuntu
```

**可选：**

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `TERMINAL_SSH_PORT` | `22` | SSH 端口 |
| `TERMINAL_SSH_KEY` | （系统默认） | SSH 私钥路径 |
| `TERMINAL_SSH_PERSISTENT` | `true` | 启用持久 shell |

**工作原理：** 在初始化时使用 `BatchMode=yes` 和 `StrictHostKeyChecking=accept-new` 连接。持久 shell 在远程主机上保持单个 `bash -l` 进程存活，通过临时文件通信。需要 `stdin_data` 或 `sudo` 的命令自动回退到一次性模式。

### Modal 后端

在 [Modal](https://modal.com) 云沙盒中运行命令。每个任务获得一个具有可配置 CPU、内存和磁盘的隔离 VM。文件系统可以在会话间快照/恢复。

```yaml
terminal:
  backend: modal
  container_cpu: 1                 # CPU 核心数
  container_memory: 5120           # MB（5GB）
  container_disk: 51200            # MB（50GB）
  container_persistent: true       # 快照/恢复文件系统
```

**必需：** 要么是 `MODAL_TOKEN_ID` + `MODAL_TOKEN_SECRET` 环境变量，要么是 `~/.modal.toml` 配置文件。

**持久性：** 启用后，沙盒文件系统在清理时进行快照，并在下次会话时恢复。快照在 `~/.hermes/modal_snapshots.json` 中跟踪。这保留了文件系统状态，而不是实时进程、PID 空间或后台作业。

**凭据文件：** 自动从 `~/.hermes/`（OAuth 令牌等）挂载，并在每个命令前同步。

### Daytona 后端

在 [Daytona](https://daytona.io) 托管的工作区中运行命令。支持停止/恢复以实现持久性。

```yaml
terminal:
  backend: daytona
  container_cpu: 1                 # CPU 核心数
  container_memory: 5120           # MB → 转换为 GiB
  container_disk: 10240            # MB → 转换为 GiB（最大 10 GiB）
  container_persistent: true       # 停止/恢复而不是删除
```

**必需：** `DAYTONA_API_KEY` 环境变量。

**持久性：** 启用后，沙盒在清理时被停止（不删除），并在下次会话时恢复。沙盒名称遵循模式 `hermes-{task_id}`。

**磁盘限制：** Daytona 强制执行 10 GiB 的最大值。超过此值的请求会被限制并发出警告。

### Singularity/Apptainer 后端

在 [Singularity/Apptainer](https://apptainer.org) 容器中运行命令。专为 HPC 集群和共享机器设计，这些地方 Docker 不可用。

```yaml
terminal:
  backend: singularity
  singularity_image: "docker://nikolaik/python-nodejs:python3.11-nodejs20"
  container_cpu: 1                 # CPU 核心数
  container_memory: 5120           # MB
  container_persistent: true       # 可写覆盖层在会话间持久化
```

**要求：** `apptainer` 或 `singularity` 二进制文件在 `$PATH` 中。

**镜像处理：** Docker URL（`docker://...`）自动转换为 SIF 文件并缓存。现有的 `.sif` 文件直接使用。

**临时目录：** 按顺序解析：`TERMINAL_SCRATCH_DIR` → `TERMINAL_SANDBOX_DIR/singularity` → `/scratch/$USER/hermes-agent`（HPC 约定）→ `~/.hermes/sandboxes/singularity`。

**隔离：** 使用 `--containall --no-home` 实现完整的命名空间隔离，不挂载主机主目录。

### 常见终端后端问题

如果终端命令立即失败或终端工具报告为禁用：

- **Local** — 无特殊要求。开始使用时的最安全默认值。
- **Docker** — 运行 `docker version` 验证 Docker 是否正常工作。如果失败，修复 Docker 或 `hermes config set terminal.backend local`。
- **SSH** — `TERMINAL_SSH_HOST` 和 `TERMINAL_SSH_USER` 都必须设置。如果缺少任一，Hermes 会记录清晰的错误。
- **Modal** — 需要 `MODAL_TOKEN_ID` 环境变量或 `~/.modal.toml`。运行 `hermes doctor` 检查。
- **Daytona** — 需要 `DAYTONA_API_KEY`。Daytona SDK 处理服务器 URL 配置。
- **Singularity** — 需要 `apptainer` 或 `singularity` 在 `$PATH` 中。在 HPC 集群上常见。

有疑问时，将 `terminal.backend` 设置回 `local` 并首先验证命令在那里运行。

### Docker 卷挂载

使用 Docker 后端时，`docker_volumes` 允许您与容器共享主机目录。每个条目使用标准 Docker `-v` 语法：`host_path:container_path[:options]`。

```yaml
terminal:
  backend: docker
  docker_volumes:
    - "/home/user/projects:/workspace/projects"   # 读写（默认）
    - "/home/user/datasets:/data:ro"              # 只读
    - "/home/user/outputs:/outputs"               # 智能体写入，您读取
```

这适用于：
- **向智能体提供文件**（数据集、配置、参考代码）
- **从智能体接收文件**（生成的代码、报告、导出）
- **共享工作区**，您和智能体都可以访问相同的文件

也可以通过环境变量设置：`TERMINAL_DOCKER_VOLUMES='["/host:/container"]'`（JSON 数组）。

### Docker 凭据转发

默认情况下，Docker 终端会话不继承任意主机凭据。如果您需要在容器内使用特定令牌，请将其添加到 `terminal.docker_forward_env`。

```yaml
terminal:
  backend: docker
  docker_forward_env:
    - "GITHUB_TOKEN"
    - "NPM_TOKEN"
```

Hermes 首先从您当前的 shell 解析每个列出的变量，然后如果使用 `hermes config set` 保存，则回退到 `~/.hermes/.env`。

:::warning
列在 `docker_forward_env` 中的任何内容都会对容器内运行的命令可见。仅转发您愿意暴露给终端会话的凭据。
:::

### 可选：将启动目录挂载到 `/workspace`

默认情况下，Docker 沙盒保持隔离。除非您明确选择加入，否则 Hermes **不会**将您当前的主机工作目录传递到容器中。

在 `config.yaml` 中启用：

```yaml
terminal:
  backend: docker
  docker_mount_cwd_to_workspace: true
```

启用时：
- 如果您从 `~/projects/my-app` 启动 Hermes，该主机目录将绑定挂载到 `/workspace`
- Docker 后端在 `/workspace` 中启动
- 文件工具和终端命令都看到相同的挂载项目

禁用时，`/workspace` 保持沙盒所有，除非您通过 `docker_volumes` 显式挂载某些内容。

安全权衡：
- `false` 保留沙盒边界
- `true` 让沙盒直接访问您启动 Hermes 的目录

仅在您有意希望容器处理实时主机文件时使用选择加入。

### 持久 Shell

默认情况下，每个终端命令在自己的子进程中运行 — 工作目录、环境变量和 shell 变量在命令间重置。当启用**持久 shell**时，单个长期存在的 bash 进程在 `execute()` 调用间保持存活，以便状态在命令间保持。

这对于**SSH 后端**最有用，它还消除了每个命令的连接开销。持久 shell **默认对 SSH 启用**，对本地后端禁用。

```yaml
terminal:
  persistent_shell: true   # 默认 — 为 SSH 启用持久 shell
```

要禁用：

```bash
hermes config set terminal.persistent_shell false
```

**在命令间保持的内容：**
- 工作目录（`cd /tmp` 对下一个命令保持）
- 导出的环境变量（`export FOO=bar`）
- Shell 变量（`MY_VAR=hello`）

**优先级：**

| 级别 | 变量 | 默认值 |
|-------|----------|---------|
| 配置 | `terminal.persistent_shell` | `true` |
| SSH 覆盖 | `TERMINAL_SSH_PERSISTENT` | 遵循配置 |
| 本地覆盖 | `TERMINAL_LOCAL_PERSISTENT` | `false` |

每个后端的环境变量具有最高优先级。如果您也想在本地后端启用持久 shell：

```bash
export TERMINAL_LOCAL_PERSISTENT=true
```

:::note
需要 `stdin_data` 或 sudo 的命令自动回退到一次性模式，因为持久 shell 的 stdin 已被 IPC 协议占用。
:::

有关每个后端的详细信息，请参阅[代码执行](features/code-execution.md)和 README 的[终端部分](features/tools.md)。

## 技能设置

技能可以通过其 SKILL.md 前置声明自己的配置设置。这些是非机密值（路径、偏好、域设置），存储在 `config.yaml` 中的 `skills.config` 命名空间下。

```yaml
skills:
  config:
    wiki:
      path: ~/wiki          # 由 llm-wiki 技能使用
```

**技能设置工作原理：**

- `hermes config migrate` 扫描所有启用的技能，找到未配置的设置，并提供提示您
- `hermes config show` 在"技能设置"下显示所有技能设置及其所属技能
- 当技能加载时，其解析的配置值会自动注入到技能上下文中

**手动设置值：**

```bash
hermes config set skills.config.wiki.path ~/my-research-wiki
```

有关在您自己的技能中声明配置设置的详细信息，请参阅[创建技能 — 配置设置](/docs/developer-guide/creating-skills#config-settings-configyaml)。

## 内存配置

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200   # ~800 tokens
  user_char_limit: 1375     # ~500 tokens
```

## 文件读取安全

控制单个 `read_file` 调用可以返回多少内容。超过限制的读取会被拒绝，并提示智能体使用 `offset` 和 `limit` 获取较小范围。这可以防止单个读取压缩的 JS 包或大型数据文件淹没上下文窗口。

```yaml
file_read_max_chars: 100000  # 默认 — ~25-35K tokens
```

如果您使用具有大上下文窗口的模型并经常读取大文件，请提高此值。对于小上下文模型，降低此值以保持读取效率：

```yaml
# 大上下文模型（200K+）
file_read_max_chars: 200000

# 小本地模型（16K 上下文）
file_read_max_chars: 30000
```

智能体还会自动去重文件读取 — 如果同一文件区域被读取两次且文件未更改，则返回轻量级存根而不是重新发送内容。这在上下文压缩时重置，因此智能体可以在其内容被摘要后重新读取文件。

## Git 工作树隔离

启用隔离的 git 工作树，以便在同一仓库上并行运行多个智能体：

```yaml
worktree: true    # 始终创建工作树（与 hermes -w 相同）
# worktree: false # 默认 — 仅在传递 -w 标志时
```

启用后，每个 CLI 会话会在 `.worktrees/` 下创建一个具有自己分支的新工作树。智能体可以编辑文件、提交、推送和创建 PR，而不会相互干扰。干净的工作树在退出时被移除；脏的工作树保留用于手动恢复。

您还可以通过仓库根目录中的 `.worktreeinclude` 列出要复制到工作树中的 gitignored 文件：

```
# .worktreeinclude
.env
.venv/
node_modules/
```

## 上下文压缩

Hermes 自动压缩长对话以保持在您模型的上下文窗口内。压缩摘要器是一个单独的 LLM 调用 — 您可以将其指向任何提供商或端点。

所有压缩设置都在 `config.yaml` 中（无环境变量）。

### 完整参考

```yaml
compression:
  enabled: true                                     # 切换压缩开/关
  threshold: 0.50                                   # 在此 % 的上下文限制处压缩
  target_ratio: 0.20                                # 保留为最近尾部的阈值分数
  protect_last_n: 20                                # 保持未压缩的最小最近消息数

# 摘要模型/提供商在 auxiliary 下配置：
auxiliary:
  compression:
    model: "google/gemini-3-flash-preview"          # 用于摘要的模型
    provider: "auto"                                # 提供商："auto"、"openrouter"、"nous"、"codex"、"main" 等
    base_url: null                                  # 自定义 OpenAI 兼容端点（覆盖提供商）
```

:::info 旧配置迁移
具有 `compression.summary_model`、`compression.summary_provider` 和 `compression.summary_base_url` 的旧配置在首次加载时自动迁移到 `auxiliary.compression.*`（配置版本 17）。无需手动操作。
:::

### 常见设置

**默认（自动检测）— 无需配置：**
```yaml
compression:
  enabled: true
  threshold: 0.50
```
使用第一个可用的提供商（OpenRouter → Nous → Codex）和 Gemini Flash。

**强制特定提供商**（基于 OAuth 或 API 密钥）：
```yaml
auxiliary:
  compression:
    provider: nous
    model: gemini-3-flash
```
适用于任何提供商：`nous`、`openrouter`、`codex`、`anthropic`、`main` 等。

**自定义端点**（自托管、Ollama、zai、DeepSeek 等）：
```yaml
auxiliary:
  compression:
    model: glm-4.7
    base_url: https://api.z.ai/api/coding/paas/v4
```
指向自定义 OpenAI 兼容端点。使用 `OPENAI_API_KEY` 进行认证。

### How the three knobs interact

| `auxiliary.compression.provider` | `auxiliary.compression.base_url` | Result |
|---------------------|---------------------|--------|
| `auto` (default) | not set | Auto-detect best available provider |
| `nous` / `openrouter` / etc. | not set | Force that provider, use its auth |
| any | set | Use the custom endpoint directly (provider ignored) |

:::warning Summary model context length requirement
The summary model **must** have a context window at least as large as your main agent model's. The compressor sends the full middle section of the conversation to the summary model — if that model's context window is smaller than the main model's, the summarization call will fail with a context length error. When this happens, the middle turns are **dropped without a summary**, losing conversation context silently. If you override the model, verify its context length meets or exceeds your main model's.
:::

## Context Engine

The context engine controls how conversations are managed when approaching the model's token limit. The built-in `compressor` engine uses lossy summarization (see [Context Compression](/docs/developer-guide/context-compression-and-caching)). Plugin engines can replace it with alternative strategies.

```yaml
context:
  engine: "compressor"    # default — built-in lossy summarization
```

To use a plugin engine (e.g., LCM for lossless context management):

```yaml
context:
  engine: "lcm"          # must match the plugin's name
```

Plugin engines are **never auto-activated** — you must explicitly set `context.engine` to the plugin name. Available engines can be browsed and selected via `hermes plugins` → Provider Plugins → Context Engine.

See [Memory Providers](/docs/user-guide/features/memory-providers) for the analogous single-select system for memory plugins.

## Iteration Budget Pressure

When the agent is working on a complex task with many tool calls, it can burn through its iteration budget (default: 90 turns) without realizing it's running low. Budget pressure automatically warns the model as it approaches the limit:

| Threshold | Level | What the model sees |
|-----------|-------|---------------------|
| **70%** | Caution | `[BUDGET: 63/90. 27 iterations left. Start consolidating.]` |
| **90%** | Warning | `[BUDGET WARNING: 81/90. Only 9 left. Respond NOW.]` |

Warnings are injected into the last tool result's JSON (as a `_budget_warning` field) rather than as separate messages — this preserves prompt caching and doesn't disrupt the conversation structure.

```yaml
agent:
  max_turns: 90                # Max iterations per conversation turn (default: 90)
```

Budget pressure is enabled by default. The agent sees warnings naturally as part of tool results, encouraging it to consolidate its work and deliver a response before running out of iterations.

When the iteration budget is fully exhausted, the CLI shows a notification to the user: `⚠ Iteration budget reached (90/90) — response may be incomplete`. If the budget runs out during active work, the agent generates a summary of what was accomplished before stopping.

### Streaming Timeouts

The LLM streaming connection has two timeout layers. Both auto-adjust for local providers (localhost, LAN IPs) — no configuration needed for most setups.

| Timeout | Default | Local providers | Env var |
|---------|---------|----------------|---------|
| Socket read timeout | 120s | Auto-raised to 1800s | `HERMES_STREAM_READ_TIMEOUT` |
| Stale stream detection | 180s | Auto-disabled | `HERMES_STREAM_STALE_TIMEOUT` |
| API call (non-streaming) | 1800s | Unchanged | `HERMES_API_TIMEOUT` |

The **socket read timeout** controls how long httpx waits for the next chunk of data from the provider. Local LLMs can take minutes for prefill on large contexts before producing the first token, so Hermes raises this to 30 minutes when it detects a local endpoint. If you explicitly set `HERMES_STREAM_READ_TIMEOUT`, that value is always used regardless of endpoint detection.

The **stale stream detection** kills connections that receive SSE keep-alive pings but no actual content. This is disabled entirely for local providers since they don't send keep-alive pings during prefill.

## Context Pressure Warnings

Separate from iteration budget pressure, context pressure tracks how close the conversation is to the **compaction threshold** — the point where context compression fires to summarize older messages. This helps both you and the agent understand when the conversation is getting long.

| Progress | Level | What happens |
|----------|-------|-------------|
| **≥ 60%** to threshold | Info | CLI shows a cyan progress bar; gateway sends an informational notice |
| **≥ 85%** to threshold | Warning | CLI shows a bold yellow bar; gateway warns compaction is imminent |

In the CLI, context pressure appears as a progress bar in the tool output feed:

```
  ◐ context ████████████░░░░░░░░ 62% to compaction  48k threshold (50%) · approaching compaction
```

On messaging platforms, a plain-text notification is sent:

```
◐ Context: ████████████░░░░░░░░ 62% to compaction (threshold: 50% of window).
```

如果自动压缩被禁用，警告会告诉您上下文可能会被截断。

上下文压力是自动的 — 无需配置。它纯粹作为面向用户的通知触发，不会修改消息流或向模型的上下文中注入任何内容。

## 凭据池策略

当您有多个相同提供商的 API 密钥或 OAuth 令牌时，配置轮换策略：

```yaml
credential_pool_strategies:
  openrouter: round_robin    # 均匀循环使用密钥
  anthropic: least_used      # 总是选择使用最少的密钥
```

选项：`fill_first`（默认）、`round_robin`、`least_used`、`random`。完整文档请参阅[凭据池](/docs/user-guide/features/credential-pools)。

## 辅助模型

Hermes 使用轻量级"辅助"模型来处理图像分析、网页摘要和浏览器截图分析等辅助任务。默认情况下，这些通过自动检测使用**Gemini Flash** — 您无需配置任何内容。

### 通用配置模式

Hermes 中的每个模型插槽 — 辅助任务、压缩、备用 — 都使用相同的三个控制旋钮：

| 键 | 作用 | 默认值 |
|-----|-------------|---------|
| `provider` | 用于认证和路由的提供商 | `"auto"` |
| `model` | 请求的模型 | 提供商的默认值 |
| `base_url` | 自定义 OpenAI 兼容端点（覆盖提供商） | 未设置 |

当设置 `base_url` 时，Hermes 忽略提供商并直接调用该端点（使用 `api_key` 或 `OPENAI_API_KEY` 进行认证）。当仅设置 `provider` 时，Hermes 使用该提供商的内置认证和基础 URL。

辅助任务的可用提供商：`auto`、`openrouter`、`nous`、`codex`、`copilot`、`anthropic`、`main`、`zai`、`kimi-coding`、`kimi-coding-cn`、`arcee`、`minimax`、[提供商注册表](/docs/reference/environment-variables)中注册的任何提供商，或来自您的 `custom_providers` 列表的任何命名自定义提供商（例如 `provider: "beans"`）。

:::warning `"main"` 仅用于辅助任务
`"main"` 提供商选项意味着"使用我的主智能体使用的任何提供商" — 它仅在 `auxiliary:`、`compression:` 和 `fallback_model:` 配置中有效。它不是您顶级 `model.provider` 设置的有效值。如果您使用自定义 OpenAI 兼容端点，请在 `model:` 部分设置 `provider: custom`。所有主模型提供商选项请参阅 [AI 提供商](/docs/integrations/providers)。
:::

### 完整辅助配置参考

```yaml
auxiliary:
  # 图像分析（vision_analyze 工具 + 浏览器截图）
  vision:
    provider: "auto"           # "auto"、"openrouter"、"nous"、"codex"、"main" 等
    model: ""                  # 例如 "openai/gpt-4o"、"google/gemini-2.5-flash"
    base_url: ""               # 自定义 OpenAI 兼容端点（覆盖提供商）
    api_key: ""                # base_url 的 API 密钥（回退到 OPENAI_API_KEY）
    timeout: 120               # 秒 — LLM API 调用超时；视觉负载需要宽松的超时
    download_timeout: 30       # 秒 — 图像 HTTP 下载；为慢速连接增加

  # 网页摘要 + 浏览器页面文本提取
  web_extract:
    provider: "auto"
    model: ""                  # 例如 "google/gemini-2.5-flash"
    base_url: ""
    api_key: ""
    timeout: 360               # 秒（6分钟）— 每次尝试的 LLM 摘要

  # 危险命令批准分类器
  approval:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 30                # 秒

  # 上下文压缩超时（与 compression.* 配置分开）
  compression:
    timeout: 120               # 秒 — 压缩摘要长对话，需要更多时间

  # 会话搜索 — 摘要过去的会话匹配
  session_search:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 30

  # 技能中心 — 技能匹配和搜索
  skills_hub:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 30

  # MCP 工具分发
  mcp:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 30

  # 内存刷新 — 为持久化内存摘要对话
  flush_memories:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 30
```

:::tip
每个辅助任务都有可配置的 `timeout`（以秒为单位）。默认值：vision 120秒、web_extract 360秒、approval 30秒、compression 120秒。如果您为辅助任务使用慢速本地模型，请增加这些值。Vision 还有一个单独的 `download_timeout`（默认 30秒）用于 HTTP 图像下载 — 为慢速连接或自托管图像服务器增加此值。
:::

:::info
上下文压缩有自己的 `compression:` 块用于阈值，以及 `auxiliary.compression:` 块用于模型/提供商设置 — 请参阅上面的[上下文压缩](#context-compression)。备用模型使用 `fallback_model:` 块 — 请参阅[备用模型](/docs/integrations/providers#fallback-model)。所有三个都遵循相同的 provider/model/base_url 模式。
:::

### 更改 Vision 模型

要使用 GPT-4o 而不是 Gemini Flash 进行图像分析：

```yaml
auxiliary:
  vision:
    model: "openai/gpt-4o"
```

或通过环境变量（在 `~/.hermes/.env` 中）：

```bash
AUXILIARY_VISION_MODEL=openai/gpt-4o
```

### 提供商选项

这些选项适用于**辅助任务配置**（`auxiliary:`、`compression:`、`fallback_model:`），而不是您的主 `model.provider` 设置。

| 提供商 | 描述 | 要求 |
|----------|-------------|-------------|
| `"auto"` | 最佳可用（默认）。Vision 尝试 OpenRouter → Nous → Codex。 | — |
| `"openrouter"` | 强制 OpenRouter — 路由到任何模型（Gemini、GPT-4o、Claude 等） | `OPENROUTER_API_KEY` |
| `"nous"` | 强制 Nous Portal | `hermes auth` |
| `"codex"` | 强制 Codex OAuth（ChatGPT 帐户）。支持 vision（gpt-5.3-codex）。 | `hermes model` → Codex |
| `"main"` | 使用您活动的自定义/主端点。这可以来自 `OPENAI_BASE_URL` + `OPENAI_API_KEY` 或通过 `hermes model` / `config.yaml` 保存的自定义端点。适用于 OpenAI、本地模型或任何 OpenAI 兼容 API。**仅用于辅助任务 — 对 `model.provider` 无效。** | 自定义端点凭据 + 基础 URL |

### 常见设置

**使用直接自定义端点**（比 `provider: "main"` 更清晰，适用于本地/自托管 API）：
```yaml
auxiliary:
  vision:
    base_url: "http://localhost:1234/v1"
    api_key: "local-key"
    model: "qwen2.5-vl"
```

`base_url` 优先于 `provider`，因此这是将辅助任务路由到特定端点的最明确方式。对于直接端点覆盖，Hermes 使用配置的 `api_key` 或回退到 `OPENAI_API_KEY`；它不会为该自定义端点重用 `OPENROUTER_API_KEY`。

**使用 OpenAI API 密钥进行 vision：**
```yaml
# 在 ~/.hermes/.env 中：
# OPENAI_BASE_URL=https://api.openai.com/v1
# OPENAI_API_KEY=sk-...

auxiliary:
  vision:
    provider: "main"
    model: "gpt-4o"       # 或 "gpt-4o-mini" 更便宜
```

**使用 OpenRouter 进行 vision**（路由到任何模型）：
```yaml
auxiliary:
  vision:
    provider: "openrouter"
    model: "openai/gpt-4o"      # 或 "google/gemini-2.5-flash" 等
```

**使用 Codex OAuth**（ChatGPT Pro/Plus 帐户 — 无需 API 密钥）：
```yaml
auxiliary:
  vision:
    provider: "codex"     # 使用您的 ChatGPT OAuth 令牌
    # model 默认为 gpt-5.3-codex（支持 vision）
```

**使用本地/自托管模型：**
```yaml
auxiliary:
  vision:
    provider: "main"      # 使用您活动的自定义端点
    model: "my-local-model"
```

`provider: "main"` 使用 Hermes 用于正常聊天的任何提供商 — 无论是命名自定义提供商（例如 `beans`）、内置提供商如 `openrouter`，还是遗留的 `OPENAI_BASE_URL` 端点。

:::tip
如果您使用 Codex OAuth 作为您的主模型提供商，vision 会自动工作 — 无需额外配置。Codex 包含在 vision 的自动检测链中。
:::

:::warning
**Vision 需要多模态模型。**如果您设置 `provider: "main"`，请确保您的端点支持多模态/vision — 否则图像分析将失败。
:::

### Environment Variables (legacy)

Auxiliary models can also be configured via environment variables. However, `config.yaml` is the preferred method — it's easier to manage and supports all options including `base_url` and `api_key`.

| Setting | Environment Variable |
|---------|---------------------|
| Vision provider | `AUXILIARY_VISION_PROVIDER` |
| Vision model | `AUXILIARY_VISION_MODEL` |
| Vision endpoint | `AUXILIARY_VISION_BASE_URL` |
| Vision API key | `AUXILIARY_VISION_API_KEY` |
| Web extract provider | `AUXILIARY_WEB_EXTRACT_PROVIDER` |
| Web extract model | `AUXILIARY_WEB_EXTRACT_MODEL` |
| Web extract endpoint | `AUXILIARY_WEB_EXTRACT_BASE_URL` |
| Web extract API key | `AUXILIARY_WEB_EXTRACT_API_KEY` |

Compression and fallback model settings are config.yaml-only.

:::tip
Run `hermes config` to see your current auxiliary model settings. Overrides only show up when they differ from the defaults.
:::

## Reasoning Effort

Control how much "thinking" the model does before responding:

```yaml
agent:
  reasoning_effort: ""   # empty = medium (default). Options: none, minimal, low, medium, high, xhigh (max)
```

When unset (default), reasoning effort defaults to "medium" — a balanced level that works well for most tasks. Setting a value overrides it — higher reasoning effort gives better results on complex tasks at the cost of more tokens and latency.

You can also change the reasoning effort at runtime with the `/reasoning` command:

```
/reasoning           # Show current effort level and display state
/reasoning high      # Set reasoning effort to high
/reasoning none      # Disable reasoning
/reasoning show      # Show model thinking above each response
/reasoning hide      # Hide model thinking
```

## Tool-Use Enforcement

Some models occasionally describe intended actions as text instead of making tool calls ("I would run the tests..." instead of actually calling the terminal). Tool-use enforcement injects system prompt guidance that steers the model back to actually calling tools.

```yaml
agent:
  tool_use_enforcement: "auto"   # "auto" | true | false | ["model-substring", ...]
```

| Value | Behavior |
|-------|----------|
| `"auto"` (default) | Enabled for models matching: `gpt`, `codex`, `gemini`, `gemma`, `grok`. Disabled for all others (Claude, DeepSeek, Qwen, etc.). |
| `true` | Always enabled, regardless of model. Useful if you notice your current model describing actions instead of performing them. |
| `false` | Always disabled, regardless of model. |
| `["gpt", "codex", "qwen", "llama"]` | Enabled only when the model name contains one of the listed substrings (case-insensitive). |

### What it injects

When enabled, three layers of guidance may be added to the system prompt:

1. **General tool-use enforcement** (all matched models) — instructs the model to make tool calls immediately instead of describing intentions, keep working until the task is complete, and never end a turn with a promise of future action.

2. **OpenAI execution discipline** (GPT and Codex models only) — additional guidance addressing GPT-specific failure modes: abandoning work on partial results, skipping prerequisite lookups, hallucinating instead of using tools, and declaring "done" without verification.

3. **Google operational guidance** (Gemini and Gemma models only) — conciseness, absolute paths, parallel tool calls, and verify-before-edit patterns.

These are transparent to the user and only affect the system prompt. Models that already use tools reliably (like Claude) don't need this guidance, which is why `"auto"` excludes them.

### When to turn it on

If you're using a model not in the default auto list and notice it frequently describes what it *would* do instead of doing it, set `tool_use_enforcement: true` or add the model substring to the list:

```yaml
agent:
  tool_use_enforcement: ["gpt", "codex", "gemini", "grok", "my-custom-model"]
```

## TTS Configuration

```yaml
tts:
  provider: "edge"              # "edge" | "elevenlabs" | "openai" | "minimax" | "mistral" | "neutts"
  speed: 1.0                    # Global speed multiplier (fallback for all providers)
  edge:
    voice: "en-US-AriaNeural"   # 322 voices, 74 languages
    speed: 1.0                  # Speed multiplier (converted to rate percentage, e.g. 1.5 → +50%)
  elevenlabs:
    voice_id: "pNInz6obpgDQGcFmaJgB"
    model_id: "eleven_multilingual_v2"
  openai:
    model: "gpt-4o-mini-tts"
    voice: "alloy"              # alloy, echo, fable, onyx, nova, shimmer
    speed: 1.0                  # Speed multiplier (clamped to 0.25–4.0 by the API)
    base_url: "https://api.openai.com/v1"  # Override for OpenAI-compatible TTS endpoints
  minimax:
    speed: 1.0                  # Speech speed multiplier
    # base_url: ""              # Optional: override for OpenAI-compatible TTS endpoints
  neutts:
    ref_audio: ''
    ref_text: ''
    model: neuphonic/neutts-air-q4-gguf
    device: cpu
```

This controls both the `text_to_speech` tool and spoken replies in voice mode (`/voice tts` in the CLI or messaging gateway).

**Speed fallback hierarchy:** provider-specific speed (e.g. `tts.edge.speed`) → global `tts.speed` → `1.0` default. Set the global `tts.speed` to apply a uniform speed across all providers, or override per-provider for fine-grained control.

## Display Settings

```yaml
display:
  tool_progress: all      # off | new | all | verbose
  tool_progress_command: false  # Enable /verbose slash command in messaging gateway
  tool_progress_overrides: {}  # Per-platform overrides (see below)
  interim_assistant_messages: true  # Gateway: send natural mid-turn assistant updates as separate messages
  skin: default           # Built-in or custom CLI skin (see user-guide/features/skins)
  personality: "kawaii"  # Legacy cosmetic field still surfaced in some summaries
  compact: false          # Compact output mode (less whitespace)
  resume_display: full    # full (show previous messages on resume) | minimal (one-liner only)
  bell_on_complete: false # Play terminal bell when agent finishes (great for long tasks)
  show_reasoning: false   # Show model reasoning/thinking above each response (toggle with /reasoning show|hide)
  streaming: false        # Stream tokens to terminal as they arrive (real-time output)
  show_cost: false        # Show estimated $ cost in the CLI status bar
  tool_preview_length: 0  # Max chars for tool call previews (0 = no limit, show full paths/commands)
```

| Mode | What you see |
|------|-------------|
| `off` | Silent — just the final response |
| `new` | Tool indicator only when the tool changes |
| `all` | Every tool call with a short preview (default) |
| `verbose` | Full args, results, and debug logs |

In the CLI, cycle through these modes with `/verbose`. To use `/verbose` in messaging platforms (Telegram, Discord, Slack, etc.), set `tool_progress_command: true` in the `display` section above. The command will then cycle the mode and save to config.

### Per-platform progress overrides

Different platforms have different verbosity needs. For example, Signal can't edit messages, so each progress update becomes a separate message — noisy. Use `tool_progress_overrides` to set per-platform modes:

```yaml
display:
  tool_progress: all          # global default
  tool_progress_overrides:
    signal: 'off'             # silence progress on Signal
    telegram: verbose         # detailed progress on Telegram
    slack: 'off'              # quiet in shared Slack workspace
```

Platforms without an override fall back to the global `tool_progress` value. Valid platform keys: `telegram`, `discord`, `slack`, `signal`, `whatsapp`, `matrix`, `mattermost`, `email`, `sms`, `homeassistant`, `dingtalk`, `feishu`, `wecom`, `weixin`, `bluebubbles`, `qqbot`.

`interim_assistant_messages` is gateway-only. When enabled, Hermes sends completed mid-turn assistant updates as separate chat messages. This is independent from `tool_progress` and does not require gateway streaming.

## Privacy

```yaml
privacy:
  redact_pii: false  # Strip PII from LLM context (gateway only)
```

When `redact_pii` is `true`, the gateway redacts personally identifiable information from the system prompt before sending it to the LLM on supported platforms:

| Field | Treatment |
|-------|-----------|
| Phone numbers (user ID on WhatsApp/Signal) | Hashed to `user_<12-char-sha256>` |
| User IDs | Hashed to `user_<12-char-sha256>` |
| Chat IDs | Numeric portion hashed, platform prefix preserved (`telegram:<hash>`) |
| Home channel IDs | Numeric portion hashed |
| User names / usernames | **Not affected** (user-chosen, publicly visible) |

**Platform support:** Redaction applies to WhatsApp, Signal, and Telegram. Discord and Slack are excluded because their mention systems (`<@user_id>`) require the real ID in the LLM context.

Hashes are deterministic — the same user always maps to the same hash, so the model can still distinguish between users in group chats. Routing and delivery use the original values internally.

## Speech-to-Text (STT)

```yaml
stt:
  provider: "local"            # "local" | "groq" | "openai" | "mistral"
  local:
    model: "base"              # tiny, base, small, medium, large-v3
  openai:
    model: "whisper-1"         # whisper-1 | gpt-4o-mini-transcribe | gpt-4o-transcribe
  # model: "whisper-1"         # Legacy fallback key still respected
```

Provider behavior:

- `local` uses `faster-whisper` running on your machine. Install it separately with `pip install faster-whisper`.
- `groq` uses Groq's Whisper-compatible endpoint and reads `GROQ_API_KEY`.
- `openai` uses the OpenAI speech API and reads `VOICE_TOOLS_OPENAI_KEY`.

If the requested provider is unavailable, Hermes falls back automatically in this order: `local` → `groq` → `openai`.

Groq and OpenAI model overrides are environment-driven:

```bash
STT_GROQ_MODEL=whisper-large-v3-turbo
STT_OPENAI_MODEL=whisper-1
GROQ_BASE_URL=https://api.groq.com/openai/v1
STT_OPENAI_BASE_URL=https://api.openai.com/v1
```

## Voice Mode (CLI)

```yaml
voice:
  record_key: "ctrl+b"         # Push-to-talk key inside the CLI
  max_recording_seconds: 120    # Hard stop for long recordings
  auto_tts: false               # Enable spoken replies automatically when /voice on
  silence_threshold: 200        # RMS threshold for speech detection
  silence_duration: 3.0         # Seconds of silence before auto-stop
```

Use `/voice on` in the CLI to enable microphone mode, `record_key` to start/stop recording, and `/voice tts` to toggle spoken replies. See [Voice Mode](/docs/user-guide/features/voice-mode) for end-to-end setup and platform-specific behavior.

## Streaming

Stream tokens to the terminal or messaging platforms as they arrive, instead of waiting for the full response.

### CLI Streaming

```yaml
display:
  streaming: true         # Stream tokens to terminal in real-time
  show_reasoning: true    # Also stream reasoning/thinking tokens (optional)
```

When enabled, responses appear token-by-token inside a streaming box. Tool calls are still captured silently. If the provider doesn't support streaming, it falls back to the normal display automatically.

### Gateway Streaming (Telegram, Discord, Slack)

```yaml
streaming:
  enabled: true           # Enable progressive message editing
  transport: edit         # "edit" (progressive message editing) or "off"
  edit_interval: 0.3      # Seconds between message edits
  buffer_threshold: 40    # Characters before forcing an edit flush
  cursor: " ▉"            # Cursor shown during streaming
```

When enabled, the bot sends a message on the first token, then progressively edits it as more tokens arrive. Platforms that don't support message editing (Signal, Email, Home Assistant) are auto-detected on the first attempt — streaming is gracefully disabled for that session with no flood of messages.

For separate natural mid-turn assistant updates without progressive token editing, set `display.interim_assistant_messages: true`.

**Overflow handling:** If the streamed text exceeds the platform's message length limit (~4096 chars), the current message is finalized and a new one starts automatically.

:::note
Streaming is disabled by default. Enable it in `~/.hermes/config.yaml` to try the streaming UX.
:::

## Group Chat Session Isolation

Control whether shared chats keep one conversation per room or one conversation per participant:

```yaml
group_sessions_per_user: true  # true = per-user isolation in groups/channels, false = one shared session per chat
```

- `true` is the default and recommended setting. In Discord channels, Telegram groups, Slack channels, and similar shared contexts, each sender gets their own session when the platform provides a user ID.
- `false` reverts to the old shared-room behavior. That can be useful if you explicitly want Hermes to treat a channel like one collaborative conversation, but it also means users share context, token costs, and interrupt state.
- Direct messages are unaffected. Hermes still keys DMs by chat/DM ID as usual.
- Threads stay isolated from their parent channel either way; with `true`, each participant also gets their own session inside the thread.

For the behavior details and examples, see [Sessions](/docs/user-guide/sessions) and the [Discord guide](/docs/user-guide/messaging/discord).

## Unauthorized DM Behavior

Control what Hermes does when an unknown user sends a direct message:

```yaml
unauthorized_dm_behavior: pair

whatsapp:
  unauthorized_dm_behavior: ignore
```

- `pair` is the default. Hermes denies access, but replies with a one-time pairing code in DMs.
- `ignore` silently drops unauthorized DMs.
- Platform sections override the global default, so you can keep pairing enabled broadly while making one platform quieter.

## Quick Commands

Define custom commands that run shell commands without invoking the LLM — zero token usage, instant execution. Especially useful from messaging platforms (Telegram, Discord, etc.) for quick server checks or utility scripts.

```yaml
quick_commands:
  status:
    type: exec
    command: systemctl status hermes-agent
  disk:
    type: exec
    command: df -h /
  update:
    type: exec
    command: cd ~/.hermes/hermes-agent && git pull && pip install -e .
  gpu:
    type: exec
    command: nvidia-smi --query-gpu=name,utilization.gpu,memory.used,memory.total --format=csv,noheader
```

Usage: type `/status`, `/disk`, `/update`, or `/gpu` in the CLI or any messaging platform. The command runs locally on the host and returns the output directly — no LLM call, no tokens consumed.

- **30-second timeout** — long-running commands are killed with an error message
- **Priority** — quick commands are checked before skill commands, so you can override skill names
- **Autocomplete** — quick commands are resolved at dispatch time and are not shown in the built-in slash-command autocomplete tables
- **Type** — only `exec` is supported (runs a shell command); other types show an error
- **Works everywhere** — CLI, Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant

## Human Delay

Simulate human-like response pacing in messaging platforms:

```yaml
human_delay:
  mode: "off"                  # off | natural | custom
  min_ms: 800                  # Minimum delay (custom mode)
  max_ms: 2500                 # Maximum delay (custom mode)
```

## Code Execution

Configure the sandboxed Python code execution tool:

```yaml
code_execution:
  timeout: 300                 # Max execution time in seconds
  max_tool_calls: 50           # Max tool calls within code execution
```

## Web Search Backends

The `web_search`, `web_extract`, and `web_crawl` tools support four backend providers. Configure the backend in `config.yaml` or via `hermes tools`:

```yaml
web:
  backend: firecrawl    # firecrawl | parallel | tavily | exa
```

| Backend | Env Var | Search | Extract | Crawl |
|---------|---------|--------|---------|-------|
| **Firecrawl** (default) | `FIRECRAWL_API_KEY` | ✔ | ✔ | ✔ |
| **Parallel** | `PARALLEL_API_KEY` | ✔ | ✔ | — |
| **Tavily** | `TAVILY_API_KEY` | ✔ | ✔ | ✔ |
| **Exa** | `EXA_API_KEY` | ✔ | ✔ | — |

**Backend selection:** If `web.backend` is not set, the backend is auto-detected from available API keys. If only `EXA_API_KEY` is set, Exa is used. If only `TAVILY_API_KEY` is set, Tavily is used. If only `PARALLEL_API_KEY` is set, Parallel is used. Otherwise Firecrawl is the default.

**Self-hosted Firecrawl:** Set `FIRECRAWL_API_URL` to point at your own instance. When a custom URL is set, the API key becomes optional (set `USE_DB_AUTHENTICATION=false` on the server to disable auth).

**Parallel search modes:** Set `PARALLEL_SEARCH_MODE` to control search behavior — `fast`, `one-shot`, or `agentic` (default: `agentic`).

## Browser

Configure browser automation behavior:

```yaml
browser:
  inactivity_timeout: 120        # Seconds before auto-closing idle sessions
  command_timeout: 30             # Timeout in seconds for browser commands (screenshot, navigate, etc.)
  record_sessions: false         # Auto-record browser sessions as WebM videos to ~/.hermes/browser_recordings/
  camofox:
    managed_persistence: false   # When true, Camofox sessions persist cookies/logins across restarts
```

The browser toolset supports multiple providers. See the [Browser feature page](/docs/user-guide/features/browser) for details on Browserbase, Browser Use, and local Chrome CDP setup.

## Timezone

Override the server-local timezone with an IANA timezone string. Affects timestamps in logs, cron scheduling, and system prompt time injection.

```yaml
timezone: "America/New_York"   # IANA timezone (default: "" = server-local time)
```

Supported values: any IANA timezone identifier (e.g. `America/New_York`, `Europe/London`, `Asia/Kolkata`, `UTC`). Leave empty or omit for server-local time.

## Discord

Configure Discord-specific behavior for the messaging gateway:

```yaml
discord:
  require_mention: true          # Require @mention to respond in server channels
  free_response_channels: ""     # Comma-separated channel IDs where bot responds without @mention
  auto_thread: true              # Auto-create threads on @mention in channels
```

- `require_mention` — when `true` (default), the bot only responds in server channels when mentioned with `@BotName`. DMs always work without mention.
- `free_response_channels` — comma-separated list of channel IDs where the bot responds to every message without requiring a mention.
- `auto_thread` — when `true` (default), mentions in channels automatically create a thread for the conversation, keeping channels clean (similar to Slack threading).

## Security

Pre-execution security scanning and secret redaction:

```yaml
security:
  redact_secrets: true           # Redact API key patterns in tool output and logs
  tirith_enabled: true           # Enable Tirith security scanning for terminal commands
  tirith_path: "tirith"          # Path to tirith binary (default: "tirith" in $PATH)
  tirith_timeout: 5              # Seconds to wait for tirith scan before timing out
  tirith_fail_open: true         # Allow command execution if tirith is unavailable
  website_blocklist:             # See Website Blocklist section below
    enabled: false
    domains: []
    shared_files: []
```

- `redact_secrets` — automatically detects and redacts patterns that look like API keys, tokens, and passwords in tool output before it enters the conversation context and logs.
- `tirith_enabled` — when `true`, terminal commands are scanned by [Tirith](https://github.com/StackGuardian/tirith) before execution to detect potentially dangerous operations.
- `tirith_path` — path to the tirith binary. Set this if tirith is installed in a non-standard location.
- `tirith_timeout` — maximum seconds to wait for a tirith scan. Commands proceed if the scan times out.
- `tirith_fail_open` — when `true` (default), commands are allowed to execute if tirith is unavailable or fails. Set to `false` to block commands when tirith cannot verify them.

## Website Blocklist

Block specific domains from being accessed by the agent's web and browser tools:

```yaml
security:
  website_blocklist:
    enabled: false               # Enable URL blocking (default: false)
    domains:                     # List of blocked domain patterns
      - "*.internal.company.com"
      - "admin.example.com"
      - "*.local"
    shared_files:                # Load additional rules from external files
      - "/etc/hermes/blocked-sites.txt"
```

When enabled, any URL matching a blocked domain pattern is rejected before the web or browser tool executes. This applies to `web_search`, `web_extract`, `browser_navigate`, and any tool that accesses URLs.

Domain rules support:
- Exact domains: `admin.example.com`
- Wildcard subdomains: `*.internal.company.com` (blocks all subdomains)
- TLD wildcards: `*.local`

Shared files contain one domain rule per line (blank lines and `#` comments are ignored). Missing or unreadable files log a warning but don't disable other web tools.

The policy is cached for 30 seconds, so config changes take effect quickly without restart.

## Smart Approvals

Control how Hermes handles potentially dangerous commands:

```yaml
approvals:
  mode: manual   # manual | smart | off
```

| Mode | Behavior |
|------|----------|
| `manual` (default) | Prompt the user before executing any flagged command. In the CLI, shows an interactive approval dialog. In messaging, queues a pending approval request. |
| `smart` | Use an auxiliary LLM to assess whether a flagged command is actually dangerous. Low-risk commands are auto-approved with session-level persistence. Genuinely risky commands are escalated to the user. |
| `off` | Skip all approval checks. Equivalent to `HERMES_YOLO_MODE=true`. **Use with caution.** |

Smart mode is particularly useful for reducing approval fatigue — it lets the agent work more autonomously on safe operations while still catching genuinely destructive commands.

:::warning
Setting `approvals.mode: off` disables all safety checks for terminal commands. Only use this in trusted, sandboxed environments.
:::

## Checkpoints

Automatic filesystem snapshots before destructive file operations. See the [Checkpoints & Rollback](/docs/user-guide/checkpoints-and-rollback) for details.

```yaml
checkpoints:
  enabled: true                  # Enable automatic checkpoints (also: hermes --checkpoints)
  max_snapshots: 50              # Max checkpoints to keep per directory
```


## Delegation

Configure subagent behavior for the delegate tool:

```yaml
delegation:
  # model: "google/gemini-3-flash-preview"  # Override model (empty = inherit parent)
  # provider: "openrouter"                  # Override provider (empty = inherit parent)
  # base_url: "http://localhost:1234/v1"    # Direct OpenAI-compatible endpoint (takes precedence over provider)
  # api_key: "local-key"                    # API key for base_url (falls back to OPENAI_API_KEY)
```

**Subagent provider:model override:** By default, subagents inherit the parent agent's provider and model. Set `delegation.provider` and `delegation.model` to route subagents to a different provider:model pair — e.g., use a cheap/fast model for narrowly-scoped subtasks while your primary agent runs an expensive reasoning model.

**Direct endpoint override:** If you want the obvious custom-endpoint path, set `delegation.base_url`, `delegation.api_key`, and `delegation.model`. That sends subagents directly to that OpenAI-compatible endpoint and takes precedence over `delegation.provider`. If `delegation.api_key` is omitted, Hermes falls back to `OPENAI_API_KEY` only.

The delegation provider uses the same credential resolution as CLI/gateway startup. All configured providers are supported: `openrouter`, `nous`, `copilot`, `zai`, `kimi-coding`, `minimax`, `minimax-cn`. When a provider is set, the system automatically resolves the correct base URL, API key, and API mode — no manual credential wiring needed.

**Precedence:** `delegation.base_url` in config → `delegation.provider` in config → parent provider (inherited). `delegation.model` in config → parent model (inherited). Setting just `model` without `provider` changes only the model name while keeping the parent's credentials (useful for switching models within the same provider like OpenRouter).

## Clarify

Configure the clarification prompt behavior:

```yaml
clarify:
  timeout: 120                 # Seconds to wait for user clarification response
```

## Context Files (SOUL.md, AGENTS.md)

Hermes uses two different context scopes:

| File | Purpose | Scope |
|------|---------|-------|
| `SOUL.md` | **Primary agent identity** — defines who the agent is (slot #1 in the system prompt) | `~/.hermes/SOUL.md` or `$HERMES_HOME/SOUL.md` |
| `.hermes.md` / `HERMES.md` | Project-specific instructions (highest priority) | Walks to git root |
| `AGENTS.md` | Project-specific instructions, coding conventions | Recursive directory walk |
| `CLAUDE.md` | Claude Code context files (also detected) | Working directory only |
| `.cursorrules` | Cursor IDE rules (also detected) | Working directory only |
| `.cursor/rules/*.mdc` | Cursor rule files (also detected) | Working directory only |

- **SOUL.md** is the agent's primary identity. It occupies slot #1 in the system prompt, completely replacing the built-in default identity. Edit it to fully customize who the agent is.
- If SOUL.md is missing, empty, or cannot be loaded, Hermes falls back to a built-in default identity.
- **Project context files use a priority system** — only ONE type is loaded (first match wins): `.hermes.md` → `AGENTS.md` → `CLAUDE.md` → `.cursorrules`. SOUL.md is always loaded independently.
- **AGENTS.md** is hierarchical: if subdirectories also have AGENTS.md, all are combined.
- Hermes automatically seeds a default `SOUL.md` if one does not already exist.
- All loaded context files are capped at 20,000 characters with smart truncation.

See also:
- [Personality & SOUL.md](/docs/user-guide/features/personality)
- [Context Files](/docs/user-guide/features/context-files)

## Working Directory

| Context | Default |
|---------|---------|
| **CLI (`hermes`)** | Current directory where you run the command |
| **Messaging gateway** | Home directory `~` (override with `MESSAGING_CWD`) |
| **Docker / Singularity / Modal / SSH** | User's home directory inside the container or remote machine |

Override the working directory:
```bash
# In ~/.hermes/.env or ~/.hermes/config.yaml:
MESSAGING_CWD=/home/myuser/projects    # Gateway sessions
TERMINAL_CWD=/workspace                # All terminal sessions
```