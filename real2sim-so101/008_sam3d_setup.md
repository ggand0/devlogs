# 008 — SAM 3D Objects Environment Setup

2026-05-04

## Overview

Set up Meta's SAM 3D Objects (facebook/sam-3d-objects) for single-image 3D mesh/splat generation. Goal: scan the real red cube and green bowl for sim grasping.

## Environment

- **Venv:** `/data/sam-3d-objects/.venv` (Python 3.11.11, ~13 GB)
- **Repo:** `/data/sam-3d-objects/` (cloned from HuggingFace)
- **CUDA toolkit:** `/data/cuda-12.1` (installed from runfile, used for gsplat build)
- **uv cache:** `/data/.cache/uv` (redirected from `~/.cache/uv` to keep root disk free)

## Key Package Versions

| Package | Version | Source |
|---|---|---|
| torch | 2.5.1+cu121 | PyTorch wheel index |
| pytorch3d | 0.7.8+pt2.5.1cu121 | MiroPsota prebuilt wheels |
| flash_attn | 2.8.3 | PyPI |
| kaolin | 0.17.0 | NVIDIA S3 wheels |
| gsplat | git@2323de5 | Built from source |
| xformers | 0.0.28.post3 | PyPI |
| open3d | 0.18.0 | PyPI |
| spconv-cu121 | 2.3.8 | PyPI |
| lightning | 2.3.3 | PyPI |

## Issues Encountered and Solutions

### 1. nvidia-pyindex breaks uv builds
`nvidia-pyindex` modifies pip.conf at build time and requires the `pip` module. Removed from requirements.txt along with `pip-system-certs`, `conda-pack`, and `dataclasses` (built-in on 3.11).

### 2. CUDA 12.1 incompatible with Ubuntu 24.04 glibc
CUDA 12.1's nvcc rejects `_Float32`/`_Float64` types from glibc 2.39 (Ubuntu 24.04). Even `--allow-unsupported-compiler` doesn't help since the issue is in system headers, not the compiler version.

**Solution:** Build CUDA extensions with system CUDA 13.2 (which supports glibc 2.39) and patch `torch/utils/cpp_extension.py` to skip the CUDA version mismatch check (12.1 vs 13.2). The patch is at line 395 in the venv's copy.

### 3. GCC 13 unsupported by CUDA 12.1
CUDA 12.1 only supports up to GCC 12. System has GCC 13.3.

**Solution:** Moot — using system CUDA 13.2 for builds (see above).

### 4. pytorch3d linker errors (pulsar module)
Building pytorch3d from the pinned git commit fails with undefined reference errors in the pulsar renderer when using CUDA 13.2 + GCC 13.

**Solution:** Use MiroPsota's prebuilt wheels: `--extra-index-url https://miropsota.github.io/torch_packages_builder` with `pytorch3d==0.7.8+pt2.5.1cu121`. Facebook's official wheel hosting (dl.fbaipublicfiles.com) is dead (403).

### 5. gsplat needs torch at build time
gsplat doesn't declare torch as a build dependency. uv's build isolation can't find it.

**Solution:** `--no-build-isolation` so gsplat's setup.py can import torch from the venv.

### 6. open3d needs libc++
open3d 0.18.0 links against `libc++.so.1` which isn't installed by default on Ubuntu 24.04.

**Solution:** `sudo apt install libc++1`

### 7. setuptools 82 removed pkg_resources
Lightning imports `pkg_resources` which was removed in setuptools 82.

**Solution:** Pin `setuptools<81`.

### 8. sam3d_objects.init doesn't exist
The `__init__.py` imports `sam3d_objects.init` which is a Meta-internal module.

**Solution:** Set `LIDRA_SKIP_INIT=true` (the inference notebook already does this).

## Install Commands (Reproducible)

