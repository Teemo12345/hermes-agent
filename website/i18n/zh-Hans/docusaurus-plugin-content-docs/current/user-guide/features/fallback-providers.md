---
title: 备用提供商
---

# 备用提供商

Hermes Agent 有三层弹性机制，可以在提供商遇到问题时保持您的会话运行：

1. **[凭证池](./credential-pools.md)** — 在同一提供商的多个 API 密钥之间轮换（首先尝试）
2. **主模型备用** — 当主模型失败时自动切换到不同的提供商:模型
3. **辅助任务备用** — 针对视觉、压缩和网页提取等辅助任务的独立提供商解析

凭证池处理同一提供商的轮换（例如，多个 OpenRouter 密钥）。本页介绍跨提供商备用。两者都是可选的，并且独立工作。

## 主模型备用

当您的主要 LLM 提供商遇到错误 — 速率限制、服务器过载、认证失败、连接断开 — Hermes 可以在会话中期自动切换到备用提供商:模型对，而不会丢失您的对话。

### 配置

在 `~/.hermes/config.yaml` 中添加 `fallback_model` 部分：

```yaml
fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4
```

`provider` 和 `model` 都是**必需的**。如果缺少任何一个，备用功能将被禁用。

### 支持的提供商

| 提供商 | 值 | 要求 |
|----------|-------|-------------|
| AI Gateway | `ai-gateway` | `AI_GATEWAY_API_KEY` |
| OpenRouter | `openrouter` | `OPENROUTER_API_KEY` |
| Nous Portal | `nous` | `hermes auth` (OAuth) |
| OpenAI Codex | `openai-codex` | `hermes model` (ChatGPT OAuth) |
| GitHub Copilot | `copilot` | `COPILOT_GITHUB_TOKEN`、`GH_TOKEN` 或 `GITHUB_TOKEN` |
| GitHub Copilot ACP | `copilot-acp` | 外部进程（编辑器集成） |
| Anthropic | `anthropic` | `ANTHROPIC_API_KEY` 或 Claude Code 凭证 |
| z.ai / GLM | `zai` | `GLM_API_KEY` |
| Kimi / Moonshot | `kimi-coding` | `KIMI_API_KEY` |
| MiniMax | `minimax` | `MINIMAX_API_KEY` |
| MiniMax (中国) | `minimax-cn` | `MINIMAX_CN_API_KEY` |
| DeepSeek | `deepseek` | `DEEPSEEK_API_KEY` |
| OpenCode Zen | `opencode-zen` | `OPENCODE_ZEN_API_KEY` |
| OpenCode Go | `opencode-go` | `OPENCODE_GO_API_KEY` |
| Kilo Code | `kilocode` | `KILOCODE_API_KEY` |
| Xiaomi MiMo | `xiaomi` | `XIAOMI_API_KEY` |
| Arcee AI | `arcee` | `ARCEEAI_API_KEY` |
| Alibaba / DashScope | `alibaba` | `DASHSCOPE_API_KEY` |
| Hugging Face | `huggingface` | `HF_TOKEN` |
| 自定义端点 | `custom` | `base_url` + `api_key_env`（见下文） |

### 自定义端点备用

对于自定义 OpenAI 兼容端点，添加 `base_url` 和可选的 `api_key_env`：

```yaml
fallback_model:
  provider: custom
  model: my-local-model
  base_url: http://localhost:8000/v1
  api_key_env: MY_LOCAL_KEY          # 包含 API 密钥的环境变量名称
```

### 备用触发条件

当主模型因以下原因失败时，备用会自动激活：

- **速率限制**（HTTP 429）— 用尽重试尝试后
- **服务器错误**（HTTP 500、502、503）— 用尽重试尝试后
- **认证失败**（HTTP 401、403）— 立即（重试没有意义）
- **未找到**（HTTP 404）— 立即
- **无效响应** — 当 API 反复返回格式错误或空响应时

当触发时，Hermes：

1. 解析备用提供商的凭证
2. 构建新的 API 客户端
3. 原地交换模型、提供商和客户端
4. 重置重试计数器并继续对话

切换是无缝的 — 您的对话历史、工具调用和上下文都被保留。智能体从它离开的地方继续，只是使用不同的模型。

