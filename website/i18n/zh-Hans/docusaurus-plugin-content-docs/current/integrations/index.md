---
title: "集成"
sidebar_label: "概述"
sidebar_position: 0
---

# 集成

Hermes Agent 连接到外部系统以进行 AI 推理、工具服务器、IDE 工作流、编程访问等。这些集成扩展了 Hermes 的功能和运行位置。

## AI 提供商与路由

Hermes 开箱即用支持多个 AI 推理提供商。使用 `hermes model` 进行交互式配置，或在 `config.yaml` 中设置它们。

- **[AI 提供商](/docs/user-guide/features/provider-routing)** — OpenRouter、Anthropic、OpenAI、Google 以及任何 OpenAI 兼容端点。Hermes 自动检测每个提供商的功能，如视觉、流式传输和工具使用。
- **[提供商路由](/docs/user-guide/features/provider-routing)** — 对哪些底层提供商处理您的 OpenRouter 请求进行细粒度控制。通过排序、白名单、黑名单和显式优先级排序来优化成本、速度或质量。
- **[备用提供商](/docs/user-guide/features/fallback-providers)** — 当您的主要模型遇到错误时，自动故障转移到备用 LLM 提供商。包括主要模型故障转移和独立的辅助任务故障转移，用于视觉、压缩和网络提取。

## 工具服务器 (MCP)

- **[MCP 服务器](/docs/user-guide/features/mcp)** — 通过模型上下文协议将 Hermes 连接到外部工具服务器。访问来自 GitHub、数据库、文件系统、浏览器堆栈、内部 API 等的工具，而无需编写原生 Hermes 工具。支持 stdio 和 SSE 传输、每服务器工具过滤以及能力感知的资源/提示注册。

## 网络搜索后端

`web_search` 和 `web_extract` 工具支持四个后端提供商，通过 `config.yaml` 或 `hermes tools` 配置：

| 后端 | 环境变量 | 搜索 | 提取 | 爬取 |
|---------|---------|--------|---------|-------|
| **Firecrawl** (默认) | `FIRECRAWL_API_KEY` | ✔ | ✔ | ✔ |
| **Parallel** | `PARALLEL_API_KEY` | ✔ | ✔ | — |
| **Tavily** | `TAVILY_API_KEY` | ✔ | ✔ | ✔ |
| **Exa** | `EXA_API_KEY` | ✔ | ✔ | — |

快速设置示例：

```yaml
web:
  backend: firecrawl    # firecrawl | parallel | tavily | exa
```

如果未设置 `web.backend`，则后端会根据可用的 API 密钥自动检测。自托管的 Firecrawl 也通过 `FIRECRAWL_API_URL` 支持。

## 浏览器自动化

Hermes 包含完整的浏览器自动化，具有多个后端选项，用于导航网站、填写表单和提取信息：

- **Browserbase** — 托管云浏览器，具有反机器人工具、CAPTCHA 解决和住宅代理
- **Browser Use** — 替代云浏览器提供商
- **通过 CDP 的本地 Chrome** — 使用 `/browser connect` 连接到您正在运行的 Chrome 实例
- **本地 Chromium** — 通过 `agent-browser` CLI 的无头本地浏览器

See [Browser Automation](/docs/user-guide/features/browser) for setup and usage.

## Voice & TTS Providers

Text-to-speech and speech-to-text across all messaging platforms:

| Provider | Quality | Cost | API Key |
||----------|---------|------|---------|
|| **Edge TTS** (default) | Good | Free | None needed |
|| **ElevenLabs** | Excellent | Paid | `ELEVENLABS_API_KEY` |
|| **OpenAI TTS** | Good | Paid | `VOICE_TOOLS_OPENAI_KEY` |
|| **MiniMax** | Good | Paid | `MINIMAX_API_KEY` |
|| **NeuTTS** | Good | Free | None needed |

Speech-to-text supports three providers: local Whisper (free, runs on-device), Groq (fast cloud), and OpenAI Whisper API. Voice message transcription works across Telegram, Discord, WhatsApp, and other messaging platforms. See [Voice & TTS](/docs/user-guide/features/tts) and [Voice Mode](/docs/user-guide/features/voice-mode) for details.

