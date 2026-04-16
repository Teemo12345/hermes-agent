---
sidebar_position: 12
title: "使用技能"
description: "查找、安装、使用和创建技能 — 按需知识，教会 Hermes 新的工作流程"
---

# 使用技能

技能是按需知识文档，教会 Hermes 如何处理特定任务 — 从生成 ASCII 艺术到管理 GitHub PR。本指南将引导您日常使用它们。

有关完整的技术参考，请参阅[技能系统](/docs/user-guide/features/skills)。

---

## 查找技能

每个 Hermes 安装都附带捆绑的技能。查看可用内容：

```bash
# 在任何聊天会话中：
/skills

# 或从 CLI：
hermes skills list
```

这将显示一个包含名称和描述的紧凑列表：

```
ascii-art         使用 pyfiglet、cowsay、boxes 生成 ASCII 艺术...
arxiv             从 arXiv 搜索和检索学术论文...
github-pr-workflow 完整的 PR 生命周期 — 创建分支、提交...
plan              计划模式 — 检查上下文，编写 markdown...
excalidraw        使用 Excalidraw 创建手绘风格图表...
```

### 搜索技能

```bash
# 按关键字搜索
/skills search docker
/skills search music
```

### 技能中心

官方可选技能（默认不激活的较重或利基技能）可通过中心获得：

```bash
# 浏览官方可选技能
/skills browse

# Search the hub
/skills search blockchain
```

---

## Using a Skill

Every installed skill is automatically a slash command. Just type its name:

```bash
# Load a skill and give it a task
/ascii-art Make a banner that says "HELLO WORLD"
/plan Design a REST API for a todo app
/github-pr-workflow Create a PR for the auth refactor

# Just the skill name (no task) loads it and lets you describe what you need
/excalidraw
```

You can also trigger skills through natural conversation — ask Hermes to use a specific skill, and it will load it via the `skill_view` tool.

### Progressive Disclosure

Skills use a token-efficient loading pattern. The agent doesn't load everything at once:

1. **`skills_list()`** — compact list of all skills (~3k tokens). Loaded at session start.
2. **`skill_view(name)`** — full SKILL.md content for one skill. Loaded when the agent decides it needs that skill.
3. **`skill_view(name, file_path)`** — a specific reference file within the skill. Only loaded if needed.

This means skills don't cost tokens until they're actually used.

---

## Installing from the Hub

Official optional skills ship with Hermes but aren't active by default. Install them explicitly:

```bash
# Install an official optional skill
hermes skills install official/research/arxiv

# Install from the hub in a chat session
/skills install official/creative/songwriting-and-ai-music
```

What happens:
1. The skill directory is copied to `~/.hermes/skills/`
2. It appears in your `skills_list` output
3. It becomes available as a slash command

:::tip
Installed skills take effect in new sessions. If you want it available in the current session, use `/reset` to start fresh, or add `--now` to invalidate the prompt cache immediately (costs more tokens on the next turn).
:::

### Verifying Installation

```bash
# Check it's there
hermes skills list | grep arxiv

# Or in chat
/skills search arxiv
```

---

## Plugin-Provided Skills

Plugins can bundle their own skills using namespaced names (`plugin:skill`). This prevents name collisions with built-in skills.

```bash
# Load a plugin skill by its qualified name
skill_view("superpowers:writing-plans")

# Built-in skill with the same base name is unaffected
skill_view("writing-plans")
```

Plugin skills are **not** listed in the system prompt and don't appear in `skills_list`. They're opt-in — load them explicitly when you know a plugin provides one. When loaded, the agent sees a banner listing sibling skills from the same plugin.