:::info 一次性
每个会话备用**最多激活一次**。如果备用提供商也失败，正常的错误处理会接管（重试，然后错误消息）。这可以防止级联故障转移循环。
:::

### 示例

**将 OpenRouter 作为 Anthropic 原生的备用：**
```yaml
model:
  provider: anthropic
  default: claude-sonnet-4-6

fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4
```

**将 Nous Portal 作为 OpenRouter 的备用：**
```yaml
model:
  provider: openrouter
  default: anthropic/claude-opus-4

fallback_model:
  provider: nous
  model: nous-hermes-3
```

**将本地模型作为云端的备用：**
```yaml
fallback_model:
  provider: custom
  model: llama-3.1-70b
  base_url: http://localhost:8000/v1
  api_key_env: LOCAL_API_KEY
```

**将 Codex OAuth 作为备用：**
```yaml
fallback_model:
  provider: openai-codex
  model: gpt-5.3-codex
```

### 备用在哪里工作

| 上下文 | 支持备用 |
|---------|-------------------|
| CLI 会话 | ✔ |
| 消息网关（Telegram、Discord 等） | ✔ |
| 子智能体委托 | ✘（子智能体不继承备用配置） |
| Cron 作业 | ✘（使用固定提供商运行） |
| 辅助任务（视觉、压缩） | ✘（使用自己的提供商链 — 见下文） |

:::tip
`fallback_model` 没有环境变量 — 它仅通过 `config.yaml` 配置。这是有意的：备用配置是一个深思熟虑的选择，而不是过时的 shell 导出应该覆盖的东西。
:::

---

## 辅助任务备用

Hermes 为辅助任务使用单独的轻量级模型。每个任务都有自己的提供商解析链，作为内置的备用系统。

### 具有独立提供商解析的任务

| 任务 | 功能 | 配置键 |
|------|-------------|-----------|
| Vision | 图像分析、浏览器屏幕截图 | `auxiliary.vision` |
| Web Extract | 网页摘要 | `auxiliary.web_extract` |
| Compression | 上下文压缩摘要 | `auxiliary.compression` |
| Session Search | 过去会话摘要 | `auxiliary.session_search` |
| Skills Hub | 技能搜索和发现 | `auxiliary.skills_hub` |
| MCP | MCP 辅助操作 | `auxiliary.mcp` |
| Memory Flush | 记忆整合 | `auxiliary.flush_memories` |

### 自动检测链

当任务的提供商设置为 `"auto"`（默认）时，Hermes 按顺序尝试提供商，直到找到一个可用的：

**对于文本任务（压缩、网页提取等）：**

```text
OpenRouter → Nous Portal → 自定义端点 → Codex OAuth →
API 密钥提供商（z.ai、Kimi、MiniMax、Xiaomi MiMo、Hugging Face、Anthropic）→ 放弃
```

**对于视觉任务：**

```text
主提供商（如果支持视觉）→ OpenRouter → Nous Portal →
Codex OAuth → Anthropic → 自定义端点 → 放弃
```

如果解析的提供商在调用时失败，Hermes 还有内部重试：如果提供商不是 OpenRouter 且没有设置显式 `base_url`，它会尝试 OpenRouter 作为最后的备用方案。

### 配置辅助提供商

每个任务可以在 `config.yaml` 中独立配置：

```yaml
auxiliary:
  vision:
    provider: "auto"              # auto | openrouter | nous | codex | main | anthropic
    model: ""                     # 例如 "openai/gpt-4o"
    base_url: ""                  # 直接端点（优先于提供商）
    api_key: ""                   # base_url 的 API 密钥

  web_extract:
    provider: "auto"
    model: ""

  compression:
    provider: "auto"
    model: ""

  session_search:
    provider: "auto"
    model: ""

  skills_hub:
    provider: "auto"
    model: ""

  mcp:
    provider: "auto"
    model: ""

  flush_memories:
    provider: "auto"
    model: ""
```

Every task above follows the same **provider / model / base_url** pattern. Context compression is configured under `auxiliary.compression`:

```yaml
auxiliary:
  compression:
    provider: main                                    # Same provider options as other auxiliary tasks
    model: google/gemini-3-flash-preview
    base_url: null                                    # Custom OpenAI-compatible endpoint
```

And the fallback model uses:

