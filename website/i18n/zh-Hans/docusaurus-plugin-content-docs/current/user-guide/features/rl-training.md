---
sidebar_position: 13
title: "强化学习训练"
description: "使用 Tinker-Atropos 对智能体行为进行强化学习 — 环境发现、训练和评估"
---

# 强化学习训练

Hermes Agent 包含一个基于 **Tinker-Atropos** 构建的集成 RL（强化学习）训练管道。这使得可以使用带有 LoRA 适配器的 GRPO（组相对策略优化）在环境特定任务上训练语言模型，完全通过智能体的工具接口进行编排。

## 概述

RL 训练系统由三个组件组成：

1. **Atropos** — 一个轨迹 API 服务器，协调环境交互、管理展开组和计算优势
2. **Tinker** — 一个训练服务，处理模型权重、LoRA 训练、采样/推理和优化器步骤
3. **环境** — 定义任务、评分和奖励函数的 Python 类（例如，GSM8K 数学问题）

智能体可以发现环境、配置训练参数、启动训练运行和监控指标 — 所有这些都通过一组 `rl_*` 工具完成。

## 要求

RL 训练需要：

- **Python >= 3.11**（Tinker 包要求）
- **TINKER_API_KEY** — Tinker 训练服务的 API 密钥
- **WANDB_API_KEY** — Weights & Biases 指标跟踪的 API 密钥
- `tinker-atropos` 子模块（位于 Hermes 根目录的 `tinker-atropos/`）

```bash
# 设置 API 密钥
hermes config set TINKER_API_KEY your-tinker-key
hermes config set WANDB_API_KEY your-wandb-key
```

当两个密钥都存在且 Python >= 3.11 可用时，`rl` 工具集会自动启用。

## 可用工具

| 工具 | 描述 |
|------|-------------|
| `rl_list_environments` | 发现可用的 RL 环境 |
| `rl_select_environment` | 选择环境并加载其配置 |
| `rl_get_current_config` | 查看可配置和锁定的字段 |
| `rl_edit_config` | 修改可配置的训练参数 |
| `rl_start_training` | 启动训练运行（生成 3 个进程） |
| `rl_check_status` | 监控训练进度和 WandB 指标 |
| `rl_stop_training` | 停止运行中的训练作业 |
| `rl_get_results` | 获取最终指标和模型权重路径 |
| `rl_list_runs` | 列出所有活跃和已完成的运行 |
| `rl_test_inference` | 使用 OpenRouter 进行快速推理测试 |

## 工作流程

### 1. 发现环境

```
列出可用的 RL 环境
```

智能体调用 `rl_list_environments()`，它使用 AST 解析扫描 `tinker-atropos/tinker_atropos/environments/` 来查找继承自 `BaseEnv` 的 Python 类。每个环境定义：

- **数据集加载** — 训练数据的来源（例如，HuggingFace 数据集）
- **提示构建** — 如何为模型格式化项目
- **评分/验证** — 如何评估模型输出并分配奖励

### 2. 选择和配置

```
选择 GSM8K 环境并显示配置
```

智能体调用 `rl_select_environment("gsm8k_tinker")`，然后调用 `rl_get_current_config()` 查看所有参数。

配置字段分为两类：

**可配置字段**（可以修改）：
- `group_size` — 每个项目的完成次数（默认：16）
- `batch_size` — 训练批大小（默认：128）
- `wandb_name` — WandB 运行名称（自动设置为 `{env}-{timestamp}`）
- 其他环境特定参数

**锁定字段**（基础设施设置，不能更改）：
- `tokenizer_name` — 模型分词器（例如，`Qwen/Qwen3-8B`）
- `rollout_server_url` — Atropos API URL（`http://localhost:8000`）
- `max_token_length` — 最大令牌长度（8192）
- `max_num_workers` — 最大并行工作器（2048）
- `total_steps` — 总训练步骤（2500）
- `lora_rank` — LoRA 适配器秩（32）
- `learning_rate` — 学习率（4e-5）
- `max_token_trainer_length` — 训练器的最大令牌数（9000）

### 3. 开始训练

```
开始训练运行
```

智能体调用 `rl_start_training()`，它：

1. 生成一个 YAML 配置文件，合并锁定设置和可配置覆盖
2. 创建唯一的运行 ID
3. 生成三个进程：
   - **Atropos API 服务器** (`run-api`) — 轨迹协调
   - **Tinker 训练器** (`launch_training.py`) — LoRA 训练 + 端口 8001 上的 FastAPI 推理服务器
   - **环境** (`environment.py serve`) — 连接到 Atropos 的选定环境

