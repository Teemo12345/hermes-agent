---
sidebar_position: 11
sidebar_label: "插件"
title: "插件"
description: "通过插件系统使用自定义工具、钩子和集成扩展 Hermes"
---

# 插件

Hermes 有一个插件系统，用于在不修改核心代码的情况下添加自定义工具、钩子和集成。

**→ [构建 Hermes 插件](/docs/guides/build-a-hermes-plugin)** — 包含完整工作示例的分步指南。

## 快速概览

将包含 `plugin.yaml` 和 Python 代码的目录放入 `~/.hermes/plugins/`：

```
~/.hermes/plugins/my-plugin/
├── plugin.yaml      # 清单
├── __init__.py      # register() — 将模式连接到处理程序
├── schemas.py       # 工具模式（LLM 看到的内容）
└── tools.py         # 工具处理程序（调用时运行的内容）
```

启动 Hermes — 您的工具会与内置工具一起出现。模型可以立即调用它们。

### 最小工作示例

这是一个完整的插件，添加了 `hello_world` 工具并通过钩子记录每次工具调用。

**`~/.hermes/plugins/hello-world/plugin.yaml`**

```yaml
name: hello-world
version: "1.0"
description: A minimal example plugin
```

**`~/.hermes/plugins/hello-world/__init__.py`**

```python
"""Minimal Hermes plugin — registers a tool and a hook."""


def register(ctx):
    # --- Tool: hello_world ---
    schema = {
        "name": "hello_world",
        "description": "Returns a friendly greeting for the given name.",
        "parameters": {
            "type": "object",
            "properties": {
                "name": {
                    "type": "string",
                    "description": "Name to greet",
                }
            },
            "required": ["name"],
        },
    }

    def handle_hello(params):
        name = params.get("name", "World")
        return f"Hello, {name}! 👋  (from the hello-world plugin)"

    ctx.register_tool("hello_world", schema, handle_hello)

    # --- Hook: log every tool call ---
    def on_tool_call(tool_name, params, result):
        print(f"[hello-world] tool called: {tool_name}")

    ctx.register_hook("post_tool_call", on_tool_call)
```

将两个文件放入 `~/.hermes/plugins/hello-world/`，重启 Hermes，模型可以立即调用 `hello_world`。钩子在每次工具调用后打印日志行。

项目本地插件在 `./.hermes/plugins/` 下默认禁用。通过在启动 Hermes 前设置 `HERMES_ENABLE_PROJECT_PLUGINS=true`，只为受信任的存储库启用它们。

## 插件可以做什么

