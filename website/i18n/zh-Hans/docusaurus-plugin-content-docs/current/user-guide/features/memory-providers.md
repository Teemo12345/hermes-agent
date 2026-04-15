---
sidebar_position: 4
title: "记忆提供者"
description: "外部记忆提供者插件 — Honcho、OpenViking、Mem0、Hindsight、Holographic、RetainDB、ByteRover、Supermemory"
---

# 记忆提供者

Hermes Agent 附带 8 个外部记忆提供者插件，为智能体提供超越内置 MEMORY.md 和 USER.md 的持久、跨会话知识。一次只能激活**一个**外部提供者 — 内置记忆始终与它一起激活。

## 快速开始

```bash
hermes memory setup      # 交互式选择器 + 配置
hermes memory status     # 检查当前激活的提供者
hermes memory off        # 禁用外部提供者
```

您也可以通过 `hermes plugins` → Provider Plugins → Memory Provider 选择激活的记忆提供者。

或在 `~/.hermes/config.yaml` 中手动设置：

```yaml
memory:
  provider: openviking   # 或 honcho, mem0, hindsight, holographic, retaindb, byterover, supermemory
```

## 工作原理

当记忆提供者激活时，Hermes 会自动：

1. **注入提供者上下文**到系统提示中（提供者所知道的内容）
2. **预获取相关记忆**在每个回合之前（后台，非阻塞）
3. **同步对话回合**到提供者在每个响应之后
4. **在会话结束时提取记忆**（对于支持此功能的提供者）
5. **镜像内置记忆写入**到外部提供者
6. **添加提供者特定工具**以便智能体可以搜索、存储和管理记忆

内置记忆（MEMORY.md / USER.md）继续按原样工作。外部提供者是附加的。

## 可用提供者

### Honcho

AI 原生跨会话用户建模，具有辩证问答、语义搜索和持久结论。

