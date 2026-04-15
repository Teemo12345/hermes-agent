---
sidebar_position: 2
title: "技能系统"
description: "按需知识文档 — 渐进式披露、智能体管理的技能和技能中心"
---

# 技能系统

技能是智能体可以在需要时加载的按需知识文档。它们遵循**渐进式披露**模式以最小化令牌使用，并与 [agentskills.io](https://agentskills.io/specification) 开放标准兼容。

所有技能都存储在 **`~/.hermes/skills/`** 目录中 — 这是主要目录和真实来源。在全新安装时，捆绑的技能会从仓库复制。通过中心安装和智能体创建的技能也会存放在这里。智能体可以修改或删除任何技能。

您还可以将 Hermes 指向**外部技能目录** — 与本地目录一起扫描的其他文件夹。请参阅下面的[外部技能目录](#external-skill-directories)。

另请参阅：

- [捆绑技能目录](/docs/reference/skills-catalog)
- [官方可选技能目录](/docs/reference/optional-skills-catalog)

## 使用技能

每个已安装的技能都会自动作为斜杠命令可用：

```bash
# 在 CLI 或任何消息平台中：
/gif-search funny cats
/axolotl help me fine-tune Llama 3 on my dataset
/github-pr-workflow create a PR for the auth refactor
/plan design a rollout for migrating our auth provider

# 仅技能名称会加载它并让智能体询问您需要什么：
/excalidraw
```

捆绑的 `plan` 技能是一个带有自定义行为的基于技能的斜杠命令的好例子。运行 `/plan [request]` 会告诉 Hermes 在需要时检查上下文，编写 Markdown 实现计划而不是执行任务，并将结果保存在相对于活动工作区/后端工作目录的 `.hermes/plans/` 下。

您也可以通过自然对话与技能交互：

```bash
hermes chat --toolsets skills -q "What skills do you have?"
hermes chat --toolsets skills -q "Show me the axolotl skill"
```

## 渐进式披露

技能使用令牌高效的加载模式：

```
Level 0: skills_list()           → [{name, description, category}, ...]   (~3k tokens)
Level 1: skill_view(name)        → 完整内容 + 元数据       ( varies )
Level 2: skill_view(name, path)  → 特定参考文件       ( varies )
```

智能体只有在实际需要时才会加载完整的技能内容。

## SKILL.md 格式

```markdown
---
name: my-skill
description: 此技能功能的简要描述
version: 1.0.0
platforms: [macos, linux]     # 可选 — 限制为特定操作系统平台
metadata:
  hermes:
    tags: [python, automation]
    category: devops
    fallback_for_toolsets: [web]    # 可选 — 条件激活（见下文）
    requires_toolsets: [terminal]   # 可选 — 条件激活（见下文）
    config:                          # 可选 — config.yaml 设置
      - key: my.setting
        description: "此设置控制什么"
        default: "value"
        prompt: "设置提示"
---

# 技能标题

## 使用时机
此技能的触发条件。

## 步骤
1. 第一步
2. 第二步

## 注意事项
- 已知的失败模式和修复方法

## 验证
如何确认它有效。
```

### 平台特定技能

技能可以使用 `platforms` 字段将自己限制在特定的操作系统上：

| 值 | 匹配 |
|-------|---------|
| `macos` | macOS (Darwin) |
| `linux` | Linux |
| `windows` | Windows |

```yaml
platforms: [macos]            # 仅 macOS（例如，iMessage、Apple Reminders、FindMy）
platforms: [macos, linux]     # macOS 和 Linux
```

设置后，该技能会在不兼容的平台上自动从系统提示、`skills_list()` 和斜杠命令中隐藏。如果省略，则该技能会在所有平台上加载。

### 条件激活（后备技能）

技能可以根据当前会话中可用的工具自动显示或隐藏自己。这对于**后备技能**最有用 — 当高级工具不可用时才应该出现的免费或本地替代方案。

```yaml
metadata:
  hermes:
    fallback_for_toolsets: [web]      # 仅当这些工具集不可用时显示
    requires_toolsets: [terminal]     # 仅当这些工具集可用时显示
    fallback_for_tools: [web_search]  # 仅当这些特定工具不可用时显示
    requires_tools: [terminal]        # 仅当这些特定工具可用时显示
```

| 字段 | 行为 |
|-------|----------|
| `fallback_for_toolsets` | 当列出的工具集可用时，技能会**隐藏**。当它们缺失时显示。 |
| `fallback_for_tools` | 相同，但检查单个工具而不是工具集。 |
| `requires_toolsets` | 当列出的工具集不可用时，技能会**隐藏**。当它们存在时显示。 |
| `requires_tools` | 相同，但检查单个工具。 |

**示例：** 内置的 `duckduckgo-search` 技能使用 `fallback_for_toolsets: [web]`。当您设置了 `FIRECRAWL_API_KEY` 时，web 工具集可用，智能体使用 `web_search` — DuckDuckGo 技能保持隐藏。如果 API 密钥缺失，web 工具集不可用，DuckDuckGo 技能会自动作为后备方案出现。

没有任何条件字段的技能行为与以前完全一样 — 它们总是显示。

## 加载时的安全设置

技能可以声明所需的环境变量，而不会从发现中消失：

```yaml
required_environment_variables:
  - name: TENOR_API_KEY
    prompt: Tenor API key
    help: Get a key from https://developers.google.com/tenor
    required_for: full functionality
```

当遇到缺失值时，Hermes 仅在技能实际在本地 CLI 中加载时才安全地要求提供。您可以跳过设置并继续使用该技能。消息界面永远不会在聊天中要求提供密钥 — 它们会告诉您改为在本地使用 `hermes setup` 或 `~/.hermes/.env`。

设置后，声明的环境变量会**自动传递**到 `execute_code` 和 `terminal` 沙箱 — 技能的脚本可以直接使用 `$TENOR_API_KEY`。对于非技能环境变量，请使用 `terminal.env_passthrough` 配置选项。有关详细信息，请参阅[环境变量传递](/docs/user-guide/security#environment-variable-passthrough)。

### 技能配置设置

技能还可以声明存储在 `config.yaml` 中的非密钥配置设置（路径、偏好设置）：

```yaml
metadata:
  hermes:
    config:
      - key: wiki.path
        description: Path to the wiki directory
        default: "~/wiki"
        prompt: Wiki directory path
```

Settings are stored under `skills.config` in your config.yaml. `hermes config migrate` prompts for unconfigured settings, and `hermes config show` displays them. When a skill loads, its resolved config values are injected into the context so the agent knows the configured values automatically.

See [Skill Settings](/docs/user-guide/configuration#skill-settings) and [Creating Skills — Config Settings](/docs/developer-guide/creating-skills#config-settings-configyaml) for details.

## Skill Directory Structure

```text
~/.hermes/skills/                  # Single source of truth
├── mlops/                         # Category directory
│   ├── axolotl/
│   │   ├── SKILL.md               # Main instructions (required)
│   │   ├── references/            # Additional docs
│   │   ├── templates/             # Output formats
│   │   ├── scripts/               # Helper scripts callable from the skill
│   │   └── assets/                # Supplementary files
│   └── vllm/
│       └── SKILL.md
├── devops/
│   └── deploy-k8s/                # Agent-created skill
│       ├── SKILL.md
│       └── references/
├── .hub/                          # Skills Hub state
│   ├── lock.json
│   ├── quarantine/
│   └── audit.log
└── .bundled_manifest              # Tracks seeded bundled skills
```

## External Skill Directories

If you maintain skills outside of Hermes — for example, a shared `~/.agents/skills/` directory used by multiple AI tools — you can tell Hermes to scan those directories too.

Add `external_dirs` under the `skills` section in `~/.hermes/config.yaml`:

```yaml
skills:
  external_dirs:
    - ~/.agents/skills
    - /home/shared/team-skills
    - ${SKILLS_REPO}/skills
```

Paths support `~` expansion and `${VAR}` environment variable substitution.

### How it works

- **Read-only**: External dirs are only scanned for skill discovery. When the agent creates or edits a skill, it always writes to `~/.hermes/skills/`.
- **Local precedence**: If the same skill name exists in both the local dir and an external dir, the local version wins.
- **Full integration**: External skills appear in the system prompt index, `skills_list`, `skill_view`, and as `/skill-name` slash commands — no different from local skills.
- **Non-existent paths are silently skipped**: If a configured directory doesn't exist, Hermes ignores it without errors. Useful for optional shared directories that may not be present on every machine.

### Example

```text
~/.hermes/skills/               # Local (primary, read-write)
├── devops/deploy-k8s/
│   └── SKILL.md
└── mlops/axolotl/
    └── SKILL.md

~/.agents/skills/               # External (read-only, shared)
├── my-custom-workflow/
│   └── SKILL.md
└── team-conventions/
    └── SKILL.md
```

All four skills appear in your skill index. If you create a new skill called `my-custom-workflow` locally, it shadows the external version.

## Agent-Managed Skills (skill_manage tool)

The agent can create, update, and delete its own skills via the `skill_manage` tool. This is the agent's **procedural memory** — when it figures out a non-trivial workflow, it saves the approach as a skill for future reuse.

### When the Agent Creates Skills

- After completing a complex task (5+ tool calls) successfully
- When it hit errors or dead ends and found the working path
- When the user corrected its approach
- When it discovered a non-trivial workflow

### Actions

| Action | Use for | Key params |
|--------|---------|------------|
| `create` | New skill from scratch | `name`, `content` (full SKILL.md), optional `category` |
| `patch` | Targeted fixes (preferred) | `name`, `old_string`, `new_string` |
| `edit` | Major structural rewrites | `name`, `content` (full SKILL.md replacement) |
| `delete` | Remove a skill entirely | `name` |
| `write_file` | Add/update supporting files | `name`, `file_path`, `file_content` |
| `remove_file` | Remove a supporting file | `name`, `file_path` |

:::tip
The `patch` action is preferred for updates — it's more token-efficient than `edit` because only the changed text appears in the tool call.
:::

## Skills Hub

Browse, search, install, and manage skills from online registries, `skills.sh`, direct well-known skill endpoints, and official optional skills.

### Common commands

```bash
hermes skills browse                              # Browse all hub skills (official first)
hermes skills browse --source official            # Browse only official optional skills
hermes skills search kubernetes                   # Search all sources
hermes skills search react --source skills-sh     # Search the skills.sh directory
hermes skills search https://mintlify.com/docs --source well-known
hermes skills inspect openai/skills/k8s           # Preview before installing
hermes skills install openai/skills/k8s           # Install with security scan
hermes skills install official/security/1password
hermes skills install skills-sh/vercel-labs/json-render/json-render-react --force
hermes skills install well-known:https://mintlify.com/docs/.well-known/skills/mintlify
hermes skills list --source hub                   # List hub-installed skills
hermes skills check                               # Check installed hub skills for upstream updates
hermes skills update                              # Reinstall hub skills with upstream changes when needed
hermes skills audit                               # Re-scan all hub skills for security
hermes skills uninstall k8s                       # Remove a hub skill
hermes skills publish skills/my-skill --to github --repo owner/repo
hermes skills snapshot export setup.json          # Export skill config
hermes skills tap add myorg/skills-repo           # Add a custom GitHub source
```

### Supported hub sources

| Source | Example | Notes |
|--------|---------|-------|
| `official` | `official/security/1password` | Optional skills shipped with Hermes. |
| `skills-sh` | `skills-sh/vercel-labs/agent-skills/vercel-react-best-practices` | Searchable via `hermes skills search <query> --source skills-sh`. Hermes resolves alias-style skills when the skills.sh slug differs from the repo folder. |
| `well-known` | `well-known:https://mintlify.com/docs/.well-known/skills/mintlify` | Skills served directly from `/.well-known/skills/index.json` on a website. Search using the site or docs URL. |
| `github` | `openai/skills/k8s` | Direct GitHub repo/path installs and custom taps. |
| `clawhub`, `lobehub`, `claude-marketplace` | Source-specific identifiers | Community or marketplace integrations. |

### Integrated hubs and registries

Hermes currently integrates with these skills ecosystems and discovery sources:

#### 1. Official optional skills (`official`)

These are maintained in the Hermes repository itself and install with builtin trust.

- Catalog: [Official Optional Skills Catalog](../../reference/optional-skills-catalog)
- Source in repo: `optional-skills/`
- Example:

```bash
hermes skills browse --source official
hermes skills install official/security/1password
```

#### 2. skills.sh (`skills-sh`)

This is Vercel's public skills directory. Hermes can search it directly, inspect skill detail pages, resolve alias-style slugs, and install from the underlying source repo.

- Directory: [skills.sh](https://skills.sh/)
- CLI/tooling repo: [vercel-labs/skills](https://github.com/vercel-labs/skills)
- Official Vercel skills repo: [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills)
- Example:

```bash
hermes skills search react --source skills-sh
hermes skills inspect skills-sh/vercel-labs/json-render/json-render-react
hermes skills install skills-sh/vercel-labs/json-render/json-render-react --force
```

#### 3. Well-known skill endpoints (`well-known`)

This is URL-based discovery from sites that publish `/.well-known/skills/index.json`. It is not a single centralized hub — it is a web discovery convention.

- Example live endpoint: [Mintlify docs skills index](https://mintlify.com/docs/.well-known/skills/index.json)
- Reference server implementation: [vercel-labs/skills-handler](https://github.com/vercel-labs/skills-handler)
- Example:

```bash
hermes skills search https://mintlify.com/docs --source well-known
hermes skills inspect well-known:https://mintlify.com/docs/.well-known/skills/mintlify
hermes skills install well-known:https://mintlify.com/docs/.well-known/skills/mintlify
```

#### 4. Direct GitHub skills (`github`)

Hermes can install directly from GitHub repositories and GitHub-based taps. This is useful when you already know the repo/path or want to add your own custom source repo.

Default taps (browsable without any setup):
- [openai/skills](https://github.com/openai/skills)
- [anthropics/skills](https://github.com/anthropics/skills)
- [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills)
- [garrytan/gstack](https://github.com/garrytan/gstack)

- Example:

```bash
hermes skills install openai/skills/k8s
hermes skills tap add myorg/skills-repo
```

#### 5. ClawHub (`clawhub`)

A third-party skills marketplace integrated as a community source.

- Site: [clawhub.ai](https://clawhub.ai/)
- Hermes source id: `clawhub`

#### 6. Claude marketplace-style repos (`claude-marketplace`)

Hermes supports marketplace repos that publish Claude-compatible plugin/marketplace manifests.

Known integrated sources include:
- [anthropics/skills](https://github.com/anthropics/skills)
- [aiskillstore/marketplace](https://github.com/aiskillstore/marketplace)

Hermes source id: `claude-marketplace`

#### 7. LobeHub (`lobehub`)

Hermes can search and convert agent entries from LobeHub's public catalog into installable Hermes skills.

- Site: [LobeHub](https://lobehub.com/)
- Public agents index: [chat-agents.lobehub.com](https://chat-agents.lobehub.com/)
- Backing repo: [lobehub/lobe-chat-agents](https://github.com/lobehub/lobe-chat-agents)
- Hermes source id: `lobehub`

### Security scanning and `--force`

All hub-installed skills go through a **security scanner** that checks for data exfiltration, prompt injection, destructive commands, supply-chain signals, and other threats.

`hermes skills inspect ...` now also surfaces upstream metadata when available:
- repo URL
- skills.sh detail page URL
- install command
- weekly installs
- upstream security audit statuses
- well-known index/endpoint URLs

Use `--force` when you have reviewed a third-party skill and want to override a non-dangerous policy block:

```bash
hermes skills install skills-sh/anthropics/skills/pdf --force
```

Important behavior:
- `--force` can override policy blocks for caution/warn-style findings.
- `--force` does **not** override a `dangerous` scan verdict.
- Official optional skills (`official/...`) are treated as builtin trust and do not show the third-party warning panel.

### Trust levels

| Level | Source | Policy |
|-------|--------|--------|
| `builtin` | Ships with Hermes | Always trusted |
| `official` | `optional-skills/` in the repo | Builtin trust, no third-party warning |
| `trusted` | Trusted registries/repos such as `openai/skills`, `anthropics/skills` | More permissive policy than community sources |
| `community` | Everything else (`skills.sh`, well-known endpoints, custom GitHub repos, most marketplaces) | Non-dangerous findings can be overridden with `--force`; `dangerous` verdicts stay blocked |

### Update lifecycle

The hub now tracks enough provenance to re-check upstream copies of installed skills:

```bash
hermes skills check          # Report which installed hub skills changed upstream
hermes skills update         # Reinstall only the skills with updates available
hermes skills update react   # Update one specific installed hub skill
```

This uses the stored source identifier plus the current upstream bundle content hash to detect drift.

:::tip GitHub rate limits
Skills hub operations use the GitHub API, which has a rate limit of 60 requests/hour for unauthenticated users. If you see rate-limit errors during install or search, set `GITHUB_TOKEN` in your `.env` file to increase the limit to 5,000 requests/hour. The error message includes an actionable hint when this happens.
:::

### Slash commands (inside chat)

All the same commands work with `/skills`:

```text
/skills browse
/skills search react --source skills-sh
/skills search https://mintlify.com/docs --source well-known
/skills inspect skills-sh/vercel-labs/json-render/json-render-react
/skills install openai/skills/skill-creator --force
/skills check
/skills update
/skills list
```

Official optional skills still use identifiers like `official/security/1password` and `official/migration/openclaw-migration`.
