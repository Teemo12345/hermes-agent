---
sidebar_position: 9
title: "可选技能目录"
description: "随 hermes-agent 提供的官方可选技能 — 通过 hermes skills install official/<category>/<skill> 安装"
---

# 可选技能目录

官方可选技能随 hermes-agent 仓库提供，位于 `optional-skills/` 下，但**默认不激活**。请显式安装它们：

```bash
hermes skills install official/<category>/<skill>
```

例如：

```bash
hermes skills install official/blockchain/solana
hermes skills install official/mlops/flash-attention
```

安装后，技能会出现在智能体的技能列表中，并可以在检测到相关任务时自动加载。

要卸载：

```bash
hermes skills uninstall <skill-name>
```

---

## 自主 AI 智能体

| 技能 | 描述 |
|-------|-------------|
| **blackbox** | 将编码任务委托给 Blackbox AI CLI 智能体。具有内置评判器的多模型智能体，通过多个 LLM 运行任务并选择最佳结果。 |
| **honcho** | 配置和使用 Honcho 内存与 Hermes — 跨会话用户建模、多配置文件对等隔离、观察配置和辩证推理。 |

## 区块链

| 技能 | 描述 |
|-------|-------------|
| **base** | 查询 Base（以太坊 L2）区块链数据，带美元定价 — 钱包余额、代币信息、交易详情、燃气分析、合约检查、鲸鱼检测和实时网络统计。无需 API 密钥。 |
| **solana** | 查询 Solana 区块链数据，带美元定价 — 钱包余额、代币组合、交易详情、NFT、鲸鱼检测和实时网络统计。无需 API 密钥。 |

## 通信

| 技能 | 描述 |
|-------|-------------|
| **one-three-one-rule** | 用于提案和决策的结构化通信框架。 |

## Creative

| Skill | Description |
|-------|-------------|
| **blender-mcp** | Control Blender directly from Hermes via socket connection to the blender-mcp addon. Create 3D objects, materials, animations, and run arbitrary Blender Python (bpy) code. |
| **meme-generation** | Generate real meme images by picking a template and overlaying text with Pillow. Produces actual `.png` meme files. |

## DevOps

| Skill | Description |
|-------|-------------|
| **cli** | Run 150+ AI apps via inference.sh CLI (infsh) — image generation, video creation, LLMs, search, 3D, and social automation. |
| **docker-management** | Manage Docker containers, images, volumes, networks, and Compose stacks — lifecycle ops, debugging, cleanup, and Dockerfile optimization. |

## Email

| Skill | Description |
|-------|-------------|
| **agentmail** | Give the agent its own dedicated email inbox via AgentMail. Send, receive, and manage email autonomously using agent-owned email addresses. |

## Health

| Skill | Description |
|-------|-------------|
| **neuroskill-bci** | Brain-Computer Interface (BCI) integration for neuroscience research workflows. |

## MCP

| Skill | Description |
|-------|-------------|
| **fastmcp** | Build, test, inspect, install, and deploy MCP servers with FastMCP in Python. Covers wrapping APIs or databases as MCP tools, exposing resources or prompts, and deployment. |

## Migration

| Skill | Description |
|-------|-------------|
| **openclaw-migration** | Migrate a user's OpenClaw customization footprint into Hermes Agent. Imports memories, SOUL.md, command allowlists, user skills, and selected workspace assets. |

## MLOps

The largest optional category — covers the full ML pipeline from data curation to production inference.