| | |
|---|---|
| **最适合** | 具有跨会话上下文、用户-智能体对齐的多智能体系统 |
| **要求** | `pip install honcho-ai` + [API 密钥](https://app.honcho.dev) 或自托管实例 |
| **数据存储** | Honcho Cloud 或自托管 |
| **成本** | Honcho 定价（云）/ 免费（自托管） |

**工具：** `honcho_profile`（对等卡）、`honcho_search`（语义搜索）、`honcho_context`（LLM 合成）、`honcho_conclude`（存储事实）

**设置向导：**
```bash
hermes honcho setup        # (传统命令) 
# 或
hermes memory setup        # 选择 "honcho"
```

**配置：** `$HERMES_HOME/honcho.json`（配置文件本地）或 `~/.honcho/config.json`（全局）。解析顺序：`$HERMES_HOME/honcho.json` > `~/.hermes/honcho.json` > `~/.honcho/config.json`。参见 [配置参考](https://github.com/hermes-ai/hermes-agent/blob/main/plugins/memory/honcho/README.md) 和 [Honcho 集成指南](https://docs.honcho.dev/v3/guides/integrations/hermes)。

<details>
<summary>关键配置选项</summary>

| 键 | 默认值 | 描述 |
|-----|---------|-------------|
| `apiKey` | -- | 来自 [app.honcho.dev](https://app.honcho.dev) 的 API 密钥 |
| `baseUrl` | -- | 自托管 Honcho 的基础 URL |
| `peerName` | -- | 用户对等身份 |
| `aiPeer` | 主机键 | AI 对等身份（每个配置文件一个） |
| `workspace` | 主机键 | 共享工作区 ID |
| `recallMode` | `hybrid` | `hybrid`（自动注入 + 工具）、`context`（仅注入）、`tools`（仅工具） |
| `observation` | 全部开启 | 每个对等体的 `observeMe`/`observeOthers` 布尔值 |
| `writeFrequency` | `async` | `async`、`turn`、`session` 或整数 N |
| `sessionStrategy` | `per-directory` | `per-directory`、`per-repo`、`per-session`、`global` |
| `dialecticReasoningLevel` | `low` | `minimal`、`low`、`medium`、`high`、`max` |
| `dialecticDynamic` | `true` | 按查询长度自动提高推理级别 |
| `messageMaxChars` | `25000` | 每条消息的最大字符数（超过则分块） |

</details>

<details>
<summary>最小 honcho.json（云）</summary>

```json
{
  "apiKey": "your-key-from-app.honcho.dev",
  "hosts": {
    "hermes": {
      "enabled": true,
      "aiPeer": "hermes",
      "peerName": "your-name",
      "workspace": "hermes"
    }
  }
}
```

</details>

<details>
<summary>最小 honcho.json（自托管）</summary>

```json
{
  "baseUrl": "http://localhost:8000",
  "hosts": {
    "hermes": {
      "enabled": true,
      "aiPeer": "hermes",
      "peerName": "your-name",
      "workspace": "hermes"
    }
  }
}
```

</details>

:::tip 从 `hermes honcho` 迁移
如果您之前使用过 `hermes honcho setup`，您的配置和所有服务器端数据都保持不变。只需再次通过设置向导重新启用，或手动设置 `memory.provider: honcho` 以通过新系统重新激活。
:::

**多智能体 / 配置文件：**

每个 Hermes 配置文件都有自己的 Honcho AI 对等体，同时共享同一个工作区 — 所有配置文件看到相同的用户表示，但每个智能体构建自己的身份和观察。

```bash
hermes profile create coder --clone   # 创建 honcho 对等体 "coder"，从默认继承配置
```

`--clone` 的作用：在 `honcho.json` 中创建一个 `hermes.coder` 主机块，包含 `aiPeer: "coder"`、共享的 `workspace`、继承的 `peerName`、`recallMode`、`writeFrequency`、`observation` 等。该对等体在 Honcho 中被急切创建，因此在第一条消息之前就存在。

对于在 Honcho 设置之前创建的配置文件：

```bash
hermes honcho sync   # 扫描所有配置文件，为任何缺失的配置文件创建主机块
```

这会从默认的 `hermes` 主机块继承设置，并为每个配置文件创建新的 AI 对等体。幂等操作 — 跳过已经有主机块的配置文件。

<details>
<summary>完整 honcho.json 示例（多配置文件）</summary>

```json
{
  "apiKey": "your-key",
  "workspace": "hermes",
  "peerName": "eri",
  "hosts": {
    "hermes": {
      "enabled": true,
      "aiPeer": "hermes",
      "workspace": "hermes",
      "peerName": "eri",
      "recallMode": "hybrid",
      "writeFrequency": "async",
      "sessionStrategy": "per-directory",
      "observation": {
        "user": { "observeMe": true, "observeOthers": true },
        "ai": { "observeMe": true, "observeOthers": true }
      },
      "dialecticReasoningLevel": "low",
      "dialecticDynamic": true,
      "dialecticMaxChars": 600,
      "messageMaxChars": 25000,
      "saveMessages": true
    },
    "hermes.coder": {
      "enabled": true,
      "aiPeer": "coder",
      "workspace": "hermes",
      "peerName": "eri",
      "recallMode": "tools",
      "observation": {
        "user": { "observeMe": true, "observeOthers": false },
        "ai": { "observeMe": true, "observeOthers": true }
      }
    },
    "hermes.writer": {
      "enabled": true,
      "aiPeer": "writer",
      "workspace": "hermes",
      "peerName": "eri"
    }
  },
  "sessions": {
    "/home/user/myproject": "myproject-main"
  }
}
```

</details>

参见 [配置参考](https://github.com/hermes-ai/hermes-agent/blob/main/plugins/memory/honcho/README.md) 和 [Honcho 集成指南](https://docs.honcho.dev/v3/guides/integrations/hermes)。


---

### OpenViking

Context database by Volcengine (ByteDance) with filesystem-style knowledge hierarchy, tiered retrieval, and automatic memory extraction into 6 categories.

| | |
|---|---|
| **Best for** | Self-hosted knowledge management with structured browsing |
| **Requires** | `pip install openviking` + running server |
| **Data storage** | Self-hosted (local or cloud) |
| **Cost** | Free (open-source, AGPL-3.0) |

**Tools:** `viking_search` (semantic search), `viking_read` (tiered: abstract/overview/full), `viking_browse` (filesystem navigation), `viking_remember` (store facts), `viking_add_resource` (ingest URLs/docs)

**Setup:**
```bash
# Start the OpenViking server first
pip install openviking
openviking-server

# Then configure Hermes
hermes memory setup    # select "openviking"
# Or manually:
hermes config set memory.provider openviking
echo "OPENVIKING_ENDPOINT=http://localhost:1933" >> ~/.hermes/.env
```

**Key features:**
- Tiered context loading: L0 (~100 tokens) → L1 (~2k) → L2 (full)
- Automatic memory extraction on session commit (profile, preferences, entities, events, cases, patterns)
- `viking://` URI scheme for hierarchical knowledge browsing

---

### Mem0

Server-side LLM fact extraction with semantic search, reranking, and automatic deduplication.

| | |
|---|---|
| **Best for** | Hands-off memory management — Mem0 handles extraction automatically |
| **Requires** | `pip install mem0ai` + API key |
| **Data storage** | Mem0 Cloud |
| **Cost** | Mem0 pricing |

**Tools:** `mem0_profile` (all stored memories), `mem0_search` (semantic search + reranking), `mem0_conclude` (store verbatim facts)

**Setup:**
```bash
hermes memory setup    # select "mem0"
# Or manually:
hermes config set memory.provider mem0
echo "MEM0_API_KEY=your-key" >> ~/.hermes/.env
```

**Config:** `$HERMES_HOME/mem0.json`

| Key | Default | Description |
|-----|---------|-------------|
| `user_id` | `hermes-user` | User identifier |
| `agent_id` | `hermes` | Agent identifier |

---

### Hindsight

Long-term memory with knowledge graph, entity resolution, and multi-strategy retrieval. The `hindsight_reflect` tool provides cross-memory synthesis that no other provider offers. Automatically retains full conversation turns (including tool calls) with session-level document tracking.

| | |
|---|---|
| **Best for** | Knowledge graph-based recall with entity relationships |
| **Requires** | Cloud: API key from [ui.hindsight.vectorize.io](https://ui.hindsight.vectorize.io). Local: LLM API key (OpenAI, Groq, OpenRouter, etc.) |
| **Data storage** | Hindsight Cloud or local embedded PostgreSQL |
| **Cost** | Hindsight pricing (cloud) or free (local) |

**Tools:** `hindsight_retain` (store with entity extraction), `hindsight_recall` (multi-strategy search), `hindsight_reflect` (cross-memory synthesis)

**Setup:**
```bash
hermes memory setup    # select "hindsight"
# Or manually:
hermes config set memory.provider hindsight
echo "HINDSIGHT_API_KEY=your-key" >> ~/.hermes/.env
```

The setup wizard installs dependencies automatically and only installs what's needed for the selected mode (`hindsight-client` for cloud, `hindsight-all` for local). Requires `hindsight-client >= 0.4.22` (auto-upgraded on session start if outdated).

**Local mode UI:** `hindsight-embed -p hermes ui start`

**Config:** `$HERMES_HOME/hindsight/config.json`

| Key | Default | Description |
|-----|---------|-------------|
| `mode` | `cloud` | `cloud` or `local` |
| `bank_id` | `hermes` | Memory bank identifier |
| `recall_budget` | `mid` | Recall thoroughness: `low` / `mid` / `high` |
| `memory_mode` | `hybrid` | `hybrid` (context + tools), `context` (auto-inject only), `tools` (tools only) |
| `auto_retain` | `true` | Automatically retain conversation turns |
| `auto_recall` | `true` | Automatically recall memories before each turn |
| `retain_async` | `true` | Process retain asynchronously on the server |
| `tags` | — | Tags applied when storing memories |
| `recall_tags` | — | Tags to filter on recall |

See [plugin README](https://github.com/NousResearch/hermes-agent/blob/main/plugins/memory/hindsight/README.md) for the full configuration reference.

---

### Holographic

Local SQLite fact store with FTS5 full-text search, trust scoring, and HRR (Holographic Reduced Representations) for compositional algebraic queries.

| | |
|---|---|
| **Best for** | Local-only memory with advanced retrieval, no external dependencies |
| **Requires** | Nothing (SQLite is always available). NumPy optional for HRR algebra. |
| **Data storage** | Local SQLite |
| **Cost** | Free |

**Tools:** `fact_store` (9 actions: add, search, probe, related, reason, contradict, update, remove, list), `fact_feedback` (helpful/unhelpful rating that trains trust scores)

**Setup:**
```bash
hermes memory setup    # select "holographic"
# Or manually:
hermes config set memory.provider holographic
```

**Config:** `config.yaml` under `plugins.hermes-memory-store`

| Key | Default | Description |
|-----|---------|-------------|
| `db_path` | `$HERMES_HOME/memory_store.db` | SQLite database path |
| `auto_extract` | `false` | Auto-extract facts at session end |
| `default_trust` | `0.5` | Default trust score (0.0–1.0) |

**Unique capabilities:**
- `probe` — entity-specific algebraic recall (all facts about a person/thing)
- `reason` — compositional AND queries across multiple entities
- `contradict` — automated detection of conflicting facts
- Trust scoring with asymmetric feedback (+0.05 helpful / -0.10 unhelpful)

---

### RetainDB

Cloud memory API with hybrid search (Vector + BM25 + Reranking), 7 memory types, and delta compression.

| | |
|---|---|
| **Best for** | Teams already using RetainDB's infrastructure |
| **Requires** | RetainDB account + API key |
| **Data storage** | RetainDB Cloud |
| **Cost** | $20/month |

**Tools:** `retaindb_profile` (user profile), `retaindb_search` (semantic search), `retaindb_context` (task-relevant context), `retaindb_remember` (store with type + importance), `retaindb_forget` (delete memories)

**Setup:**
```bash
hermes memory setup    # select "retaindb"
# Or manually:
hermes config set memory.provider retaindb
echo "RETAINDB_API_KEY=your-key" >> ~/.hermes/.env
```

---

### ByteRover

Persistent memory via the `brv` CLI — hierarchical knowledge tree with tiered retrieval (fuzzy text → LLM-driven search). Local-first with optional cloud sync.

| | |
|---|---|
| **Best for** | Developers who want portable, local-first memory with a CLI |
| **Requires** | ByteRover CLI (`npm install -g byterover-cli` or [install script](https://byterover.dev)) |
| **Data storage** | Local (default) or ByteRover Cloud (optional sync) |
| **Cost** | Free (local) or ByteRover pricing (cloud) |

**Tools:** `brv_query` (search knowledge tree), `brv_curate` (store facts/decisions/patterns), `brv_status` (CLI version + tree stats)

**Setup:**
```bash
# Install the CLI first
curl -fsSL https://byterover.dev/install.sh | sh

# Then configure Hermes
hermes memory setup    # select "byterover"
# Or manually:
hermes config set memory.provider byterover
```

**Key features:**
- Automatic pre-compression extraction (saves insights before context compression discards them)
- Knowledge tree stored at `$HERMES_HOME/byterover/` (profile-scoped)
- SOC2 Type II certified cloud sync (optional)

---

### Supermemory

Semantic long-term memory with profile recall, semantic search, explicit memory tools, and session-end conversation ingest via the Supermemory graph API.

| | |
|---|---|
| **Best for** | Semantic recall with user profiling and session-level graph building |
| **Requires** | `pip install supermemory` + [API key](https://supermemory.ai) |
| **Data storage** | Supermemory Cloud |
| **Cost** | Supermemory pricing |

**Tools:** `supermemory_store` (save explicit memories), `supermemory_search` (semantic similarity search), `supermemory_forget` (forget by ID or best-match query), `supermemory_profile` (persistent profile + recent context)

**Setup:**
```bash
hermes memory setup    # select "supermemory"
# Or manually:
hermes config set memory.provider supermemory
echo 'SUPERMEMORY_API_KEY=***' >> ~/.hermes/.env
```

**Config:** `$HERMES_HOME/supermemory.json`

| Key | Default | Description |
|-----|---------|-------------|
| `container_tag` | `hermes` | Container tag used for search and writes. Supports `{identity}` template for profile-scoped tags. |
| `auto_recall` | `true` | Inject relevant memory context before turns |
| `auto_capture` | `true` | Store cleaned user-assistant turns after each response |
| `max_recall_results` | `10` | Max recalled items to format into context |
| `profile_frequency` | `50` | Include profile facts on first turn and every N turns |
| `capture_mode` | `all` | Skip tiny or trivial turns by default |
| `search_mode` | `hybrid` | Search mode: `hybrid`, `memories`, or `documents` |
| `api_timeout` | `5.0` | Timeout for SDK and ingest requests |

**Environment variables:** `SUPERMEMORY_API_KEY` (required), `SUPERMEMORY_CONTAINER_TAG` (overrides config).

**Key features:**
- Automatic context fencing — strips recalled memories from captured turns to prevent recursive memory pollution
- Session-end conversation ingest for richer graph-level knowledge building
- Profile facts injected on first turn and at configurable intervals
- Trivial message filtering (skips "ok", "thanks", etc.)
- **Profile-scoped containers** — use `{identity}` in `container_tag` (e.g. `hermes-{identity}` → `hermes-coder`) to isolate memories per Hermes profile
- **Multi-container mode** — enable `enable_custom_container_tags` with a `custom_containers` list to let the agent read/write across named containers. Automatic operations (sync, prefetch) stay on the primary container.

<details>
<summary>Multi-container example</summary>

```json
{
  "container_tag": "hermes",
  "enable_custom_container_tags": true,
  "custom_containers": ["project-alpha", "shared-knowledge"],
  "custom_container_instructions": "Use project-alpha for coding context."
}
```

</details>

**Support:** [Discord](https://supermemory.link/discord) · [support@supermemory.com](mailto:support@supermemory.com)

---

## Provider Comparison

| Provider | Storage | Cost | Tools | Dependencies | Unique Feature |
|----------|---------|------|-------|-------------|----------------|
| **Honcho** | Cloud | Paid | 4 | `honcho-ai` | Dialectic user modeling |
| **OpenViking** | Self-hosted | Free | 5 | `openviking` + server | Filesystem hierarchy + tiered loading |
| **Mem0** | Cloud | Paid | 3 | `mem0ai` | Server-side LLM extraction |
| **Hindsight** | Cloud/Local | Free/Paid | 3 | `hindsight-client` | Knowledge graph + reflect synthesis |
| **Holographic** | Local | Free | 2 | None | HRR algebra + trust scoring |
| **RetainDB** | Cloud | $20/mo | 5 | `requests` | Delta compression |
| **ByteRover** | Local/Cloud | Free/Paid | 3 | `brv` CLI | Pre-compression extraction |
| **Supermemory** | Cloud | Paid | 4 | `supermemory` | Context fencing + session graph ingest + multi-container |

## Profile Isolation

Each provider's data is isolated per [profile](/docs/user-guide/profiles):

- **Local storage providers** (Holographic, ByteRover) use `$HERMES_HOME/` paths which differ per profile
- **Config file providers** (Honcho, Mem0, Hindsight, Supermemory) store config in `$HERMES_HOME/` so each profile has its own credentials
- **Cloud providers** (RetainDB) auto-derive profile-scoped project names
- **Env var providers** (OpenViking) are configured via each profile's `.env` file

## Building a Memory Provider

See the [Developer Guide: Memory Provider Plugins](/docs/developer-guide/memory-provider-plugin) for how to create your own.
