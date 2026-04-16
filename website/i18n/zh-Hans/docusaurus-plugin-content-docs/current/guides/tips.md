---
sidebar_position: 1
title: "技巧与最佳实践"
description: "充分利用 Hermes Agent 的实用建议 — 提示技巧、CLI 快捷键、上下文文件、记忆、成本优化和安全性"
---

# 技巧与最佳实践

一系列实用的技巧集合，让您立即更有效地使用 Hermes Agent。每个部分针对不同的方面 — 扫描标题并跳转到相关内容。

---

## 获得最佳结果

### 具体说明您想要什么

模糊的提示会产生模糊的结果。不要说"修复代码"，而要说"修复 `api/handlers.py` 第 47 行的 TypeError — `process_request()` 函数从 `parse_body()` 接收到 `None`"。您提供的上下文越多，需要的迭代次数就越少。

### 提前提供上下文

在请求前面加载相关细节：文件路径、错误消息、预期行为。一个精心制作的消息胜过三轮澄清。直接粘贴错误回溯 — 智能体可以解析它们。

### 使用上下文文件处理重复指令

如果您发现自己重复相同的指令（"使用制表符而不是空格"、"我们使用 pytest"、"API 在 `/api/v2`"），请将它们放在 `AGENTS.md` 文件中。智能体在每个会话中自动读取它 — 设置后零努力。

### 让智能体使用其工具

不要试图手把手指导每一步。说"查找并修复失败的测试"，而不是"打开 `tests/test_foo.py`，查看第 42 行，然后..."。智能体具有文件搜索、终端访问和代码执行功能 — 让它探索和迭代。

### 使用技能处理复杂工作流

在编写长提示解释如何做某事之前，检查是否已经有相应的技能。输入 `/skills` 浏览可用技能，或直接调用一个，如 `/axolotl` 或 `/github-pr-workflow`。

## CLI 高级用户技巧

### 多行输入

按 **Alt+Enter**（或 **Ctrl+J**）插入新行而不发送。这使您可以在按 Enter 发送之前编写多行提示、粘贴代码块或构建复杂请求。

### 粘贴检测

CLI 自动检测多行粘贴。只需直接粘贴代码块或错误回溯 — 它不会将每行作为单独的消息发送。粘贴被缓冲并作为一条消息发送。

### 中断和重定向

按 **Ctrl+C** 一次以在响应过程中中断智能体。然后您可以输入新消息来重定向它。在 2 秒内双击 Ctrl+C 以强制退出。当智能体开始走错路时，这非常宝贵。

### 使用 `-c` 恢复会话

Forgot something from your last session? Run `hermes -c` to resume exactly where you left off, with full conversation history restored. You can also resume by title: `hermes -r "my research project"`.

### Clipboard Image Paste

Press **Ctrl+V** to paste an image from your clipboard directly into the chat. The agent uses vision to analyze screenshots, diagrams, error popups, or UI mockups — no need to save to a file first.

### Slash Command Autocomplete

Type `/` and press **Tab** to see all available commands. This includes built-in commands (`/compress`, `/model`, `/title`) and every installed skill. You don't need to memorize anything — Tab completion has you covered.

:::tip
Use `/verbose` to cycle through tool output display modes: **off → new → all → verbose**. The "all" mode is great for watching what the agent does; "off" is cleanest for simple Q&A.
:::

## Context Files

### AGENTS.md: Your Project's Brain

Create an `AGENTS.md` in your project root with architecture decisions, coding conventions, and project-specific instructions. This is automatically injected into every session, so the agent always knows your project's rules.

```markdown
# Project Context
- This is a FastAPI backend with SQLAlchemy ORM
- Always use async/await for database operations
- Tests go in tests/ and use pytest-asyncio
- Never commit .env files
```

### SOUL.md: Customize Personality

Want Hermes to have a stable default voice? Edit `~/.hermes/SOUL.md` (or `$HERMES_HOME/SOUL.md` if you use a custom Hermes home). Hermes now seeds a starter SOUL automatically and uses that global file as the instance-wide personality source.

For a full walkthrough, see [Use SOUL.md with Hermes](/docs/guides/use-soul-with-hermes).

```markdown
# Soul
You are a senior backend engineer. Be terse and direct.
Skip explanations unless asked. Prefer one-liners over verbose solutions.
Always consider error handling and edge cases.
```

Use `SOUL.md` for durable personality. Use `AGENTS.md` for project-specific instructions.

### .cursorrules Compatibility

Already have a `.cursorrules` or `.cursor/rules/*.mdc` file? Hermes reads those too. No need to duplicate your coding conventions — they're loaded automatically from the working directory.

### Discovery

Hermes loads the top-level `AGENTS.md` from the current working directory at session start. Subdirectory `AGENTS.md` files are discovered lazily during tool calls (via `subdirectory_hints.py`) and injected into tool results — they are not loaded upfront into the system prompt.

:::tip
Keep context files focused and concise. Every character counts against your token budget since they're injected into every single message.
:::

## Memory & Skills

### Memory vs. Skills: What Goes Where

**Memory** is for facts: your environment, preferences, project locations, and things the agent has learned about you. **Skills** are for procedures: multi-step workflows, tool-specific instructions, and reusable recipes. Use memory for "what," skills for "how."

### When to Create Skills

If you find a task that takes 5+ steps and you'll do it again, ask the agent to create a skill for it. Say "save what you just did as a skill called `deploy-staging`." Next time, just type `/deploy-staging` and the agent loads the full procedure.

### Managing Memory Capacity