进程以交错延迟启动（API 为 5 秒，训练器为 30 秒，环境再增加 90 秒），以确保正确的初始化顺序。

### 4. 监控进度

```
检查训练运行 abc12345 的状态
```

智能体调用 `rl_check_status(run_id)`，它会报告：

- 进程状态（3 个进程各自的运行/退出状态）
- 运行时间
- WandB 指标（步骤、奖励均值、正确率、评估准确率）
- 用于调试的日志文件位置

:::note 速率限制
状态检查的速率限制为每个运行 ID 每 **30 分钟** 一次。这可以防止在需要数小时的长时间训练作业期间过度轮询。
:::

### 5. 停止或获取结果

```
停止训练运行
# 或
获取运行 abc12345 的最终结果
```

`rl_stop_training()` 以相反的顺序终止所有三个进程（环境 → 训练器 → API）。`rl_get_results()` 检索最终的 WandB 指标和训练历史。

## 推理测试

在进行完整的训练运行之前，您可以使用 `rl_test_inference` 测试环境是否正常工作。这使用 OpenRouter 运行几个推理和评分步骤 — 不需要 Tinker API，只需要一个 `OPENROUTER_API_KEY`。

```
使用推理测试选定的环境
```

默认配置：
- **3 步 × 16 个完成 = 每个模型 48 次展开**
- 测试 3 个不同规模的模型以确保鲁棒性：
  - `qwen/qwen3-8b`（小型）
  - `z-ai/glm-4.7-flash`（中型）
  - `minimax/minimax-m2.7` (large)
- Total: ~144 rollouts

This validates:
- Environment loads correctly
- Prompt construction works
- Inference response parsing is robust across model scales
- Verifier/scoring logic produces valid rewards

## Tinker API Integration

The trainer uses the [Tinker](https://tinker.computer) API for model training operations:

- **ServiceClient** — Creates training and sampling clients
- **Training client** — Handles forward-backward passes with importance sampling loss, optimizer steps (Adam), and weight checkpointing
- **Sampling client** — Provides inference using the latest trained weights

The training loop:
1. Fetches a batch of rollouts from Atropos (prompt + completions + scores)
2. Converts to Tinker Datum objects with padded logprobs and advantages
3. Runs forward-backward pass with importance sampling loss
4. Takes an optimizer step (Adam: lr=4e-5, β1=0.9, β2=0.95)
5. Saves weights and creates a new sampling client for next-step inference
6. Logs metrics to WandB

## Architecture Diagram

```mermaid
flowchart LR
    api["Atropos API<br/>run-api<br/>port 8000"]
    env["Environment<br/>BaseEnv implementation"]
    infer["OpenAI / sglang<br/>inference API<br/>port 8001"]
    trainer["Tinker Trainer<br/>LoRA training + FastAPI"]

    env <--> api
    env --> infer
    api -->|"batches: tokens, scores, logprobs"| trainer
    trainer -->|"serves inference"| infer
```

## Creating Custom Environments

To create a new RL environment:

1. Create a Python file in `tinker-atropos/tinker_atropos/environments/`
2. Define a class that inherits from `BaseEnv`
3. Implement the required methods:
   - `load_dataset()` — Load your training data
   - `get_next_item()` — Provide the next item to the model
   - `score_answer()` — Score model outputs and assign rewards
   - `collect_trajectories()` — Collect and return trajectories
4. Optionally define a custom config class inheriting from `BaseEnvConfig`

Study the existing `gsm8k_tinker.py` as a template. The agent can help you create new environments — it can read existing environment files, inspect HuggingFace datasets, and write new environment code.

## WandB Metrics

Training runs log to Weights & Biases with these key metrics:

| Metric | Description |
|--------|-------------|
| `train/loss` | Training loss (importance sampling) |
| `train/learning_rate` | Current learning rate |
| `reward/mean` | Mean reward across groups |
| `logprobs/mean` | Mean reference logprobs |
| `logprobs/mean_training` | Mean training logprobs |
| `logprobs/diff` | Logprob drift (reference - training) |
| `advantages/mean` | Mean advantage values |
| `advantages/std` | Advantage standard deviation |

## Log Files

Each training run generates log files in `~/.hermes/logs/rl_training/`:

```
logs/
├── api_{run_id}.log        # Atropos API server logs
├── trainer_{run_id}.log    # Tinker trainer logs
├── env_{run_id}.log        # Environment process logs
└── inference_tests/        # Inference test results
    ├── test_{env}_{model}.jsonl
    └── test_{env}_{model}.log
```

These are invaluable for debugging when training fails or produces unexpected results.
