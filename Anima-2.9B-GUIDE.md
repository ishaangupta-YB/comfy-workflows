# Anima-2.9B — Complete Practical Guide

*A working reference for running, prompting, and testing Anima-2.9B locally on Apple Silicon (tested on M5 Pro, 24 GB, ComfyUI 0.34.0, PyTorch 2.14.0 / MPS).*

---

## 1. What this model is (read this first)

**Anima-2.9B is a text-to-image diffusion transformer (DiT). It is not a language model.** You cannot run it in Ollama, Unsloth, or a plain `diffusers` pipeline — none of them have the diffusion sampler / VAE / text-encoder-to-DiT path this model needs. **ComfyUI is the only supported way to run it**, and it has native support (since ComfyUI v0.33.1).

### Genealogy (matters for behavior *and* licensing)

```
nvidia/Cosmos-Predict2-2B-Text2Image        (NVIDIA Open Model License)
   └── circlestone-labs/Anima                28 DiT blocks, ~2B params   ("Anima-Base")
          └── Gazingstars123/Anima-2.9B       40 DiT blocks, ~2.9B params  ← THIS MODEL
```

Anima-2.9B is a **community build** (Gazingstars123), not an official CircleStone release. It deepens Anima-Base from 28 to 40 transformer blocks. Everything downstream — the 40-block count, the prompting behavior, the licensing chain — comes from this lineage.

### The three required components

| Part | File | Size | Notes |
|---|---|---|---|
| DiT (the model) | `Anima-2.9B-preview-v1.safetensors` | ~5.8 GB | bf16, **40 blocks** |
| Text encoder | `qwen_3_06b_base.safetensors` | 1.19 GB | **Qwen3-0.6B** (not T5-XXL) — this is why it fits in 24 GB |
| VAE | `qwen_image_vae.safetensors` | 254 MB | Qwen-Image VAE |

They live under `ComfyUI/models/` in `diffusion_models/`, `text_encoders/`, and `vae/` respectively.

---

