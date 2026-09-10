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

## 2. How it works (the pipeline)

```
  your prompt ──► CLIPLoader (Qwen3-0.6B) ──► CLIPTextEncode (positive) ─┐
  negative    ──► CLIPLoader (Qwen3-0.6B) ──► CLIPTextEncode (negative) ─┤
                                                                         ▼
  EmptyLatentImage (W×H) ─────────────────────────────────────► KSampler ──► VAEDecode (Qwen VAE) ──► image
                                       UNETLoader (Anima DiT) ──►    ▲
                                                                  (sampler, steps, cfg, seed)
```

1. **Text encoder (Qwen3-0.6B)** turns your positive and negative prompts into conditioning vectors.
2. **KSampler** starts from random noise (seeded) in a latent space and, over N steps, denoises it — the **DiT** predicts what to remove at each step, steered toward the positive prompt and away from the negative (strength = **CFG**).
3. **VAE** decodes the final latent into the actual pixels.

Because the text encoder is a tiny 0.6B model (not the ~5 GB T5-XXL used by many image models), total memory is low (~7 GB in use), so a 24 GB Mac has plenty of headroom.

---

## 3. What to expect — capabilities & limits

**It is good at:** anime / illustration / painterly non-photoreal imagery across a wide range of substyles (Ghibli-ish, retro-90s cel, shonen/shoujo, cyberpunk, watercolor, ukiyo-e, dark fantasy, mecha, chibi, cartoon, etc.), landscapes and environments, single-character portraits, mood/lighting.

**It is weak at (by design or by scale):**
- **Photorealism** — intentionally. It is a non-photoreal model; don't fight it.
- **Legible text** in images — expect garbled or approximate lettering. Don't rely on it for signage/logos.
- **High resolution** — it's trained around ~1 MP. Pushing far past that degrades coherence.
- **Complex multi-subject scenes / hands** — improves with explicit prompting ("both hands clearly visible and correct", specify subject counts like `2girls`/`4girls`) but remains the hardest case, as with all diffusion models.
- It's a **preview** checkpoint — details and consistency are rougher than a finished release.

---

## 4. Prompting

Anima uses Anima-Base's captioning, which is a **mix of Danbooru tags and natural language** — both work, and you can combine them.

**Tag order that works best:**
```
[quality / meta / year / safety]  [1girl / 1boy / 2girls ...]  [character]  [series]  [artist]  [general tags & natural language]
```

**Rules that actually matter (verified):**
- **Use LONG prompts.** Short prompts produce noticeably blander, mushier backgrounds. Describe the scene, lighting, mood, and details generously. (Every prompt in the test set is 40–80 words for this reason.)
- **Artist tags need a leading `@`** (e.g. `@artistname`) or the effect barely registers.
- **Score tags have no effect.** `score_9`, `score_7`, etc. were **not** in this model's training captions — they're inert. Don't include them (they were removed from this workflow's defaults).
- **Style is best summoned by describing the style**, e.g. `studio ghibli style, hand-painted watercolor backgrounds, ...` or `1990s retro anime style, cel shaded, film grain`. Named-franchise tags may or may not be trained; a style *description* is reliable.
- **`safe` / `sensitive` / `explicit`** are meta/safety tags from the Danbooru vocabulary; keep `safe` for SFW.

**Negative prompt** (a solid general-purpose default, already in the workflow):
```
lowres, worst quality, bad anatomy, bad hands, extra digits, fewer digits, jpeg artifacts, signature, watermark, username, blurry
```
Add targeted terms to suppress recurring artifacts you actually see.

---

## 5. Settings

| Setting | Default here | Range / notes |
|---|---|---|
| **Sampler** | `er_sde` | The base author's own pick. `euler` is a fine alternative to A/B. |
| **Scheduler** | `simple` | `sgm_uniform`, `beta`, `normal` are worth comparing. |
| **Steps** | 28 (iteration) / **50 (finals)** | Model card suggests 30–50. 28 is clean; more steps = more convergence, slower. |
| **CFG** | 4.0 | Model card 3.5–5. ↑ = stronger prompt adherence but risks oversaturation/contrast. <3.5 washes out. |
| **Seed** | any (fix it while testing) | Fix the seed to make *prompt/setting* the only variable. |
| **Weight dtype** | `default` (bf16) | Best quality. **Do not** use the int8 build on Mac (see §8). |

**Measured speed (M5 Pro, 24 GB, MPS):** ~**7 s/it**, so a 28-step ~1 MP image ≈ **3.5 min** (first run adds ~30–60 s for model load + MPS graph compile). A 50-step finals image ≈ **6 min**.

---

## 6. Resolution types

Target **~1 megapixel**. Good width×height pairs (all near 1 MP, dimensions are multiples of 8/16):