| 功能 | 方法 |
|-----------|-----|
| 添加工具 | `ctx.register_tool(name, schema, handler)` |
| 添加钩子 | `ctx.register_hook("post_tool_call", callback)` |
| 添加 CLI 命令 | `ctx.register_cli_command(name, help, setup_fn, handler_fn)` — 添加 `hermes <plugin> <subcommand>` |
| 注入消息 | `ctx.inject_message(content, role="user")` — 参见 [注入消息](#injecting-messages) |
| 附带数据文件 | `Path(__file__).parent / "data" / "file.yaml"` |
| 捆绑技能 | `ctx.register_skill(name, path)` — 命名空间为 `plugin:skill`，通过 `skill_view("plugin:skill")` 加载 |
| 基于环境变量的门控 | plugin.yaml 中的 `requires_env: [API_KEY]` — 在 `hermes plugins install` 期间提示 |
| 通过 pip 分发 | `[project.entry-points."hermes_agent.plugins"]` |

## 插件发现

| 来源 | 路径 | 使用场景 |
|--------|------|----------|
| 用户 | `~/.hermes/plugins/` | 个人插件 |
| 项目 | `.hermes/plugins/` | 项目特定插件（需要 `HERMES_ENABLE_PROJECT_PLUGINS=true`） |
| pip | `hermes_agent.plugins` entry_points | 分布式包 |

## Available hooks

Plugins can register callbacks for these lifecycle events. See the **[Event Hooks page](/docs/user-guide/features/hooks#plugin-hooks)** for full details, callback signatures, and examples.

| Hook | Fires when |
|------|-----------|
| [`pre_tool_call`](/docs/user-guide/features/hooks#pre_tool_call) | Before any tool executes |
| [`post_tool_call`](/docs/user-guide/features/hooks#post_tool_call) | After any tool returns |
| [`pre_llm_call`](/docs/user-guide/features/hooks#pre_llm_call) | Once per turn, before the LLM loop — can return `{"context": "..."}` to [inject context into the user message](/docs/user-guide/features/hooks#pre_llm_call) |
| [`post_llm_call`](/docs/user-guide/features/hooks#post_llm_call) | Once per turn, after the LLM loop (successful turns only) |
| [`on_session_start`](/docs/user-guide/features/hooks#on_session_start) | New session created (first turn only) |
| [`on_session_end`](/docs/user-guide/features/hooks#on_session_end) | End of every `run_conversation` call + CLI exit handler |

## Plugin types

Hermes has three kinds of plugins:

| Type | What it does | Selection | Location |
|------|-------------|-----------|----------|
| **General plugins** | Add tools, hooks, CLI commands | Multi-select (enable/disable) | `~/.hermes/plugins/` |
| **Memory providers** | Replace or augment built-in memory | Single-select (one active) | `plugins/memory/` |
| **Context engines** | Replace the built-in context compressor | Single-select (one active) | `plugins/context_engine/` |

Memory providers and context engines are **provider plugins** — only one of each type can be active at a time. General plugins can be enabled in any combination.

## Managing plugins

```bash
hermes plugins                  # unified interactive UI
hermes plugins list             # table view with enabled/disabled status
hermes plugins install user/repo  # install from Git
hermes plugins update my-plugin   # pull latest
hermes plugins remove my-plugin   # uninstall
hermes plugins enable my-plugin   # re-enable a disabled plugin
hermes plugins disable my-plugin  # disable without removing
```

### Interactive UI

Running `hermes plugins` with no arguments opens a composite interactive screen:

```
Plugins
  ↑↓ navigate  SPACE toggle  ENTER configure/confirm  ESC done

  General Plugins
 → [✓] my-tool-plugin — Custom search tool
   [ ] webhook-notifier — Event hooks

  Provider Plugins
     Memory Provider          ▸ honcho
     Context Engine           ▸ compressor
```

- **General Plugins section** — checkboxes, toggle with SPACE
- **Provider Plugins section** — shows current selection. Press ENTER to drill into a radio picker where you choose one active provider.

Provider plugin selections are saved to `config.yaml`:

```yaml
memory:
  provider: "honcho"      # empty string = built-in only

context:
  engine: "compressor"    # default built-in compressor
```

### Disabling general plugins

Disabled plugins remain installed but are skipped during loading. The disabled list is stored in `config.yaml` under `plugins.disabled`:

```yaml
plugins:
  disabled:
    - my-noisy-plugin
```

In a running session, `/plugins` shows which plugins are currently loaded.

## Injecting Messages

Plugins can inject messages into the active conversation using `ctx.inject_message()`:

```python
ctx.inject_message("New data arrived from the webhook", role="user")
```

**Signature:** `ctx.inject_message(content: str, role: str = "user") -> bool`

How it works:

- If the agent is **idle** (waiting for user input), the message is queued as the next input and starts a new turn.
- If the agent is **mid-turn** (actively running), the message interrupts the current operation — the same as a user typing a new message and pressing Enter.
- For non-`"user"` roles, the content is prefixed with `[role]` (e.g. `[system] ...`).
- Returns `True` if the message was queued successfully, `False` if no CLI reference is available (e.g. in gateway mode).

This enables plugins like remote control viewers, messaging bridges, or webhook receivers to feed messages into the conversation from external sources.

:::note
`inject_message` is only available in CLI mode. In gateway mode, there is no CLI reference and the method returns `False`.
:::

See the **[full guide](/docs/guides/build-a-hermes-plugin)** for handler contracts, schema format, hook behavior, error handling, and common mistakes.