Memory is intentionally bounded (~2,200 chars for MEMORY.md, ~1,375 chars for USER.md). When it fills up, the agent consolidates entries. You can help by saying "clean up your memory" or "replace the old Python 3.9 note — we're on 3.12 now."

### Let the Agent Remember

After a productive session, say "remember this for next time" and the agent will save the key takeaways. You can also be specific: "save to memory that our CI uses GitHub Actions with the `deploy.yml` workflow."

:::warning
Memory is a frozen snapshot — changes made during a session don't appear in the system prompt until the next session starts. The agent writes to disk immediately, but the prompt cache isn't invalidated mid-session.
:::

## Performance & Cost

### Don't Break the Prompt Cache

Most LLM providers cache the system prompt prefix. If you keep your system prompt stable (same context files, same memory), subsequent messages in a session get **cache hits** that are significantly cheaper. Avoid changing the model or system prompt mid-session.

### Use /compress Before Hitting Limits

Long sessions accumulate tokens. When you notice responses slowing down or getting truncated, run `/compress`. This summarizes the conversation history, preserving key context while dramatically reducing token count. Use `/usage` to check where you stand.

### Delegate for Parallel Work

Need to research three topics at once? Ask the agent to use `delegate_task` with parallel subtasks. Each subagent runs independently with its own context, and only the final summaries come back — massively reducing your main conversation's token usage.

### Use execute_code for Batch Operations

Instead of running terminal commands one at a time, ask the agent to write a script that does everything at once. "Write a Python script to rename all `.jpeg` files to `.jpg` and run it" is cheaper and faster than renaming files individually.

### Choose the Right Model

Use `/model` to switch models mid-session. Use a frontier model (Claude Sonnet/Opus, GPT-4o) for complex reasoning and architecture decisions. Switch to a faster model for simple tasks like formatting, renaming, or boilerplate generation.

:::tip
Run `/usage` periodically to see your token consumption. Run `/insights` for a broader view of usage patterns over the last 30 days.
:::

## Messaging Tips

### Set a Home Channel

Use `/sethome` in your preferred Telegram or Discord chat to designate it as the home channel. Cron job results and scheduled task outputs are delivered here. Without it, the agent has nowhere to send proactive messages.

### Use /title to Organize Sessions

Name your sessions with `/title auth-refactor` or `/title research-llm-quantization`. Named sessions are easy to find with `hermes sessions list` and resume with `hermes -r "auth-refactor"`. Unnamed sessions pile up and become impossible to distinguish.

### DM Pairing for Team Access

Instead of manually collecting user IDs for allowlists, enable DM pairing. When a teammate DMs the bot, they get a one-time pairing code. You approve it with `hermes pairing approve telegram XKGH5N7P` — simple and secure.

### Tool Progress Display Modes

Use `/verbose` to control how much tool activity you see. In messaging platforms, less is usually more — keep it on "new" to see just new tool calls. In the CLI, "all" gives you a satisfying live view of everything the agent does.

:::tip
On messaging platforms, sessions auto-reset after idle time (default: 24 hours) or daily at 4 AM. Adjust per-platform in `~/.hermes/config.yaml` if you need longer sessions.
:::

## Security

### Use Docker for Untrusted Code

When working with untrusted repositories or running unfamiliar code, use Docker or Daytona as your terminal backend. Set `TERMINAL_BACKEND=docker` in your `.env`. Destructive commands inside a container can't harm your host system.

```bash
# In your .env:
TERMINAL_BACKEND=docker
TERMINAL_DOCKER_IMAGE=hermes-sandbox:latest
```

### Avoid Windows Encoding Pitfalls

On Windows, some default encodings (such as `cp125x`) cannot represent all Unicode characters, which can cause `UnicodeEncodeError` when writing files in tests or scripts.

- Prefer opening files with an explicit UTF-8 encoding:

```python
with open("results.txt", "w", encoding="utf-8") as f:
    f.write("✓ All good\n")
```

- In PowerShell, you can also switch the current session to UTF-8 for console and native command output:

```powershell
$OutputEncoding = [Console]::OutputEncoding = [Text.UTF8Encoding]::new($false)
```

This keeps PowerShell and child processes on UTF-8 and helps avoid Windows-only failures.

### Review Before Choosing "Always"

When the agent triggers a dangerous command approval (`rm -rf`, `DROP TABLE`, etc.), you get four options: **once**, **session**, **always**, **deny**. Think carefully before choosing "always" — it permanently allowlists that pattern. Start with "session" until you're comfortable.

### Command Approval Is Your Safety Net

Hermes checks every command against a curated list of dangerous patterns before execution. This includes recursive deletes, SQL drops, piping curl to shell, and more. Don't disable this in production — it exists for good reasons.

:::warning
When running in a container backend (Docker, Singularity, Modal, Daytona), dangerous command checks are **skipped** because the container is the security boundary. Make sure your container images are properly locked down.
:::

### Use Allowlists for Messaging Bots

Never set `GATEWAY_ALLOW_ALL_USERS=true` on a bot with terminal access. Always use platform-specific allowlists (`TELEGRAM_ALLOWED_USERS`, `DISCORD_ALLOWED_USERS`) or DM pairing to control who can interact with your agent.

```bash
# Recommended: explicit allowlists per platform
TELEGRAM_ALLOWED_USERS=123456789,987654321
DISCORD_ALLOWED_USERS=123456789012345678

# Or use cross-platform allowlist
GATEWAY_ALLOWED_USERS=123456789,987654321
```

---

*Have a tip that should be on this page? Open an issue or PR — community contributions are welcome.*