```yaml
fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4
  # base_url: http://localhost:8000/v1               # Optional custom endpoint
```

All three — auxiliary, compression, fallback — work the same way: set `provider` to pick who handles the request, `model` to pick which model, and `base_url` to point at a custom endpoint (overrides provider).

### Provider Options for Auxiliary Tasks

These options apply to `auxiliary:`, `compression:`, and `fallback_model:` configs only — `"main"` is **not** a valid value for your top-level `model.provider`. For custom endpoints, use `provider: custom` in your `model:` section (see [AI Providers](/docs/integrations/providers)).

| Provider | Description | Requirements |
|----------|-------------|-------------|
| `"auto"` | Try providers in order until one works (default) | At least one provider configured |
| `"openrouter"` | Force OpenRouter | `OPENROUTER_API_KEY` |
| `"nous"` | Force Nous Portal | `hermes auth` |
| `"codex"` | Force Codex OAuth | `hermes model` → Codex |
| `"main"` | Use whatever provider the main agent uses (auxiliary tasks only) | Active main provider configured |
| `"anthropic"` | Force Anthropic native | `ANTHROPIC_API_KEY` or Claude Code credentials |

### Direct Endpoint Override

For any auxiliary task, setting `base_url` bypasses provider resolution entirely and sends requests directly to that endpoint:

```yaml
auxiliary:
  vision:
    base_url: "http://localhost:1234/v1"
    api_key: "local-key"
    model: "qwen2.5-vl"
```

`base_url` takes precedence over `provider`. Hermes uses the configured `api_key` for authentication, falling back to `OPENAI_API_KEY` if not set. It does **not** reuse `OPENROUTER_API_KEY` for custom endpoints.

---

## Context Compression Fallback

Context compression uses the `auxiliary.compression` config block to control which model and provider handles summarization:

```yaml
auxiliary:
  compression:
    provider: "auto"                              # auto | openrouter | nous | main
    model: "google/gemini-3-flash-preview"
```

:::info Legacy migration
Older configs with `compression.summary_model` / `compression.summary_provider` / `compression.summary_base_url` are automatically migrated to `auxiliary.compression.*` on first load (config version 17).
:::

If no provider is available for compression, Hermes drops middle conversation turns without generating a summary rather than failing the session.

---

## Delegation Provider Override

Subagents spawned by `delegate_task` do **not** use the primary fallback model. However, they can be routed to a different provider:model pair for cost optimization:

```yaml
delegation:
  provider: "openrouter"                      # override provider for all subagents
  model: "google/gemini-3-flash-preview"      # override model
  # base_url: "http://localhost:1234/v1"      # or use a direct endpoint
  # api_key: "local-key"
```

See [Subagent Delegation](/docs/user-guide/features/delegation) for full configuration details.

---

## Cron Job Providers

Cron jobs run with whatever provider is configured at execution time. They do not support a fallback model. To use a different provider for cron jobs, configure `provider` and `model` overrides on the cron job itself:

```python
cronjob(
    action="create",
    schedule="every 2h",
    prompt="Check server status",
    provider="openrouter",
    model="google/gemini-3-flash-preview"
)
```

See [Scheduled Tasks (Cron)](/docs/user-guide/features/cron) for full configuration details.

---

## Summary

| Feature | Fallback Mechanism | Config Location |
|---------|-------------------|----------------|
| Main agent model | `fallback_model` in config.yaml — one-shot failover on errors | `fallback_model:` (top-level) |
| Vision | Auto-detection chain + internal OpenRouter retry | `auxiliary.vision` |
| Web extraction | Auto-detection chain + internal OpenRouter retry | `auxiliary.web_extract` |
| Context compression | Auto-detection chain, degrades to no-summary if unavailable | `auxiliary.compression` |
| Session search | Auto-detection chain | `auxiliary.session_search` |
| Skills hub | Auto-detection chain | `auxiliary.skills_hub` |
| MCP helpers | Auto-detection chain | `auxiliary.mcp` |
| Memory flush | Auto-detection chain | `auxiliary.flush_memories` |
| Delegation | Provider override only (no automatic fallback) | `delegation.provider` / `delegation.model` |
| Cron jobs | Per-job provider override only (no automatic fallback) | Per-job `provider` / `model` |