## IDE & Editor Integration

- **[IDE Integration (ACP)](/docs/user-guide/features/acp)** — Use Hermes Agent inside ACP-compatible editors such as VS Code, Zed, and JetBrains. Hermes runs as an ACP server, rendering chat messages, tool activity, file diffs, and terminal commands inside your editor.

## Programmatic Access

- **[API Server](/docs/user-guide/features/api-server)** — Expose Hermes as an OpenAI-compatible HTTP endpoint. Any frontend that speaks the OpenAI format — Open WebUI, LobeChat, LibreChat, NextChat, ChatBox — can connect and use Hermes as a backend with its full toolset.

## Memory & Personalization

- **[Built-in Memory](/docs/user-guide/features/memory)** — Persistent, curated memory via `MEMORY.md` and `USER.md` files. The agent maintains bounded stores of personal notes and user profile data that survive across sessions.
- **[Memory Providers](/docs/user-guide/features/memory-providers)** — Plug in external memory backends for deeper personalization. Seven providers are supported: Honcho (dialectic reasoning), OpenViking (tiered retrieval), Mem0 (cloud extraction), Hindsight (knowledge graphs), Holographic (local SQLite), RetainDB (hybrid search), and ByteRover (CLI-based).

## Messaging Platforms

Hermes runs as a gateway bot on 15+ messaging platforms, all configured through the same `gateway` subsystem:

- **[Telegram](/docs/user-guide/messaging/telegram)**, **[Discord](/docs/user-guide/messaging/discord)**, **[Slack](/docs/user-guide/messaging/slack)**, **[WhatsApp](/docs/user-guide/messaging/whatsapp)**, **[Signal](/docs/user-guide/messaging/signal)**, **[Matrix](/docs/user-guide/messaging/matrix)**, **[Mattermost](/docs/user-guide/messaging/mattermost)**, **[Email](/docs/user-guide/messaging/email)**, **[SMS](/docs/user-guide/messaging/sms)**, **[DingTalk](/docs/user-guide/messaging/dingtalk)**, **[Feishu/Lark](/docs/user-guide/messaging/feishu)**, **[WeCom](/docs/user-guide/messaging/wecom)**, **[WeCom Callback](/docs/user-guide/messaging/wecom-callback)**, **[Weixin](/docs/user-guide/messaging/weixin)**, **[BlueBubbles](/docs/user-guide/messaging/bluebubbles)**, **[QQ Bot](/docs/user-guide/messaging/qqbot)**, **[Home Assistant](/docs/user-guide/messaging/homeassistant)**, **[Webhooks](/docs/user-guide/messaging/webhooks)**

See the [Messaging Gateway overview](/docs/user-guide/messaging) for the platform comparison table and setup guide.

## Home Automation

- **[Home Assistant](/docs/user-guide/messaging/homeassistant)** — Control smart home devices via four dedicated tools (`ha_list_entities`, `ha_get_state`, `ha_list_services`, `ha_call_service`). The Home Assistant toolset activates automatically when `HASS_TOKEN` is configured.

## Plugins

- **[Plugin System](/docs/user-guide/features/plugins)** — Extend Hermes with custom tools, lifecycle hooks, and CLI commands without modifying core code. Plugins are discovered from `~/.hermes/plugins/`, project-local `.hermes/plugins/`, and pip-installed entry points.
- **[Build a Plugin](/docs/guides/build-a-hermes-plugin)** — Step-by-step guide for creating Hermes plugins with tools, hooks, and CLI commands.

## Training & Evaluation

- **[RL Training](/docs/user-guide/features/rl-training)** — Generate trajectory data from agent sessions for reinforcement learning and model fine-tuning. Supports Atropos environments with customizable reward functions.
- **[Batch Processing](/docs/user-guide/features/batch-processing)** — Run the agent across hundreds of prompts in parallel, generating structured ShareGPT-format trajectory data for training data generation or evaluation.