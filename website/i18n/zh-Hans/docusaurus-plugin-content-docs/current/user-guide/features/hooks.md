---
sidebar_position: 6
title: "事件钩子"
description: "在关键生命周期点运行自定义代码 — 记录活动、发送警报、发布到 webhook"
---

# 事件钩子

Hermes 有两个钩子系统，在关键生命周期点运行自定义代码：

| 系统 | 注册方式 | 运行环境 | 使用场景 |
|--------|---------------|---------|----------|
| **[网关钩子](#gateway-event-hooks)** | `~/.hermes/hooks/` 中的 `HOOK.yaml` + `handler.py` | 仅网关 | 日志记录、警报、webhook |
| **[插件钩子](#plugin-hooks)** | [插件](/docs/user-guide/features/plugins) 中的 `ctx.register_hook()` | CLI + 网关 | 工具拦截、指标、护栏 |

两个系统都是非阻塞的 — 任何钩子中的错误都会被捕获并记录，绝不会导致智能体崩溃。

## 网关事件钩子

网关钩子在网关运行期间（Telegram、Discord、Slack、WhatsApp）自动触发，不会阻塞主智能体管道。

### 创建钩子

每个钩子是 `~/.hermes/hooks/` 下的一个目录，包含两个文件：

```text
~/.hermes/hooks/
└── my-hook/
    ├── HOOK.yaml      # 声明要监听的事件
    └── handler.py     # Python 处理函数
```

#### HOOK.yaml

```yaml
name: my-hook
description: Log all agent activity to a file
events:
  - agent:start
  - agent:end
  - agent:step
```

`events` 列表决定哪些事件会触发您的处理程序。您可以订阅任何事件组合，包括通配符如 `command:*`。

#### handler.py

```python
import json
from datetime import datetime
from pathlib import Path

LOG_FILE = Path.home() / ".hermes" / "hooks" / "my-hook" / "activity.log"

async def handle(event_type: str, context: dict):
    """Called for each subscribed event. Must be named 'handle'."""
    entry = {
        "timestamp": datetime.now().isoformat(),
        "event": event_type,
        **context,
    }
    with open(LOG_FILE, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

**处理程序规则：**
- 必须命名为 `handle`
- 接收 `event_type`（字符串）和 `context`（字典）
- 可以是 `async def` 或常规 `def` — 两者都有效
- 错误会被捕获并记录，绝不会导致智能体崩溃

### 可用事件

| 事件 | 触发时机 | 上下文键 |
|-------|---------------|--------------|
| `gateway:startup` | 网关进程启动 | `platforms`（活跃平台名称列表） |
| `session:start` | 新消息会话创建 | `platform`、`user_id`、`session_id`、`session_key` |
| `session:end` | 会话结束（重置前） | `platform`、`user_id`、`session_key` |
| `session:reset` | 用户运行 `/new` 或 `/reset` | `platform`、`user_id`、`session_key` |
| `agent:start` | 智能体开始处理消息 | `platform`、`user_id`、`session_id`、`message` |
| `agent:step` | 工具调用循环的每次迭代 | `platform`、`user_id`、`session_id`、`iteration`、`tool_names` |
| `agent:end` | 智能体完成处理 | `platform`、`user_id`、`session_id`、`message`、`response` |
| `command:*` | 执行任何斜杠命令 | `platform`、`user_id`、`command`、`args` |

#### 通配符匹配

为 `command:*` 注册的处理程序会为任何 `command:` 事件（`command:model`、`command:reset` 等）触发。使用单个订阅监控所有斜杠命令。

### 示例

#### 启动清单（BOOT.md）— 内置

网关附带一个内置的 `boot-md` 钩子，在每次启动时查找 `~/.hermes/BOOT.md`。如果文件存在，智能体会在后台会话中运行其指令。无需安装 — 只需创建文件。

**创建 `~/.hermes/BOOT.md`：**

```markdown
# 启动清单

1. 检查是否有 cron 作业在夜间失败 — 运行 `hermes cron list`
2. 向 Discord #general 发送消息说 "Gateway restarted, all systems go"
3. 检查 /opt/app/deploy.log 在过去 24 小时内是否有任何错误
```

智能体在后台线程中运行这些指令，因此不会阻塞网关启动。如果没有需要注意的事项，智能体回复 `` 且不传递任何消息。

:::tip
没有 BOOT.md？钩子会静默跳过 — 零开销。在需要启动自动化时创建文件，不需要时删除它。
:::

#### 长任务 Telegram 警报

当智能体超过 10 步时向自己发送消息：

```yaml
# ~/.hermes/hooks/long-task-alert/HOOK.yaml
name: long-task-alert
description: Alert when agent is taking many steps
events:
  - agent:step
```

```python
# ~/.hermes/hooks/long-task-alert/handler.py
import os
import httpx

THRESHOLD = 10
BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
CHAT_ID = os.getenv("TELEGRAM_HOME_CHANNEL")

async def handle(event_type: str, context: dict):
    iteration = context.get("iteration", 0)
    if iteration == THRESHOLD and BOT_TOKEN and CHAT_ID:
        tools = ", ".join(context.get("tool_names", []))
        text = f"⚠️ Agent has been running for {iteration} steps. Last tools: {tools}"
        async with httpx.AsyncClient() as client:
            await client.post(
                f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
                json={"chat_id": CHAT_ID, "text": text},
            )
```

#### 命令使用日志记录器

跟踪使用了哪些斜杠命令：

```yaml
# ~/.hermes/hooks/command-logger/HOOK.yaml
name: command-logger
description: Log slash command usage
events:
  - command:*
```

```python
# ~/.hermes/hooks/command-logger/handler.py
import json
from datetime import datetime
from pathlib import Path

LOG = Path.home() / ".hermes" / "logs" / "command_usage.jsonl"

def handle(event_type: str, context: dict):
    LOG.parent.mkdir(parents=True, exist_ok=True)
    entry = {
        "ts": datetime.now().isoformat(),
        "command": context.get("command"),
        "args": context.get("args"),
        "platform": context.get("platform"),
        "user": context.get("user_id"),
    }
    with open(LOG, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

#### 会话开始 Webhook

在新会话时 POST 到外部服务：

```yaml
# ~/.hermes/hooks/session-webhook/HOOK.yaml
name: session-webhook
description: Notify external service on new sessions
events:
  - session:start
  - session:reset
```

```python
# ~/.hermes/hooks/session-webhook/handler.py
import httpx

WEBHOOK_URL = "https://your-service.example.com/hermes-events"

async def handle(event_type: str, context: dict):
    async with httpx.AsyncClient() as client:
        await client.post(WEBHOOK_URL, json={
            "event": event_type,
            **context,
        }, timeout=5)
```

### How It Works

1. On gateway startup, `HookRegistry.discover_and_load()` scans `~/.hermes/hooks/`
2. Each subdirectory with `HOOK.yaml` + `handler.py` is loaded dynamically
3. Handlers are registered for their declared events
4. At each lifecycle point, `hooks.emit()` fires all matching handlers
5. Errors in any handler are caught and logged — a broken hook never crashes the agent

:::info
Gateway hooks only fire in the **gateway** (Telegram, Discord, Slack, WhatsApp). The CLI does not load gateway hooks. For hooks that work everywhere, use [plugin hooks](#plugin-hooks).
:::

## Plugin Hooks

[Plugins](/docs/user-guide/features/plugins) can register hooks that fire in **both CLI and gateway** sessions. These are registered programmatically via `ctx.register_hook()` in your plugin's `register()` function.

```python
def register(ctx):
    ctx.register_hook("pre_tool_call", my_tool_observer)
    ctx.register_hook("post_tool_call", my_tool_logger)
    ctx.register_hook("pre_llm_call", my_memory_callback)
    ctx.register_hook("post_llm_call", my_sync_callback)
    ctx.register_hook("on_session_start", my_init_callback)
    ctx.register_hook("on_session_end", my_cleanup_callback)
```

**General rules for all hooks:**

- Callbacks receive **keyword arguments**. Always accept `**kwargs` for forward compatibility — new parameters may be added in future versions without breaking your plugin.
- If a callback **crashes**, it's logged and skipped. Other hooks and the agent continue normally. A misbehaving plugin can never break the agent.
- All hooks are **fire-and-forget observers** whose return values are ignored — except `pre_llm_call`, which can [inject context](#pre_llm_call).

### Quick reference

| Hook | Fires when | Returns |
|------|-----------|---------|
| [`pre_tool_call`](#pre_tool_call) | Before any tool executes | ignored |
| [`post_tool_call`](#post_tool_call) | After any tool returns | ignored |
| [`pre_llm_call`](#pre_llm_call) | Once per turn, before the tool-calling loop | context injection |
| [`post_llm_call`](#post_llm_call) | Once per turn, after the tool-calling loop | ignored |
| [`on_session_start`](#on_session_start) | New session created (first turn only) | ignored |
| [`on_session_end`](#on_session_end) | Session ends | ignored |

---

### `pre_tool_call`

Fires **immediately before** every tool execution — built-in tools and plugin tools alike.

**Callback signature:**

```python
def my_callback(tool_name: str, args: dict, task_id: str, **kwargs):
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `tool_name` | `str` | Name of the tool about to execute (e.g. `"terminal"`, `"web_search"`, `"read_file"`) |
| `args` | `dict` | The arguments the model passed to the tool |
| `task_id` | `str` | Session/task identifier. Empty string if not set. |

**Fires:** In `model_tools.py`, inside `handle_function_call()`, before the tool's handler runs. Fires once per tool call — if the model calls 3 tools in parallel, this fires 3 times.

**Return value:** Ignored.

**Use cases:** Logging, audit trails, tool call counters, blocking dangerous operations (print a warning), rate limiting.

**Example — tool call audit log:**

```python
import json, logging
from datetime import datetime

logger = logging.getLogger(__name__)

def audit_tool_call(tool_name, args, task_id, **kwargs):
    logger.info("TOOL_CALL session=%s tool=%s args=%s",
                task_id, tool_name, json.dumps(args)[:200])

def register(ctx):
    ctx.register_hook("pre_tool_call", audit_tool_call)
```

**Example — warn on dangerous tools:**

```python
DANGEROUS = {"terminal", "write_file", "patch"}

def warn_dangerous(tool_name, **kwargs):
    if tool_name in DANGEROUS:
        print(f"⚠ Executing potentially dangerous tool: {tool_name}")

def register(ctx):
    ctx.register_hook("pre_tool_call", warn_dangerous)
```

---

### `post_tool_call`

Fires **immediately after** every tool execution returns.

**Callback signature:**

```python
def my_callback(tool_name: str, args: dict, result: str, task_id: str, **kwargs):
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `tool_name` | `str` | Name of the tool that just executed |
| `args` | `dict` | The arguments the model passed to the tool |
| `result` | `str` | The tool's return value (always a JSON string) |
| `task_id` | `str` | Session/task identifier. Empty string if not set. |

**Fires:** In `model_tools.py`, inside `handle_function_call()`, after the tool's handler returns. Fires once per tool call. Does **not** fire if the tool raised an unhandled exception (the error is caught and returned as an error JSON string instead, and `post_tool_call` fires with that error string as `result`).

**Return value:** Ignored.

**Use cases:** Logging tool results, metrics collection, tracking tool success/failure rates, sending notifications when specific tools complete.

**Example — track tool usage metrics:**

```python
from collections import Counter
import json

_tool_counts = Counter()
_error_counts = Counter()

def track_metrics(tool_name, result, **kwargs):
    _tool_counts[tool_name] += 1
    try:
        parsed = json.loads(result)
        if "error" in parsed:
            _error_counts[tool_name] += 1
    except (json.JSONDecodeError, TypeError):
        pass

def register(ctx):
    ctx.register_hook("post_tool_call", track_metrics)
```

---

### `pre_llm_call`

Fires **once per turn**, before the tool-calling loop begins. This is the **only hook whose return value is used** — it can inject context into the current turn's user message.

**Callback signature:**

```python
def my_callback(session_id: str, user_message: str, conversation_history: list,
                is_first_turn: bool, model: str, platform: str, **kwargs):
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `session_id` | `str` | Unique identifier for the current session |
| `user_message` | `str` | The user's original message for this turn (before any skill injection) |
| `conversation_history` | `list` | Copy of the full message list (OpenAI format: `[{"role": "user", "content": "..."}]`) |
| `is_first_turn` | `bool` | `True` if this is the first turn of a new session, `False` on subsequent turns |
| `model` | `str` | The model identifier (e.g. `"anthropic/claude-sonnet-4.6"`) |
| `platform` | `str` | Where the session is running: `"cli"`, `"telegram"`, `"discord"`, etc. |

**Fires:** In `run_agent.py`, inside `run_conversation()`, after context compression but before the main `while` loop. Fires once per `run_conversation()` call (i.e. once per user turn), not once per API call within the tool loop.

**Return value:** If the callback returns a dict with a `"context"` key, or a plain non-empty string, the text is appended to the current turn's user message. Return `None` for no injection.

```python
# Inject context
return {"context": "Recalled memories:\n- User likes Python\n- Working on hermes-agent"}

# Plain string (equivalent)
return "Recalled memories:\n- User likes Python"

# No injection
return None
```

**Where context is injected:** Always the **user message**, never the system prompt. This preserves the prompt cache — the system prompt stays identical across turns, so cached tokens are reused. The system prompt is Hermes's territory (model guidance, tool enforcement, personality, skills). Plugins contribute context alongside the user's input.

All injected context is **ephemeral** — added at API call time only. The original user message in the conversation history is never mutated, and nothing is persisted to the session database.

When **multiple plugins** return context, their outputs are joined with double newlines in plugin discovery order (alphabetical by directory name).

**Use cases:** Memory recall, RAG context injection, guardrails, per-turn analytics.

**Example — memory recall:**

```python
import httpx

MEMORY_API = "https://your-memory-api.example.com"

def recall(session_id, user_message, is_first_turn, **kwargs):
    try:
        resp = httpx.post(f"{MEMORY_API}/recall", json={
            "session_id": session_id,
            "query": user_message,
        }, timeout=3)
        memories = resp.json().get("results", [])
        if not memories:
            return None
        text = "Recalled context:\n" + "\n".join(f"- {m['text']}" for m in memories)
        return {"context": text}
    except Exception:
        return None

def register(ctx):
    ctx.register_hook("pre_llm_call", recall)
```

**Example — guardrails:**

```python
POLICY = "Never execute commands that delete files without explicit user confirmation."

def guardrails(**kwargs):
    return {"context": POLICY}

def register(ctx):
    ctx.register_hook("pre_llm_call", guardrails)
```

---

### `post_llm_call`

Fires **once per turn**, after the tool-calling loop completes and the agent has produced a final response. Only fires on **successful** turns — does not fire if the turn was interrupted.

**Callback signature:**

```python
def my_callback(session_id: str, user_message: str, assistant_response: str,
                conversation_history: list, model: str, platform: str, **kwargs):
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `session_id` | `str` | Unique identifier for the current session |
| `user_message` | `str` | The user's original message for this turn |
| `assistant_response` | `str` | The agent's final text response for this turn |
| `conversation_history` | `list` | Copy of the full message list after the turn completed |
| `model` | `str` | The model identifier |
| `platform` | `str` | Where the session is running |

**Fires:** In `run_agent.py`, inside `run_conversation()`, after the tool loop exits with a final response. Guarded by `if final_response and not interrupted` — so it does **not** fire when the user interrupts mid-turn or the agent hits the iteration limit without producing a response.

**Return value:** Ignored.

**Use cases:** Syncing conversation data to an external memory system, computing response quality metrics, logging turn summaries, triggering follow-up actions.

**Example — sync to external memory:**

```python
import httpx

MEMORY_API = "https://your-memory-api.example.com"

def sync_memory(session_id, user_message, assistant_response, **kwargs):
    try:
        httpx.post(f"{MEMORY_API}/store", json={
            "session_id": session_id,
            "user": user_message,
            "assistant": assistant_response,
        }, timeout=5)
    except Exception:
        pass  # best-effort

def register(ctx):
    ctx.register_hook("post_llm_call", sync_memory)
```

**Example — track response lengths:**

```python
import logging
logger = logging.getLogger(__name__)

def log_response_length(session_id, assistant_response, model, **kwargs):
    logger.info("RESPONSE session=%s model=%s chars=%d",
                session_id, model, len(assistant_response or ""))

def register(ctx):
    ctx.register_hook("post_llm_call", log_response_length)
```

---

### `on_session_start`

Fires **once** when a brand-new session is created. Does **not** fire on session continuation (when the user sends a second message in an existing session).

**Callback signature:**

```python
def my_callback(session_id: str, model: str, platform: str, **kwargs):
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `session_id` | `str` | Unique identifier for the new session |
| `model` | `str` | The model identifier |
| `platform` | `str` | Where the session is running |

**Fires:** In `run_agent.py`, inside `run_conversation()`, during the first turn of a new session — specifically after the system prompt is built but before the tool loop starts. The check is `if not conversation_history` (no prior messages = new session).

**Return value:** Ignored.

**Use cases:** Initializing session-scoped state, warming caches, registering the session with an external service, logging session starts.

**Example — initialize a session cache:**

```python
_session_caches = {}

def init_session(session_id, model, platform, **kwargs):
    _session_caches[session_id] = {
        "model": model,
        "platform": platform,
        "tool_calls": 0,
        "started": __import__("datetime").datetime.now().isoformat(),
    }

def register(ctx):
    ctx.register_hook("on_session_start", init_session)
```

---

### `on_session_end`

Fires at the **very end** of every `run_conversation()` call, regardless of outcome. Also fires from the CLI's exit handler if the agent was mid-turn when the user quit.

**Callback signature:**

```python
def my_callback(session_id: str, completed: bool, interrupted: bool,
                model: str, platform: str, **kwargs):
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `session_id` | `str` | Unique identifier for the session |
| `completed` | `bool` | `True` if the agent produced a final response, `False` otherwise |
| `interrupted` | `bool` | `True` if the turn was interrupted (user sent new message, `/stop`, or quit) |
| `model` | `str` | The model identifier |
| `platform` | `str` | Where the session is running |

**Fires:** In two places:
1. **`run_agent.py`** — at the end of every `run_conversation()` call, after all cleanup. Always fires, even if the turn errored.
2. **`cli.py`** — in the CLI's atexit handler, but **only** if the agent was mid-turn (`_agent_running=True`) when the exit occurred. This catches Ctrl+C and `/exit` during processing. In this case, `completed=False` and `interrupted=True`.

**Return value:** Ignored.

**Use cases:** Flushing buffers, closing connections, persisting session state, logging session duration, cleanup of resources initialized in `on_session_start`.

**Example — flush and cleanup:**

```python
_session_caches = {}

def cleanup_session(session_id, completed, interrupted, **kwargs):
    cache = _session_caches.pop(session_id, None)
    if cache:
        # Flush accumulated data to disk or external service
        status = "completed" if completed else ("interrupted" if interrupted else "failed")
        print(f"Session {session_id} ended: {status}, {cache['tool_calls']} tool calls")

def register(ctx):
    ctx.register_hook("on_session_end", cleanup_session)
```

**Example — session duration tracking:**

```python
import time, logging
logger = logging.getLogger(__name__)

_start_times = {}

def on_start(session_id, **kwargs):
    _start_times[session_id] = time.time()

def on_end(session_id, completed, interrupted, **kwargs):
    start = _start_times.pop(session_id, None)
    if start:
        duration = time.time() - start
        logger.info("SESSION_DURATION session=%s seconds=%.1f completed=%s interrupted=%s",
                     session_id, duration, completed, interrupted)

def register(ctx):
    ctx.register_hook("on_session_start", on_start)
    ctx.register_hook("on_session_end", on_end)
```

---

See the **[Build a Plugin guide](/docs/guides/build-a-hermes-plugin)** for the full walkthrough including tool schemas, handlers, and advanced hook patterns.
