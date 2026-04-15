---
title: 图像生成
description: 使用 FLUX 2 Pro 生成高质量图像，并通过 FAL.ai 自动放大。
sidebar_label: 图像生成
sidebar_position: 6
---

# 图像生成

Hermes Agent 可以使用 FAL.ai 的 **FLUX 2 Pro** 模型从文本提示生成图像，并通过 **Clarity Upscaler** 自动 2 倍放大以提高质量。

## 设置

### 获取 FAL API 密钥

1. 在 [fal.ai](https://fal.ai/) 注册
2. 从您的仪表板生成 API 密钥

### 配置密钥

```bash
# 添加到 ~/.hermes/.env
FAL_KEY=your-fal-api-key-here
```

### 安装客户端库

```bash
pip install fal-client
```

:::info
当设置了 `FAL_KEY` 时，图像生成工具会自动可用。不需要额外的工具集配置。
:::

## 工作原理

当您要求 Hermes 生成图像时：

1. **生成** — 您的提示被发送到 FLUX 2 Pro 模型 (`fal-ai/flux-2-pro`)
2. **放大** — 生成的图像通过 Clarity Upscaler (`fal-ai/clarity-upscaler`) 自动 2 倍放大
3. **交付** — 返回放大后的图像 URL

如果放大因任何原因失败，原始图像将作为备用返回。

## 使用

只需要求 Hermes 创建图像：

```
Generate an image of a serene mountain landscape with cherry blossoms
```

```
Create a portrait of a wise old owl perched on an ancient tree branch
```

```
Make me a futuristic cityscape with flying cars and neon lights
```

## 参数

`image_generate_tool` 接受这些参数：

| 参数 | 默认值 | 范围 | 描述 |
|-----------|---------|-------|-------------|
| `prompt` | *(必填)* | — | 所需图像的文本描述 |
| `aspect_ratio` | `"landscape"` | `landscape`, `square`, `portrait` | 图像宽高比 |
| `num_inference_steps` | `50` | 1–100 | 去噪步骤数（更多 = 更高质量，更慢） |
| `guidance_scale` | `4.5` | 0.1–20.0 | 与提示的接近程度 |
| `num_images` | `1` | 1–4 | 要生成的图像数量 |
| `output_format` | `"png"` | `png`, `jpeg` | 图像文件格式 |
| `seed` | *(随机)* | 任何整数 | 用于可重现结果的随机种子 |

## 宽高比

该工具使用简化的宽高比名称，映射到 FLUX 2 Pro 图像尺寸：

| 宽高比 | 映射到 | 最适合 |
|-------------|---------|----------|
| `landscape` | `landscape_16_9` | 壁纸、横幅、场景 |
| `square` | `square_hd` | 个人资料图片、社交媒体帖子 |
| `portrait` | `portrait_16_9` | 角色艺术、手机壁纸 |

:::tip
您也可以直接使用原始 FLUX 2 Pro 尺寸预设：`square_hd`、`square`、`portrait_4_3`、`portrait_16_9`、`landscape_4_3`、`landscape_16_9`。还支持高达 2048x2048 的自定义尺寸。
:::

## 自动放大

每个生成的图像都使用 FAL.ai 的 Clarity Upscaler 自动 2 倍放大，设置如下：

| 设置 | 值 |
|---------|-------|
| 放大因子 | 2x |
| 创意 | 0.35 |
| 相似度 | 0.6 |
| 引导比例 | 4 |
| 推理步骤 | 18 |
| Positive Prompt | `"masterpiece, best quality, highres"` + 您的原始提示 |
| Negative Prompt | `"(worst quality, low quality, normal quality:2)"` |

放大器在保留原始构图的同时增强细节和分辨率。如果放大器失败（网络问题、速率限制），会自动返回原始分辨率的图像。

## 示例提示

以下是一些有效的提示示例：

```
A candid street photo of a woman with a pink bob and bold eyeliner
```

```
Modern architecture building with glass facade, sunset lighting
```

```
Abstract art with vibrant colors and geometric patterns
```

```
Portrait of a wise old owl perched on ancient tree branch
```

```
Futuristic cityscape with flying cars and neon lights
```

## Debugging

Enable debug logging for image generation:

```bash
export IMAGE_TOOLS_DEBUG=true
```

Debug logs are saved to `./logs/image_tools_debug_<session_id>.json` with details about each generation request, parameters, timing, and any errors.

## Safety Settings

The image generation tool runs with safety checks disabled by default (`safety_tolerance: 5`, the most permissive setting). This is configured at the code level and is not user-adjustable.

## Platform Delivery

Generated images are delivered differently depending on the platform:

| Platform | Delivery method |
|----------|----------------|
| **CLI** | Image URL printed as markdown `![description](url)` — click to open in browser |
| **Telegram** | Image sent as a photo message with the prompt as caption |
| **Discord** | Image embedded in a message |
| **Slack** | Image URL in message (Slack unfurls it) |
| **WhatsApp** | Image sent as a media message |
| **Other platforms** | Image URL in plain text |

The agent uses `MEDIA:<url>` syntax in its response, which the platform adapter converts to the appropriate format.

## Limitations

- **Requires FAL API key** — image generation incurs API costs on your FAL.ai account
- **No image editing** — this is text-to-image only, no inpainting or img2img
- **URL-based delivery** — images are returned as temporary FAL.ai URLs, not saved locally. URLs expire after a period (typically hours)
- **Upscaling adds latency** — the automatic 2x upscale step adds processing time
- **Max 4 images per request** — `num_images` is capped at 4
