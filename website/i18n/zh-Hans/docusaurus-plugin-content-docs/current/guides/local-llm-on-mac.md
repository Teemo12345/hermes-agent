---
sidebar_position: 2
title: "在 Mac 上运行本地 LLM"
description: "使用 llama.cpp 或 MLX 在 macOS 上设置本地 OpenAI 兼容的 LLM 服务器，包括模型选择、内存优化和 Apple Silicon 上的真实基准测试"
---

# 在 Mac 上运行本地 LLM

本指南将引导您在 macOS 上运行具有 OpenAI 兼容 API 的本地 LLM 服务器。您将获得完全的隐私、零 API 成本，以及在 Apple Silicon 上令人惊讶的良好性能。

我们涵盖两个后端：

| 后端 | 安装 | 最擅长 | 格式 |
|---------|---------|---------|--------|
| **llama.cpp** | `brew install llama.cpp` | 最快首次令牌时间，量化 KV 缓存适用于低内存 | GGUF |
| **omlx** | [omlx.ai](https://omlx.ai) | 最快令牌生成，原生 Metal 优化 | MLX (safetensors) |

两者都暴露一个 OpenAI 兼容的 `/v1/chat/completions` 端点。Hermes 可以与其中任何一个一起工作 — 只需将其指向 `http://localhost:8080` 或 `http://localhost:8000`。

:::info 仅限 Apple Silicon
本指南针对具有 Apple Silicon（M1 及更高版本）的 Mac。Intel Mac 将与 llama.cpp 一起工作，但没有 GPU 加速 — 预计性能会显著较慢。
:::

---

## 选择模型

对于入门，我们推荐 **Qwen3.5-9B** — 这是一个强大的推理模型，通过量化可以舒适地适应 8GB+ 的统一内存。

| 变体 | 磁盘大小 | 所需 RAM（128K 上下文） | 后端 |
|---------|-------------|---------------------------|---------|
| Qwen3.5-9B-Q4_K_M (GGUF) | 5.3 GB | ~10 GB（带量化 KV 缓存） | llama.cpp |
| Qwen3.5-9B-mlx-lm-mxfp4 (MLX) | ~5 GB | ~12 GB | omlx |

**内存经验法则：** 模型大小 + KV 缓存。一个 9B Q4 模型约为 5 GB。在 128K 上下文下，使用 Q4 量化的 KV 缓存增加约 4-5 GB。使用默认（f16）KV 缓存，这会膨胀到约 16 GB。llama.cpp 中的量化 KV 缓存标志是内存受限系统的关键技巧。

对于更大的模型（27B、35B），您将需要 32 GB+ 的统一内存。9B 是 8-16 GB 机器的理想选择。

---

## 选项 A：llama.cpp

llama.cpp 是最便携的本地 LLM 运行时。在 macOS 上，它默认使用 Metal 进行 GPU 加速。

### 安装

```bash
brew install llama.cpp
```

This gives you the `llama-server` command globally.

### Download the model

You need a GGUF-format model. The easiest source is Hugging Face via the `huggingface-cli`:

```bash
brew install huggingface-cli
```

Then download:

```bash
huggingface-cli download unsloth/Qwen3.5-9B-GGUF Qwen3.5-9B-Q4_K_M.gguf --local-dir ~/models
```

:::tip Gated models
Some models on Hugging Face require authentication. Run `huggingface-cli login` first if you get a 401 or 404 error.
:::

### Start the server

```bash
llama-server -m ~/models/Qwen3.5-9B-Q4_K_M.gguf \
  -ngl 99 \
  -c 131072 \
  -np 1 \
  -fa on \
  --cache-type-k q4_0 \
  --cache-type-v q4_0 \
  --host 0.0.0.0
```

Here's what each flag does:

| Flag | Purpose |
|------|---------|
| `-ngl 99` | Offload all layers to GPU (Metal). Use a high number to ensure nothing stays on CPU. |
| `-c 131072` | Context window size (128K tokens). Reduce this if you're low on memory. |
| `-np 1` | Number of parallel slots. Keep at 1 for single-user use — more slots split your memory budget. |
| `-fa on` | Flash attention. Reduces memory usage and speeds up long-context inference. |
| `--cache-type-k q4_0` | Quantize the key cache to 4-bit. **This is the big memory saver.** |
| `--cache-type-v q4_0` | Quantize the value cache to 4-bit. Together with the above, this cuts KV cache memory by ~75% vs f16. |
| `--host 0.0.0.0` | Listen on all interfaces. Use `127.0.0.1` if you don't need network access. |

The server is ready when you see:

```
main: server is listening on http://0.0.0.0:8080
srv  update_slots: all slots are idle
```

### Memory optimization for constrained systems

The `--cache-type-k q4_0 --cache-type-v q4_0` flags are the most important optimization for systems with limited memory. Here's the impact at 128K context:

| KV cache type | KV cache memory (128K ctx, 9B model) |
|---------------|--------------------------------------|
| f16 (default) | ~16 GB |
| q8_0 | ~8 GB |
| **q4_0** | **~4 GB** |

On an 8 GB Mac, use `q4_0` KV cache and reduce context to `-c 32768` (32K). On 16 GB, you can comfortably do 128K context. On 32 GB+, you can run larger models or multiple parallel slots.

If you're still running out of memory, reduce context size first (`-c`), then try a smaller quantization (Q3_K_M instead of Q4_K_M).

### Test it

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-9B-Q4_K_M.gguf",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }' | jq .choices[0].message.content
```

### Get the model name

If you forget the model name, query the models endpoint:

```bash
curl -s http://localhost:8080/v1/models | jq '.data[].id'
```

---

## Option B: MLX via omlx

[omlx](https://omlx.ai) is a macOS-native app that manages and serves MLX models. MLX is Apple's own machine learning framework, optimized specifically for Apple Silicon's unified memory architecture.

### Install

Download and install from [omlx.ai](https://omlx.ai). It provides a GUI for model management and a built-in server.

### Download the model

Use the omlx app to browse and download models. Search for `Qwen3.5-9B-mlx-lm-mxfp4` and download it. Models are stored locally (typically in `~/.omlx/models/`).

### Start the server

omlx serves models on `http://127.0.0.1:8000` by default. Start serving from the app UI, or use the CLI if available.

### Test it

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-9B-mlx-lm-mxfp4",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }' | jq .choices[0].message.content
```

### List available models

omlx can serve multiple models simultaneously:

```bash
curl -s http://127.0.0.1:8000/v1/models | jq '.data[].id'
```

---

## Benchmarks: llama.cpp vs MLX

Both backends tested on the same machine (Apple M5 Max, 128 GB unified memory) running the same model (Qwen3.5-9B) at comparable quantization levels (Q4_K_M for GGUF, mxfp4 for MLX). Five diverse prompts, three runs each, backends tested sequentially to avoid resource contention.

### Results

| Metric | llama.cpp (Q4_K_M) | MLX (mxfp4) | Winner |
|--------|-------------------|-------------|--------|
| **TTFT (avg)** | **67 ms** | 289 ms | llama.cpp (4.3x faster) |
| **TTFT (p50)** | **66 ms** | 286 ms | llama.cpp (4.3x faster) |
| **Generation (avg)** | 70 tok/s | **96 tok/s** | MLX (37% faster) |
| **Generation (p50)** | 70 tok/s | **96 tok/s** | MLX (37% faster) |
| **Total time (512 tokens)** | 7.3s | **5.5s** | MLX (25% faster) |

### What this means

- **llama.cpp** excels at prompt processing — its flash attention + quantized KV cache pipeline gets you the first token in ~66ms. If you're building interactive applications where perceived responsiveness matters (chatbots, autocomplete), this is a meaningful advantage.

- **MLX** generates tokens ~37% faster once it gets going. For batch workloads, long-form generation, or any task where total completion time matters more than initial latency, MLX finishes sooner.

- Both backends are **extremely consistent** — variance across runs was negligible. You can rely on these numbers.

### Which one should you pick?

| Use case | Recommendation |
|----------|---------------|
| Interactive chat, low-latency tools | llama.cpp |
| Long-form generation, bulk processing | MLX (omlx) |
| Memory-constrained (8-16 GB) | llama.cpp (quantized KV cache is unmatched) |
| Serving multiple models simultaneously | omlx (built-in multi-model support) |
| Maximum compatibility (Linux too) | llama.cpp |

---

## Connect to Hermes

Once your local server is running:

```bash
hermes model
```

Select **Custom endpoint** and follow the prompts. It will ask for the base URL and model name — use the values from whichever backend you set up above.

---

## Timeouts

Hermes automatically detects local endpoints (localhost, LAN IPs) and relaxes its streaming timeouts. No configuration needed for most setups.

If you still hit timeout errors (e.g. very large contexts on slow hardware), you can override the streaming read timeout:

```bash
# In your .env — raise from the 120s default to 30 minutes
HERMES_STREAM_READ_TIMEOUT=1800
```

| Timeout | Default | Local auto-adjustment | Env var override |
|---------|---------|----------------------|------------------|
| Stream read (socket-level) | 120s | Raised to 1800s | `HERMES_STREAM_READ_TIMEOUT` |
| Stale stream detection | 180s | Disabled entirely | `HERMES_STREAM_STALE_TIMEOUT` |
| API call (non-streaming) | 1800s | No change needed | `HERMES_API_TIMEOUT` |

The stream read timeout is the one most likely to cause issues — it's the socket-level deadline for receiving the next chunk of data. During prefill on large contexts, local models may produce no output for minutes while processing the prompt. The auto-detection handles this transparently.