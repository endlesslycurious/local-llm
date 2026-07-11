# Agent Instructions

This repo contains model configurations, daemons and personal documentation related to my experiments running LLM (AI) models on a dedicated host:
- The host machine is a M2 Max Mac Studio with 32 GB RAM.
- The hosted model is accessed from my MacBook Pro during programming sessions.
- The hosted model is ran with a dedicated admin account with auto-login enabled.

## Updating the docs

If the user asks you to 'update the doc' do the following:
1. Review the oMLX and its model configurations in `config` directory.
2. Review the documents: `Models.md`, `README.md`, `Setup.md` and `Tuning.md`.
3. Treating `settings.json` as the source of truth, update `Tuning.md` section on oMLX settings. The interesting settings sections to document as sub-sections under `oMLX settings` are `memory`, `scheduler` (concurrency) and `cache`, ignore other sections.
4. Treating from `model_settings.json` as the source of truth update `Models.md`. Each model should get its own section in `Models.md`, see the following section for layout, models should be order from latest to oldest by model name. For models no longer in the `model_settings.json` should be considered not in use and left in `Models.md` for reference.
5. The User may indicate a model has been retired, its detailed listing should be removed from Models.md and a line added to the retired models table, ask the user for the note/reason.

### Model details
For example the following model settings section for Qwen 3.5:
- If in doubt ask the user for the correct URLs for model and repo, URLS should be valid (ie return HTTP 200). Check the links work!
- Additional advanced settings e.g. dflash, turboquant, MTP etc should go in the notes section and only listed if ther are enabled e.g. `dflash_enabled=: true` then list dflash settings, etc.

```markdown
### Qwen 3.5 9B

| Model | Repo | Params. | Quant | File size | Thinking | Date |
|:-------:|:-------:|:-----:|:---------:|:-------:|:-------------: |:----:|
| [Qwen 3.5](https://huggingface.co/Qwen/Qwen3.5-9B) | [Qwen3.5-9B](https://huggingface.co/mlx-community/Qwen3.5-9B-MLX-4bit) | 9B | 4 bit | ~5.3 GB | No  | 2026-06 |

Configured basic settings:
| Context | Max Tokens | Temp | Top P | Top K | Rep. Penality | Min P | Pres. Penality |
|:-------:|:-----:|:---------:|:-------:|:-------------: |:----:|:----:|:----:|
| 48k | 16k | 0.6 | 0.95 | 20 | 1.0 | 0.0 | 1.5|

#### Notes
- Current default/pinned model.
- Excellent speed-to-capability ratio.
```