| Skill | Description |
|-------|-------------|
| **accelerate** | Simplest distributed training API. 4 lines to add distributed support to any PyTorch script. Unified API for DeepSpeed/FSDP/Megatron/DDP. |
| **chroma** | Open-source embedding database. Store embeddings and metadata, perform vector and full-text search. Simple 4-function API for RAG and semantic search. |
| **faiss** | Facebook's library for efficient similarity search and clustering of dense vectors. Supports billions of vectors, GPU acceleration, and various index types (Flat, IVF, HNSW). |
| **flash-attention** | Optimize transformer attention with Flash Attention for 2-4x speedup and 10-20x memory reduction. Supports PyTorch SDPA, flash-attn library, H100 FP8, and sliding window. |
| **hermes-atropos-environments** | Build, test, and debug Hermes Agent RL environments for Atropos training. Covers the HermesAgentBaseEnv interface, reward functions, agent loop integration, and evaluation. |
| **huggingface-tokenizers** | Fast Rust-based tokenizers for research and production. Tokenizes 1GB in under 20 seconds. Supports BPE, WordPiece, and Unigram algorithms. |
| **instructor** | Extract structured data from LLM responses with Pydantic validation, retry failed extractions automatically, and stream partial results. |
| **lambda-labs** | Reserved and on-demand GPU cloud instances for ML training and inference. SSH access, persistent filesystems, and multi-node clusters. |
| **llava** | Large Language and Vision Assistant — visual instruction tuning and image-based conversations combining CLIP vision with LLaMA language models. |
| **nemo-curator** | GPU-accelerated data curation for LLM training. Fuzzy deduplication (16x faster), quality filtering (30+ heuristics), semantic dedup, PII redaction. Scales with RAPIDS. |
| **pinecone** | Managed vector database for production AI. Auto-scaling, hybrid search (dense + sparse), metadata filtering, and low latency (under 100ms p95). |
| **pytorch-lightning** | High-level PyTorch framework with Trainer class, automatic distributed training (DDP/FSDP/DeepSpeed), callbacks, and minimal boilerplate. |
| **qdrant** | High-performance vector similarity search engine. Rust-powered with fast nearest neighbor search, hybrid search with filtering, and scalable vector storage. |
| **saelens** | Train and analyze Sparse Autoencoders (SAEs) using SAELens to decompose neural network activations into interpretable features. |
| **simpo** | Simple Preference Optimization — reference-free alternative to DPO with better performance (+6.4 pts on AlpacaEval 2.0). No reference model needed. |
| **slime** | LLM post-training with RL using Megatron+SGLang framework. Custom data generation workflows and tight Megatron-LM integration for RL scaling. |
| **tensorrt-llm** | Optimize LLM inference with NVIDIA TensorRT for maximum throughput. 10-100x faster than PyTorch on A100/H100 with quantization (FP8/INT4) and in-flight batching. |
| **torchtitan** | PyTorch-native distributed LLM pretraining with 4D parallelism (FSDP2, TP, PP, CP). Scale from 8 to 512+ GPUs with Float8 and torch.compile. |

## Productivity

| Skill | Description |
|-------|-------------|
| **canvas** | Canvas LMS integration — fetch enrolled courses and assignments using API token authentication. |
| **memento-flashcards** | Spaced repetition flashcard system for learning and knowledge retention. |
| **siyuan** | SiYuan Note API for searching, reading, creating, and managing blocks and documents in a self-hosted knowledge base. |
| **telephony** | Give Hermes phone capabilities — provision a Twilio number, send/receive SMS/MMS, make calls, and place AI-driven outbound calls through Bland.ai or Vapi. |

## Research

| Skill | Description |
|-------|-------------|
| **bioinformatics** | Gateway to 400+ bioinformatics skills from bioSkills and ClawBio. Covers genomics, transcriptomics, single-cell, variant calling, pharmacogenomics, metagenomics, and structural biology. |
| **domain-intel** | Passive domain reconnaissance using Python stdlib. Subdomain discovery, SSL certificate inspection, WHOIS lookups, DNS records, and bulk multi-domain analysis. No API keys required. |
| **duckduckgo-search** | Free web search via DuckDuckGo — text, news, images, videos. No API key needed. |
| **gitnexus-explorer** | Index a codebase with GitNexus and serve an interactive knowledge graph via web UI and Cloudflare tunnel. |
| **parallel-cli** | Vendor skill for Parallel CLI — agent-native web search, extraction, deep research, enrichment, and monitoring. |
| **qmd** | Search personal knowledge bases, notes, docs, and meeting transcripts locally using qmd — a hybrid retrieval engine with BM25, vector search, and LLM reranking. |
| **scrapling** | Web scraping with Scrapling — HTTP fetching, stealth browser automation, Cloudflare bypass, and spider crawling via CLI and Python. |

## Security

| Skill | Description |
|-------|-------------|
| **1password** | Set up and use 1Password CLI (op). Install the CLI, enable desktop app integration, sign in, and read/inject secrets for commands. |
| **oss-forensics** | Open-source software forensics — analyze packages, dependencies, and supply chain risks. |
| **sherlock** | OSINT username search across 400+ social networks. Hunt down social media accounts by username. |

---

## Contributing Optional Skills

To add a new optional skill to the repository:

1. Create a directory under `optional-skills/<category>/<skill-name>/`
2. Add a `SKILL.md` with standard frontmatter (name, description, version, author)
3. Include any supporting files in `references/`, `templates/`, or `scripts/` subdirectories
4. Submit a pull request — the skill will appear in this catalog once merged