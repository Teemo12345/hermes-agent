---
sidebar_position: 9
title: "个性与 SOUL.md"
description: "使用全局 SOUL.md、内置个性和自定义角色定义来自定义 Hermes Agent 的个性"
---

# 个性与 SOUL.md

Hermes Agent 的个性是完全可定制的。`SOUL.md` 是**主要身份** — 它是系统提示中的第一件事，定义了智能体是谁。

- `SOUL.md` — 一个持久的角色文件，位于 `HERMES_HOME` 中，作为智能体的身份（系统提示中的槽位 #1）
- 内置或自定义 `/personality` 预设 — 会话级系统提示覆盖

如果您想更改 Hermes 的身份 — 或用完全不同的智能体角色替换它 — 请编辑 `SOUL.md`。

## SOUL.md 现在的工作方式

Hermes 现在会自动在以下位置生成默认的 `SOUL.md`：

```text
~/.hermes/SOUL.md
```

更准确地说，它使用当前实例的 `HERMES_HOME`，所以如果您使用自定义主目录运行 Hermes，它会使用：

```text
$HERMES_HOME/SOUL.md
```

### 重要行为

- **SOUL.md 是智能体的主要身份。** 它占据系统提示中的槽位 #1，替换硬编码的默认身份。
- 如果 `SOUL.md` 尚不存在，Hermes 会自动创建一个初始版本
- 现有的用户 `SOUL.md` 文件永远不会被覆盖
- Hermes 只从 `HERMES_HOME` 加载 `SOUL.md`
- Hermes 不会在当前工作目录中查找 `SOUL.md`
- 如果 `SOUL.md` 存在但为空，或无法加载，Hermes 会回退到内置的默认身份
- 如果 `SOUL.md` 有内容，该内容会在安全扫描和截断后逐字注入
- SOUL.md **不会**在上下文文件部分重复 — 它只出现一次，作为身份

这使得 `SOUL.md` 成为真正的每用户或每实例身份，而不仅仅是一个附加层。

## 为什么采用这种设计

这使个性保持可预测性。

如果 Hermes 从您碰巧启动它的任何目录加载 `SOUL.md`，您的个性可能会在项目之间意外更改。通过仅从 `HERMES_HOME` 加载，个性属于 Hermes 实例本身。

这也使向用户传授变得更容易：
- "编辑 `~/.hermes/SOUL.md` 以更改 Hermes 的默认个性。"

## 在哪里编辑它

对于大多数用户：

```bash
~/.hermes/SOUL.md
```

如果您使用自定义主目录：

```bash
$HERMES_HOME/SOUL.md
```

## SOUL.md 中应该包含什么？

使用它来提供持久的声音和个性指导，例如：
- 语气
- 沟通风格
- 直接程度
- 默认互动风格
- 风格上应该避免的内容
- Hermes 应如何处理不确定性、分歧或歧义

较少用于：
- 一次性项目说明
- 文件路径
- 仓库约定
- 临时工作流详细信息

这些属于 `AGENTS.md`，而不是 `SOUL.md`。

## 良好的 SOUL.md 内容

一个好的 SOUL 文件应该：
- 在不同上下文中保持稳定
- 足够广泛，适用于许多对话
- 足够具体，能够实质性地塑造声音
- 专注于沟通和身份，而不是特定任务的指令

### 示例

```markdown
# Personality

You are a pragmatic senior engineer with strong taste.
You optimize for truth, clarity, and usefulness over politeness theater.

## Style
- Be direct without being cold
- Prefer substance over filler
- Push back when something is a bad idea
- Admit uncertainty plainly
- Keep explanations compact unless depth is useful

## What to avoid
- Sycophancy
- Hype language
- Repeating the user's framing if it's wrong
- Overexplaining obvious things

## Technical posture
- Prefer simple systems over clever systems
- Care about operational reality, not idealized architecture
- Treat edge cases as part of the design, not cleanup
```

## What Hermes injects into the prompt

`SOUL.md` content goes directly into slot #1 of the system prompt — the agent identity position. No wrapper language is added around it.

The content goes through:
- prompt-injection scanning
- truncation if it is too large

