# Kimodo install and its licenses

Written by Claude Opus 5.5.

2026-10-11. NVIDIA's Kimodo (text to motion, plan 022 "The source of motion") is installed on this machine for the trial. This log records where it lives and what each piece's license allows for flanks. The licenses were read on their pages on 2026-10-11; this is a reading, not legal advice.

## Where it lives

Everything sits in `/data/ggando/kimodo` (/data had 168 GB free; the home drive was 94% full). No conda: a uv venv on pyenv's Python 3.10.9.

| Path | What |
|---|---|
| `env.sh` | `source` it: turns on the venv, keeps the uv cache, Hugging Face downloads and torch files in this folder, and sets `HF_TOKEN` from `HF_READONLY_KEY` in the flanks `.envrc` (only that one variable is read) |
| `.venv/` | Python 3.10.9, torch 2.11.0+cu128, Kimodo 1.0.0 (editable), py-soma-x 0.3.4, viser 1.0.16 |
| `src/` | github.com/nv-tlabs/kimodo at 58e7818 |
| `.uv-cache/`, `hf/`, `torch/` | caches and model downloads |

8.2 GB before any model download; the first run adds the Llama 3 8B encoder (about 16 GB) and Kimodo's weights. Deleting the folder removes all of it.

Install snag: the `all` extra pulls SOMA-X from git, and its LFS object `assets/Anny/SOMA_wrap.obj` is missing on GitHub ("remote missing object"). Installed with `GIT_LFS_SKIP_SMUDGE=1`. Nothing lost: SOMA-X downloads its body data from Hugging Face at run time (`soma/assets.py`), and Kimodo's own skeleton files (`somaskel30`, `somaskel77`) arrived whole. `kimodo_gen --help` runs on `cuda:0`.

Hugging Face: the token authenticates as `gtgando`. Access to `meta-llama/Meta-Llama-3-8B-Instruct` was refused on the first try; Gota submitted Meta's form the same day and access was granted within the hour.

## The pieces and their licenses

What runs when Kimodo generates: the prompt goes through the text encoder (Llama 3 8B Instruct with McGill's LLM2Vec adapter on top), Kimodo's diffusion model turns the encoding plus any pose constraints into joint rotations on its SOMA skeleton. Only those rotations leave the machine, after our transfer tool puts them on our skeleton.

| Piece | License | Commercial use | Outputs | Credit owed |
|---|---|---|---|---|
| Kimodo code (`src/`) | Apache-2.0 | yes | n/a | only if we ship the code |
| Kimodo-SOMA-RP-v1.1 weights | NVIDIA Open Model License | "This model is ready for commercial use." (model card) | "NVIDIA claims no ownership rights in outputs." (section 2.4) | only if we distribute the model (a Notice file, section 3.1) |
| Bones Rigplay 1, Kimodo's training data | proprietary capture, licensed by NVIDIA | n/a to us | n/a | none; the card says "dataset license constraints adhered to" |
| Meta Llama 3 8B Instruct (text encoder base) | Meta Llama 3 Community License | yes, under 700 million monthly users | the license claims none | "Built with Meta Llama 3" only when distributing the Llama Materials, or a product or service that uses them |
| LLM2Vec adapters (`McGill-NLP/LLM2Vec-Meta-Llama-3-8B-Instruct-mntp` and `-mntp-supervised`) and LLM2Vec code | MIT | yes | n/a | only if we ship them |
| py-soma-x | Apache-2.0 | yes | n/a | only if we ship it |
| viser (demo viewer) | MIT | yes | n/a | only if we ship it |

Not for us: `Kimodo-SMPLX-RP-v1` is under the NVIDIA R&D Model license (internal research only). Use the SOMA models.

## What it means for flanks

- The clips are ours to ship. NVIDIA claims no ownership of outputs, and Meta's license makes no claim on them.
- flanks ships animation files, not Kimodo, Llama or their weights, and runs none of them in the game. No credit line is owed in the game or on the store page. If that ever changes (a model inside the game, or the model files shipped), the credits above apply.
- Llama's only rule on outputs forbids using them to improve another large language model. Not our case.
- The NVIDIA license is revocable, and rights end on their own if a guardrail is circumvented (section 2.1). Clips already made are ours under section 2.4, but the grant is not permanent.
- Steam's content survey asks about AI-generated content made during development. Clips started from Kimodo go in that answer.

## Sources (read 2026-10-11)

- https://huggingface.co/nvidia/Kimodo-SOMA-RP-v1.1
- https://www.nvidia.com/en-us/agreements/enterprise-software/nvidia-open-model-license/
- https://huggingface.co/meta-llama/Meta-Llama-3-8B-Instruct (license text on the page)
- https://huggingface.co/McGill-NLP/LLM2Vec-Meta-Llama-3-8B-Instruct-mntp and `-mntp-supervised`
- `src/LICENSE`, `src/ATTRIBUTIONS.MD`, and the installed packages' metadata
- https://research.nvidia.com/labs/sil/projects/kimodo/docs (install, CLI, constraints)

## First use (2026-10-11)

- `kimodo_gen` made four 3 s high cuts in about 7 s (`/data/ggando/kimodo/out/highcut_test/`, NPZ and BVH; GIF `out/highcut_test.gif`). Sample 02 is the cleanest single overhead strike. No pose constraints yet, so they start and end in a plain stand, and the shield arm moves freely.
- The log's "MISSING lora" lines are harmless: transformers 5.1 first tries to load the MNTP adapter itself under the wrong key prefix, then Kimodo's `PeftModel.from_pretrained` loads it properly and merges it. Checked on layer 0 q_proj: the merged weight equals Llama + the MNTP delta to bf16 rounding (1.9e-3 against a 1.1e-2 delta), and the supervised adapter's weights equal the file.
- GUI: `/data/ggando/kimodo/demo.sh` serves Kimodo's web demo on http://127.0.0.1:7860, this machine only (the demo's own default listens on 0.0.0.0). It holds about 17.8 GB of GPU memory while it runs; stop it before playing or measuring.