For how to ship skills in your own plugin, see [Build a Hermes Plugin → Bundle skills](/docs/guides/build-a-hermes-plugin#bundle-skills).

---

## Configuring Skill Settings

Some skills declare configuration they need in their frontmatter:

```yaml
metadata:
  hermes:
    config:
      - key: tenor.api_key
        description: "Tenor API key for GIF search"
        prompt: "Enter your Tenor API key"
        url: "https://developers.google.com/tenor/guides/quickstart"
```

When a skill with config is first loaded, Hermes prompts you for the values. They're stored in `config.yaml` under `skills.config.*`.

Manage skill config from the CLI:

```bash
# Interactive config for a specific skill
hermes skills config gif-search

# View all skill config
hermes config get skills.config
```

---

## Creating Your Own Skill

Skills are just markdown files with YAML frontmatter. Creating one takes under five minutes.

### 1. Create the Directory

```bash
mkdir -p ~/.hermes/skills/my-category/my-skill
```

### 2. Write SKILL.md

```markdown title="~/.hermes/skills/my-category/my-skill/SKILL.md"
---
name: my-skill
description: Brief description of what this skill does
version: 1.0.0
metadata:
  hermes:
    tags: [my-tag, automation]
    category: my-category
---

# My Skill

## When to Use
Use this skill when the user asks about [specific topic] or needs to [specific task].

## Procedure
1. First, check if [prerequisite] is available
2. Run `command --with-flags`
3. Parse the output and present results

## Pitfalls
- Common failure: [description]. Fix: [solution]
- Watch out for [edge case]

## Verification
Run `check-command` to confirm the result is correct.
```

### 3. Add Reference Files (Optional)

Skills can include supporting files the agent loads on demand:

```
my-skill/
├── SKILL.md                    # Main skill document
├── references/
│   ├── api-docs.md             # API reference the agent can consult
│   └── examples.md             # Example inputs/outputs
├── templates/
│   └── config.yaml             # Template files the agent can use
└── scripts/
    └── setup.sh                # Scripts the agent can execute
```

Reference these in your SKILL.md:

```markdown
For API details, load the reference: `skill_view("my-skill", "references/api-docs.md")`
```

### 4. Test It

Start a new session and try your skill:

```bash
hermes chat -q "/my-skill help me with the thing"
```

The skill appears automatically — no registration needed. Drop it in `~/.hermes/skills/` and it's live.

:::info
The agent can also create and update skills itself using `skill_manage`. After solving a complex problem, Hermes may offer to save the approach as a skill for next time.
:::

---

## Per-Platform Skill Management

Control which skills are available on which platforms:

```bash
hermes skills
```

This opens an interactive TUI where you can enable or disable skills per platform (CLI, Telegram, Discord, etc.). Useful when you want certain skills only available in specific contexts — for example, keeping development skills off Telegram.

---

## Skills vs Memory

Both are persistent across sessions, but they serve different purposes:

| | Skills | Memory |
|---|---|---|
| **What** | Procedural knowledge — how to do things | Factual knowledge — what things are |
| **When** | Loaded on demand, only when relevant | Injected into every session automatically |
| **Size** | Can be large (hundreds of lines) | Should be compact (key facts only) |
| **Cost** | Zero tokens until loaded | Small but constant token cost |
| **Examples** | "How to deploy to Kubernetes" | "User prefers dark mode, lives in PST" |
| **Who creates** | You, the agent, or installed from Hub | The agent, based on conversations |

**Rule of thumb:** If you'd put it in a reference document, it's a skill. If you'd put it on a sticky note, it's memory.

---

## Tips

**Keep skills focused.** A skill that tries to cover "all of DevOps" will be too long and too vague. A skill that covers "deploy a Python app to Fly.io" is specific enough to be genuinely useful.

**Let the agent create skills.** After a complex multi-step task, Hermes will often offer to save the approach as a skill. Say yes — these agent-authored skills capture the exact workflow including pitfalls that were discovered along the way.

**Use categories.** Organize skills into subdirectories (`~/.hermes/skills/devops/`, `~/.hermes/skills/research/`, etc.). This keeps the list manageable and helps the agent find relevant skills faster.

**Update skills when they go stale.** If you use a skill and hit issues not covered by it, tell Hermes to update the skill with what you learned. Skills that aren't maintained become liabilities.

---

*For the complete skills reference — frontmatter fields, conditional activation, external directories, and more — see [Skills System](/docs/user-guide/features/skills).*