| Aspect | Dimensions | Use |
|---|---|---|
| 1:1 square | **1024 × 1024** | icons, single subjects, food/objects |
| 2:3 portrait | **832 × 1216** | character portraits, standing figures |
| 3:2 landscape | **1216 × 832** | scenery, group shots, action |
| 3:4 / 4:3 | 896 × 1152 / 1152 × 896 | gentler portrait/landscape |
| 9:16 / 16:9 | 768 × 1344 / 1344 × 768 | tall/wide cinematic |

Going above ~1.3 MP tends to introduce duplication and incoherence. For bigger final images, generate at ~1 MP and upscale as a separate step.

---

## 7. Running it

### A. In the ComfyUI UI (interactive)
```bash
cd /Users/ishaan/Desktop/comfy/ComfyUI && source venv/bin/activate && export PYTORCH_ENABLE_MPS_FALLBACK=1 && comfy launch
```
Open **http://127.0.0.1:8188** → **Workflows** sidebar → **Anima-2.9B**. The front node exposes the **positive prompt**, width, height, steps, cfg, seed, and the three model dropdowns. The **negative prompt** and **sampler/scheduler** live *inside* the subgraph — double-click the node to edit them. Hit **Queue** (or Cmd/Ctrl+Enter). Images save to `ComfyUI/output/`.

*(Use `comfy launch`, not `python main.py`, so console output — including s/it and loader messages — is captured to `user/comfyui_8188.log`.)*

### B. Scripted / batch (headless API)
ComfyUI exposes an HTTP API on the same port: `POST /prompt` with a graph in API format, poll `GET /history/<id>`, images land in `output/`. The batch that produced this folder's images (`anima_batch.py`) does exactly this — it's a useful template for programmatic runs (edit the `PROMPTS` list and re-run).

---

## 8. How to test / iterate (methodology)

- **Change one axis at a time**, with the **seed fixed**, or results are uninterpretable. Want to compare styles? Fix seed + settings, vary only the prompt (that's how this folder's set was made — seed 12345 throughout). Want to tune CFG? Fix everything else.
- **Fast loop:** drop steps to ~12–16 and/or resolution to 512×768 to preview composition in ~1 min, then re-run the keeper at full steps.
- **⚠️ Do NOT use a Turbo/LCM LoRA to speed up this model.** The only Anima turbo LoRA (`anima-turbo-lora-v0.2`) was trained on **28-block Anima-Base**. On this **40-block** model it lands on the wrong layers and fails **silently** — no error, plausible-looking garbage. Keep `models/loras/` empty. For speed, lower steps instead.

---

## 9. Troubleshooting & gotchas (all verified on this install)

- **The silent block-count trap (the big one).** Old ComfyUI builds could hardcode a 28-block model for this architecture, load the 40-block weights with `strict=False`, throw **no error**, and generate with 12 layers missing → plausible-looking garbage. **Fixed in v0.33.1** (PR #15555), which derives the count from the state dict. This install is **0.34.0**, and the file correctly trips the *Cosmos-Predict2* detection branch (`model_detection.py:872`, derived count = **40**), not the hardcoded Cosmos-1 branch. If you ever run on an older/Desktop build, verify the load log says **40 blocks** before trusting output.
- **Benign warning — ignore it:** `unet unexpected: ['pos_embedder.dim_spatial_range', 'pos_embedder.dim_temporal_range', 'pos_embedder.seq']`. These are *extra* RoPE metadata buffers with no slot in the model. *Unexpected* keys are harmless; *missing* keys would be the alarm.
- **Skip the int8 build on Mac.** `Anima-2.9B-preview-v1_int8_convrot.safetensors` (3.08 GB) loads on MPS then dies in the first KSampler matmul: `NotImplementedError: aten::_int_mm not implemented for MPS`. You have 24 GB — use bf16, there's nothing to gain.
- **MPS fallback:** always `export PYTORCH_ENABLE_MPS_FALLBACK=1` before launching, so any op without an MPS kernel falls back to CPU instead of crashing.

---

## 10. Local vs cloud

The Mac works and is great for correctness checks and offline iteration, but ~3–6 min/image is slow for volume. A single **L4 / A10G** does 28 steps in under 15 s. If you scale up: run ComfyUI headless on the GPU box with `--listen`, tunnel the port, and drive it from the Mac's browser. Use local MPS for validation, cloud for throughput.

---

## 11. Licensing (resolve before any commercial/client use)

The chain is: **Cosmos-Predict2 (NVIDIA Open Model License)** → **CircleStone Labs Anima (non-commercial terms)** → **Gazingstars123 community build**. Non-commercial restrictions and NVIDIA license both propagate downward, and the community build adds its own uncertainty. Read the full chain and confirm rights before using outputs commercially or for a client. This guide is not legal advice.

---

