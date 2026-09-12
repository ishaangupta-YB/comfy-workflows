# Anima-2.9B — local testing & workflow

Self-contained project for running and testing the **Anima-2.9B** anime text-to-image model
on Apple Silicon via ComfyUI. Kept separate from the ComfyUI repo checkout at
`~/Desktop/comfy/ComfyUI` so it stays clean.

## Contents

| Path | Tracked? | What it is |
|---|---|---|
| `Anima-2.9B-GUIDE.md` | ✅ | Full guide: what the model is, how it works, prompting, settings, resolutions, troubleshooting. |
| `Anima-2.9B.workflow.json` | ✅ | The ComfyUI workflow. Import into ComfyUI to run (drag it onto the canvas). |
| `ANIMA-HANDOFF.md` | ✅ | Session notes and verification history. |
| `generations/` | ❌ (gitignored) | All test images (`NN_slug.png`) + per-image prompt/config sidecars (`NN_slug.txt`), plus `_manifest.txt` and `_batch.log`. Local only, not pushed. |

## Running it

The model weights live in the ComfyUI checkout (required for ComfyUI):
`~/Desktop/comfy/ComfyUI/models/{diffusion_models,text_encoders,vae}/`. This project holds
the *workflow and docs* — not the weights.

1. Start ComfyUI: `cd ~/Desktop/comfy/ComfyUI && source venv/bin/activate && export PYTORCH_ENABLE_MPS_FALLBACK=1 && comfy launch`
2. Open http://127.0.0.1:8188 → drag `Anima-2.9B.workflow.json` onto the canvas (or drag any
   image from `generations/` — ComfyUI reads the embedded workflow).
3. See `Anima-2.9B-GUIDE.md` for prompting tips and settings.

> Any ComfyUI-generated PNG embeds its full workflow in metadata, so the images in
> `generations/` work as reloadable workflows.