```bash
# Redirect uv cache to /data
export UV_CACHE_DIR=/data/.cache/uv

# Remove toxic packages from requirements.txt first:
#   nvidia-pyindex, pip-system-certs, conda-pack, dataclasses

# Base install
uv pip install --python /data/sam-3d-objects/.venv/bin/python \
  --extra-index-url https://pypi.ngc.nvidia.com \
  --extra-index-url https://download.pytorch.org/whl/cu121 \
  --index-strategy unsafe-best-match \
  --refresh \
  -e "/data/sam-3d-objects"

# Inference extras (kaolin, gsplat) — needs --no-build-isolation for gsplat
export CUDA_HOME=/usr/local/cuda-13.2
export TORCH_CUDA_ARCH_LIST="8.6"
uv pip install --python /data/sam-3d-objects/.venv/bin/python \
  --extra-index-url https://pypi.ngc.nvidia.com \
  --extra-index-url https://download.pytorch.org/whl/cu121 \
  --find-links "https://nvidia-kaolin.s3.us-east-2.amazonaws.com/torch-2.5.1_cu121.html" \
  --index-strategy unsafe-best-match \
  --no-build-isolation \
  -e "/data/sam-3d-objects[inference]"

# pytorch3d — prebuilt wheel (Facebook's hosting is dead)
uv pip install --python /data/sam-3d-objects/.venv/bin/python \
  --extra-index-url https://miropsota.github.io/torch_packages_builder \
  "pytorch3d==0.7.8+pt2.5.1cu121"

# flash_attn
uv pip install --python /data/sam-3d-objects/.venv/bin/python flash-attn==2.8.3

# Fix setuptools for lightning
uv pip install --python /data/sam-3d-objects/.venv/bin/python "setuptools<81"

# Hydra patch
/data/sam-3d-objects/.venv/bin/python /data/sam-3d-objects/patching/hydra

# System dep
sudo apt install libc++1
```

## Running

```bash
# Required env vars
export LIDRA_SKIP_INIT=true
export CONDA_PREFIX=/usr/local/cuda-13.2
export CUDA_HOME=/usr/local/cuda-13.2
```

```python
import sys, numpy as np
sys.path.append('notebook')
from inference import Inference, load_image
from PIL import Image

inference = Inference('checkpoints/hf/pipeline.yaml', compile=False)

image = load_image('path/to/image.png')
mask = np.array(Image.open('path/to/mask.png')) > 0

output = inference._pipeline.run(
    inference.merge_mask_to_rgba(image, mask),
    None,
    seed=42,
    stage1_only=False,
    with_mesh_postprocess=False,
    with_texture_baking=False,
    with_layout_postprocess=False,
    use_vertex_color=True,
    stage1_inference_steps=None,
    decode_formats=['gaussian'],  # skip mesh decoder to avoid OOM on 24GB
)
output['gs'].save_ply('output.ply')
```

## Inference Results

### Demo (kidsroom, object index 14)
- Input: 4480x6720 RGB
- Output: `/data/sam-3d-objects/splat.ply` (55 MB)
- VRAM: 13.8 GB peak
- Time: ~45 seconds

### Red cube
- Input: 1536x2048 RGB, mask from color segmentation
- Output: `assets/table_objects/red_cube/red_cube.ply` (72 MB)
- VRAM: 13.8 GB peak
- Time: ~30 seconds

### Memory notes
- Model load: 13.7 GB VRAM
- Gaussian-only decode fits on RTX 3090 (24 GB)
- Mesh decode OOMs — use `decode_formats=['gaussian']` to skip it
- The `.ply` Gaussian splat can be converted to mesh separately if needed

## Checkpoints

Downloaded via `huggingface-hub` CLI to `/data/sam-3d-objects/checkpoints/hf/` (~12 GB).

## Disk Impact

- Venv: ~13 GB on /data
- Checkpoints: ~12 GB on /data
- CUDA 12.1 toolkit: ~7 GB on /data (only needed for future rebuilds)
- uv cache: redirected to /data/.cache/uv
- Freed ~55 GB from root by deleting ROCm PyTorch entries from uv cache
