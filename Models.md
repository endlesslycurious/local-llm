# Models Tried

Personal log of every model pulled and tested on the Mac Studio M2 Max (32 GB). See [README.md](README.md) for the tier cheat sheet, or the [insiderllm guide](https://insiderllm.com/guides/best-local-llms-mac-2026/) for current recommendations.

Add a row each time you pull and test a new model. Link directly to the HuggingFace repo so the model can be re-pulled later.

---

## Model Downloading

### oMLX

oMLX can search for and download models itself from [Hugging Face](https://huggingface.co) via the admin dashboard, all that is needed is either the repo ID or the name.

### Hugging Face CLI

We have also setup the [Hugging Face CLI](https://huggingface.co/docs/huggingface_hub/en/guides/cli) which can also download models directly too.

```bash
# Manual download via HF CLI
hf download mlx-community/Qwen3.5-9B-4bit-mlx --local-dir ~/models/mlx-community/Qwen3.5-9B-4bit-mlx
```

---

## Model Configuration

After downloading a model, configure it via the admin panel:

1. Open `http://<host>:8080/admin`
2. Go to Model Settings
3. Configure per model:
   - Context window (e.g., 131072 for 128K)
   - Temperature, top_p, top_k
   - TurboQuant KV cache (8-bit recommended)
   - Pin as default if desired

### Model Tuning
See [Tuning.md](Tuning.md) for recommended settings and tuning tips.

---

## Model Log

Ordered from latest to oldest model tried. Models not in `model_settings.json` are kept for reference.

### Qwen 3.6 27B (Thinker)

| Model | Repo | Params. | Quant | File size | Thinking | Date |
|:-------:|:-------:|:-----:|:---------:|:-------:|:-------------: |:----:|
| [Qwen 3.6](https://huggingface.co/Qwen/Qwen3.6-27B) | [Qwen3.6-27B](https://huggingface.co/mlx-community/Qwen3.6-27B-MLX-4bit) | 27B | 4 bit | ~17.6 GB | Yes | 2026-06 |

Configured basic settings:
| Context | Max Tokens | Temp | Top P | Top K | Rep. Penalty |
|:-------:|:-----:|:---------:|:-------:|:-------------: |:----:|
| 48k | 8k | 0.6 | 0.95 | 20 | 1.0 |

#### Notes
- More capable model for deep thinking tasks.
- Thinking mode enabled for chain-of-thought reasoning.
- TurboQuant KV cache enabled at 4-bit precision (q4_0) with skip last token for maximum memory efficiency.
- DFlash in-memory cache enabled with 8GB max and 4 entry limit for recent context.
- Not pinned - use when advanced reasoning is needed, not as daily driver.


### Qwen 3.5 9B (Default Model)

| Model | Repo | Params. | Quant | File size | Thinking | Date |
|:-------:|:-------:|:-----:|:---------:|:-------:|:-------------: |:----:|
| [Qwen 3.5](https://huggingface.co/Qwen/Qwen3.5-9B) | [Qwen3.5-9B](https://huggingface.co/mlx-community/Qwen3.5-9B-MLX-4bit) | 9B | 4 bit | ~5.3 GB | No | 2026-06

Configured basic settings:
| Context | Max Tokens | Temp | Top P | Top K | Rep. Penalty | Min P | Pres. Penalty |
|:-------:|:-----:|:---------:|:-------:|:-------------: |:----:|:----:|:----:|
| 128k | 8k | 0.7 | 0.8 | 20 | 1.0 | 0.0 | 1.5|

#### Advanced Settings
- **TurboQuant KV cache**: enabled at 4-bit precision with skip last token
- **Speculative Prefill**: enabled with Qwen3.5-0.8B drafter
  - Threshold: 64k tokens (triggers when context > 64k)
  - Keep ratio: 20% of draft tokens

#### Notes
- **Current default model.**
- Excellent speed-to-capability ratio for daily use.
- **Speculative decoding enabled** with Qwen3.5-0.8B drafter for faster generation on long contexts.
- Thinking mode disabled for more direct, efficient responses.
- 128k context window enabled via turboquant kv compression.
- Recommended for most 16-32GB systems due to balance of performance and quality.

### Qwen 3.5 0.8B (Drafter)

| Model | Repo | Params. | Quant | File size | Thinking | Date |
|:-------:|:-------:|:-----:|:---------:|:-------:|:-------------: |:----:|
| [Qwen 3.5](https://huggingface.co/Qwen/Qwen3.5-0.8B) | [Qwen3.5-0.8B](https://huggingface.co/mlx-community/Qwen3.5-0.8B-MLX-4bit) | 0.8B | 4 bit | ~530 MB | No | 2026-06

Configured basic settings:
| Context | Max Tokens | Temp |
|:-------:|:-----:|:---------:|
| 4k | 256 | 0.2 |

KV Cache Settings:
| Setting | Value | Description |
|:--------:|:------:|:------------|
| `turboquant_kv_enabled` | true | TurboQuant enabled for KV cache compression |
| `turboquant_kv_bits` | 4.0 | 4-bit quantization for maximum memory efficiency |

#### Notes
- **Drafter model for speculative decoding** when used with Qwen3.5-9B.
- Smallest model in the configuration, optimized for prefill phase speed.
- Lower temperature (0.2) for more deterministic draft predictions.
- TurboQuant KV cache enabled at 4-bit precision for maximum efficiency.
- Pinned - for quick code completion responses.

---

## Retired Models

|   Date  |  Model  | Notes                                                |
|:-------:|:-------:|------------------------------------------------------|
| 2026/06 | [Qwen 3 Coder](https://huggingface.co/mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit) | 30B-A3B MoE is interesting but 27B is more capable. |
| 2026/05 | [Gemma 4](https://huggingface.co/google/gemma-4-12B-it) | Seems highly capable but keeps stopping mid-thought. requiring babysitting to complete tasks. |
