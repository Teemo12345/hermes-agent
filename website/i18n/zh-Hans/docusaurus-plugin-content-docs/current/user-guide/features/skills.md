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

```
| 值 | 匹配 |
|-------|---------|
| `macos` | macOS (Darwin) |
| `linux` | Linux |
| `windows` | Windows |
```
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

设置存储在您的 config.yaml 中的 `skills.config` 下。`hermes config migrate` 会提示未配置的设置，而 `hermes config show` 会显示它们。当技能加载时，其解析后的配置值会注入到上下文中，以便智能体自动知道配置的值。

有关详细信息，请参阅[技能设置](/docs/user-guide/configuration#skill-settings)和[创建技能 — 配置设置](/docs/developer-guide/creating-skills#config-settings-configyaml)。

## 技能目录结构

```text
~/.hermes/skills/                  # 单一真实来源
├── mlops/                         # 分类目录
│   ├── axolotl/
│   │   ├── SKILL.md               # 主要说明（必需）
│   │   ├── references/            # 附加文档
│   │   ├── templates/             # 输出格式
│   │   ├── scripts/               # 可从技能调用的辅助脚本
│   │   └── assets/                # 补充文件
│   └── vllm/
│       └── SKILL.md
├── devops/
│   └── deploy-k8s/                # 智能体创建的技能
│       ├── SKILL.md
│       └── references/
├── .hub/                          # 技能中心状态
│   ├── lock.json
│   ├── quarantine/
│   └── audit.log
└── .bundled_manifest              # Tracks seeded bundled skills
```

## 外部技能目录

如果您在 Hermes 之外维护技能 — 例如，多个 AI 工具共享的 `~/.agents/skills/` 目录 — 您可以告诉 Hermes 也扫描这些目录。

在 `~/.hermes/config.yaml` 的 `skills` 部分下添加 `external_dirs`：

```yaml
skills:
  external_dirs:
    - ~/.agents/skills
    - /home/shared/team-skills
    - ${SKILLS_REPO}/skills
```

路径支持 `~` 扩展和 `${VAR}` 环境变量替换。

### 工作原理

- **只读**: 外部目录仅用于技能发现扫描。当智能体创建或编辑技能时，它总是写入 `~/.hermes/skills/`。
- **本地优先**: 如果相同的技能名称同时存在于本地目录和外部目录中，本地版本优先。
- **完全集成**: 外部技能出现在系统提示索引、`skills_list`、`skill_view` 中，并作为 `/skill-name` 斜杠命令 — 与本地技能没有区别。
- **不存在的路径会被静默跳过**: 如果配置的目录不存在，Hermes 会忽略它而不会报错。对于可能并非每台机器上都存在的可选共享目录很有用。

### 示例

```text
~/.hermes/skills/               # 本地（主要，读写）
├── devops/deploy-k8s/
│   └── SKILL.md
└── mlops/axolotl/
    └── SKILL.md

~/.agents/skills/               # 外部（只读，共享）
├── my-custom-workflow/
│   └── SKILL.md
└── team-conventions/
    └── SKILL.md
```

所有四个技能都出现在您的技能索引中。如果您在本地创建一个名为 `my-custom-workflow` 的新技能，它会遮蔽外部版本。

## 智能体管理的技能（skill_manage 工具）

智能体可以通过 `skill_manage` 工具创建、更新和删除自己的技能。这是智能体的**程序记忆** — 当它发现一个非平凡的工作流时，它会将方法保存为技能以供将来重用。

### 智能体何时创建技能

- 成功完成复杂任务（5+ 工具调用）后
- 当它遇到错误或死胡同并找到可行路径时
- 当用户纠正其方法时
- 当它发现非平凡的工作流时

### 操作

| 操作 | 用途 | 关键参数 |
|--------|---------|------------|
| `create` | 从零开始创建新技能 | `name`, `content`（完整的 SKILL.md），可选的 `category` |
| `patch` | 定向修复（首选） | `name`, `old_string`, `new_string` |
| `edit` | 主要结构重写 | `name`, `content`（完整的 SKILL.md 替换） |
| `delete` | 完全删除技能 | `name` |
| `write_file` | 添加/更新支持文件 | `name`, `file_path`, `file_content` |
| `remove_file` | 删除支持文件 | `name`, `file_path` |

:::tip
`patch` 操作是更新的首选 — 它比 `edit` 更令牌高效，因为只有更改的文本出现在工具调用中。
:::

## 技能中心

浏览、搜索、安装和管理来自在线注册表、`skills.sh`、直接知名技能端点和官方可选技能的技能。

### 常用命令

```bash
hermes skills browse                              # 浏览所有中心技能（官方优先）
hermes skills browse --source official            # 仅浏览官方可选技能
hermes skills search kubernetes                   # 搜索所有来源
hermes skills search react --source skills-sh     # 搜索 skills.sh 目录
hermes skills search https://mintlify.com/docs --source well-known
hermes skills inspect openai/skills/k8s           # 安装前预览
hermes skills install openai/skills/k8s           # 安装并安全扫描
hermes skills install official/security/1password
hermes skills install skills-sh/vercel-labs/json-render/json-render-react --force
hermes skills install well-known:https://mintlify.com/docs/.well-known/skills/mintlify
hermes skills list --source hub                   # 列出中心安装的技能
hermes skills check                               # 检查已安装的中心技能是否有上游更新
hermes skills update                              # 在需要时重新安装具有上游更改的中心技能
hermes skills audit                               # 重新扫描所有中心技能的安全性
hermes skills uninstall k8s                       # 移除中心技能
hermes skills publish skills/my-skill --to github --repo owner/repo
hermes skills snapshot export setup.json          # 导出技能配置
hermes skills tap add myorg/skills-repo           # 添加自定义 GitHub 来源
```

### 支持的中心来源

| 来源 | 示例 | 说明 |
|--------|---------|-------|
| `official` | `official/security/1password` | Hermes 附带的可选技能。 |
| `skills-sh` | `skills-sh/vercel-labs/agent-skills/vercel-react-best-practices` | 可通过 `hermes skills search <query> --source skills-sh` 搜索。当 skills.sh slug 与 repo 文件夹不同时，Hermes 会解析别名风格的技能。 |
| `well-known` | `well-known:https://mintlify.com/docs/.well-known/skills/mintlify` | 直接从网站上的 `/.well-known/skills/index.json` 提供的技能。使用站点或文档 URL 进行搜索。 |
| `github` | `openai/skills/k8s` | 直接 GitHub 仓库/路径安装和自定义源。 |
| `clawhub`, `lobehub`, `claude-marketplace` | 特定来源的标识符 | 社区或市场集成。 |

### 集成的中心和注册表

Hermes 目前集成了以下技能生态系统和发现源：

#### 1. 官方可选技能 (`official`)

这些技能在 Hermes 仓库本身中维护，并以内置信任安装。

- 目录：[官方可选技能目录](../../reference/optional-skills-catalog)
- 仓库中的来源：`optional-skills/`
- 示例：

```bash
hermes skills browse --source official
hermes skills install official/security/1password
```

#### 2. skills.sh (`skills-sh`)

这是 Vercel 的公共技能目录。Hermes 可以直接搜索它，检查技能详细页面，解析别名风格的 slug，并从底层源仓库安装。

- 目录：[skills.sh](https://skills.sh/)
- CLI/工具仓库：[vercel-labs/skills](https://github.com/vercel-labs/skills)
- 官方 Vercel 技能仓库：[vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills)
- 示例：

```bash
hermes skills search react --source skills-sh
hermes skills inspect skills-sh/vercel-labs/json-render/json-render-react
hermes skills install skills-sh/vercel-labs/json-render/json-render-react --force
```

#### 3. 知名技能端点 (`well-known`)

这是基于 URL 的发现机制，来自发布 `/.well-known/skills/index.json` 的站点。它不是单个中心化的中心 — 而是一种网络发现约定。

- 示例实时端点：[Mintlify 文档技能索引](https://mintlify.com/docs/.well-known/skills/index.json)
- 参考服务器实现：[vercel-labs/skills-handler](https://github.com/vercel-labs/skills-handler)
- 示例：

```bash
hermes skills search https://mintlify.com/docs --source well-known
hermes skills inspect well-known:https://mintlify.com/docs/.well-known/skills/mintlify
hermes skills install well-known:https://mintlify.com/docs/.well-known/skills/mintlify
```

#### 4. 直接 GitHub 技能 (`github`)

Hermes 可以直接从 GitHub 仓库和基于 GitHub 的源安装技能。当您已经知道仓库/路径或想要添加自己的自定义源仓库时，这很有用。

默认源（无需任何设置即可浏览）：
- [openai/skills](https://github.com/openai/skills)
- [anthropics/skills](https://github.com/anthropics/skills)
- [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills)
- [garrytan/gstack](https://github.com/garrytan/gstack)

- 示例：

```bash
hermes skills install openai/skills/k8s
hermes skills tap add myorg/skills-repo
```

#### 5. ClawHub (`clawhub`)

作为社区源集成的第三方技能市场。

- 站点：[clawhub.ai](https://clawhub.ai/)
- Hermes 源 ID：`clawhub`

#### 6. Claude 市场风格仓库 (`claude-marketplace`)

Hermes 支持发布 Claude 兼容的插件/市场清单的市场仓库。

已知的集成源包括：
- [anthropics/skills](https://github.com/anthropics/skills)
- [aiskillstore/marketplace](https://github.com/aiskillstore/marketplace)

Hermes 源 ID：`claude-marketplace`

#### 7. LobeHub (`lobehub`)

Hermes 可以搜索 LobeHub 公共目录中的智能体条目并将其转换为可安装的 Hermes 技能。

- 站点：[LobeHub](https://lobehub.com/)
- 公共智能体索引：[chat-agents.lobehub.com](https://chat-agents.lobehub.com/)
- 支持仓库：[lobehub/lobe-chat-agents](https://github.com/lobehub/lobe-chat-agents)
- Hermes 源 ID：`lobehub`

### 安全扫描和 `--force`

所有从中心安装的技能都要经过**安全扫描器**检查，以检测数据泄露、提示注入、破坏性命令、供应链信号和其他威胁。

`hermes skills inspect ...` 现在还会在可用时显示上游元数据：
- 仓库 URL
- skills.sh 详细页面 URL
- 安装命令
- 每周安装量
- 上游安全审计状态
- 知名索引/端点 URL

当您已审核第三方技能并希望覆盖非危险的策略阻止时，使用 `--force`：

```bash
hermes skills install skills-sh/anthropics/skills/pdf --force
```

重要行为：
- `--force` 可以覆盖警告/提示风格发现的策略阻止。
- `--force` **不会**覆盖 `dangerous` 扫描 verdict。
- 官方可选技能 (`official/...`) 被视为内置信任，不会显示第三方警告面板。

### 信任级别

| 级别 | 来源 | 策略 |
|-------|--------|--------|
| `builtin` | 随 Hermes 一起提供 | 始终受信任 |
| `official` | 仓库中的 `optional-skills/` | 内置信任，无第三方警告 |
| `trusted` | 受信任的注册表/仓库，如 `openai/skills`、`anthropics/skills` | 比社区源更宽松的策略 |
| `community` | 其他所有来源（`skills.sh`、知名端点、自定义 GitHub 仓库、大多数市场） | 非危险发现可以通过 `--force` 覆盖；`dangerous` 判定保持阻止 |

### 更新生命周期

中心现在跟踪足够的来源信息，可以重新检查已安装技能的上游副本：

```bash
hermes skills check          # 报告哪些已安装的中心技能在 upstream 发生了变化
hermes skills update         # 仅重新安装有可用更新的技能
hermes skills update react   # 更新一个特定的已安装中心技能
```

这使用存储的源标识符加上当前上游包内容哈希来检测漂移。

:::tip GitHub 速率限制
技能中心操作使用 GitHub API，未认证用户的速率限制为每小时 60 个请求。如果您在安装或搜索期间看到速率限制错误，请在 `.env` 文件中设置 `GITHUB_TOKEN` 以将限制增加到每小时 5,000 个请求。当发生这种情况时，错误消息会包含可操作的提示。
:::

### 斜杠命令（聊天内）

所有相同的命令都可以使用 `/skills`：

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

官方可选技能仍然使用类似 `official/security/1password` 和 `official/migration/openclaw-migration` 的标识符。