If the file is empty, whitespace-only, or cannot be read, Hermes falls back to a built-in default identity ("You are Hermes Agent, an intelligent AI assistant created by Nous Research..."). This fallback also applies when `skip_context_files` is set (e.g., in subagent/delegation contexts).

## Security scanning

`SOUL.md` is scanned like other context-bearing files for prompt injection patterns before inclusion.

That means you should still keep it focused on persona/voice rather than trying to sneak in strange meta-instructions.

## SOUL.md vs AGENTS.md

This is the most important distinction.

### SOUL.md
Use for:
- identity
- tone
- style
- communication defaults
- personality-level behavior

### AGENTS.md
Use for:
- project architecture
- coding conventions
- tool preferences
- repo-specific workflows
- commands, ports, paths, deployment notes

A useful rule:
- if it should follow you everywhere, it belongs in `SOUL.md`
- if it belongs to a project, it belongs in `AGENTS.md`

## SOUL.md vs `/personality`

`SOUL.md` is your durable default personality.

`/personality` is a session-level overlay that changes or supplements the current system prompt.

So:
- `SOUL.md` = baseline voice
- `/personality` = temporary mode switch

Examples:
- keep a pragmatic default SOUL, then use `/personality teacher` for a tutoring conversation
- keep a concise SOUL, then use `/personality creative` for brainstorming

## Built-in personalities

Hermes ships with built-in personalities you can switch to with `/personality`.

| Name | Description |
|------|-------------|
| **helpful** | Friendly, general-purpose assistant |
| **concise** | Brief, to-the-point responses |
| **technical** | Detailed, accurate technical expert |
| **creative** | Innovative, outside-the-box thinking |
| **teacher** | Patient educator with clear examples |
| **kawaii** | Cute expressions, sparkles, and enthusiasm ★ |
| **catgirl** | Neko-chan with cat-like expressions, nya~ |
| **pirate** | Captain Hermes, tech-savvy buccaneer |
| **shakespeare** | Bardic prose with dramatic flair |
| **surfer** | Totally chill bro vibes |
| **noir** | Hard-boiled detective narration |
| **uwu** | Maximum cute with uwu-speak |
| **philosopher** | Deep contemplation on every query |
| **hype** | MAXIMUM ENERGY AND ENTHUSIASM!!! |

## Switching personalities with commands

### CLI

```text
/personality
/personality concise
/personality technical
```

### Messaging platforms

```text
/personality teacher
```

These are convenient overlays, but your global `SOUL.md` still gives Hermes its persistent default personality unless the overlay meaningfully changes it.

## Custom personalities in config

You can also define named custom personalities in `~/.hermes/config.yaml` under `agent.personalities`.

```yaml
agent:
  personalities:
    codereviewer: >
      You are a meticulous code reviewer. Identify bugs, security issues,
      performance concerns, and unclear design choices. Be precise and constructive.
```

Then switch to it with:

```text
/personality codereviewer
```

## Recommended workflow

A strong default setup is:

1. Keep a thoughtful global `SOUL.md` in `~/.hermes/SOUL.md`
2. Put project instructions in `AGENTS.md`
3. Use `/personality` only when you want a temporary mode shift

That gives you:
- a stable voice
- project-specific behavior where it belongs
- temporary control when needed

## How personality interacts with the full prompt

At a high level, the prompt stack includes:
1. **SOUL.md** (agent identity — or built-in fallback if SOUL.md is unavailable)
2. tool-aware behavior guidance
3. memory/user context
4. skills guidance
5. context files (`AGENTS.md`, `.cursorrules`)
6. timestamp
7. platform-specific formatting hints
8. optional system-prompt overlays such as `/personality`

`SOUL.md` is the foundation — everything else builds on top of it.

## Related docs

- [Context Files](/docs/user-guide/features/context-files)
- [Configuration](/docs/user-guide/configuration)
- [Tips & Best Practices](/docs/guides/tips)
- [SOUL.md Guide](/docs/guides/use-soul-with-hermes)

## CLI appearance vs conversational personality

Conversational personality and CLI appearance are separate:

- `SOUL.md`, `agent.system_prompt`, and `/personality` affect how Hermes speaks
- `display.skin` and `/skin` affect how Hermes looks in the terminal

For terminal appearance, see [Skins & Themes](./skins.md).
