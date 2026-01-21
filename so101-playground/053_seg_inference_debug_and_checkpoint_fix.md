# Devlog 053: Segmentation Inference Debug and Checkpoint Fix

## Problem

Segmentation inference was working perfectly in `seg_depth_preview.py` but producing noisy/messy output in `rl_inference_seg_depth.py` despite using identical model loading code.

## Investigation

Added debug logging at inference time to both scripts to compare:
- Raw frame stats (shape, dtype, range, mean BGR)
- Cropped frame stats
- Segmentation mask output (shape, class distribution)
- Disparity output (shape, range)
- Resized output stats

Debug output showed the model loading was identical between scripts, so the issue had to be elsewhere.

## Root Cause

**Checkpoint mismatch**: The user was running `rl_inference_seg_depth.py` with a different segmentation checkpoint than `seg_depth_preview.py`.

- Working checkpoint: `/home/gota/ggando/ml/pick-101/outputs/efficientvit_seg_merged/best-v1.ckpt`
- Noisy checkpoint: `/home/gota/ggando/ml/pick-101/outputs/efficientvit_seg/best.ckpt`

Both scripts have the same default (`efficientvit_seg_merged/best-v1.ckpt`), but the user was explicitly passing the non-merged checkpoint to the inference script.

## Solution

Use the merged checkpoint (`efficientvit_seg_merged/best-v1.ckpt`) which was trained on more diverse data and produces cleaner segmentation.

## Changes

### `scripts/rl_inference_seg_depth.py`
- Added detailed inference debug logging with `--debug_seg` flag
- Logs raw frame, cropped frame, seg mask, disparity, and resized output stats for first 3 steps

### `scripts/seg_depth_preview.py`
- Added matching inference debug logging with `--debug_seg` flag
- Enables side-by-side comparison of inference behavior between scripts

## Usage

Run with debug logging to diagnose segmentation issues:

```bash
uv run python scripts/rl_inference_seg_depth.py \
    --checkpoint /path/to/policy.pt \
    --genesis_mode --camera_index 1 \
    --debug_seg
```

Example debug output:
```
[INFERENCE DEBUG] Step 0
    Raw frame: shape=(480, 640, 3), dtype=uint8
    Raw frame range: [20, 215]
    Raw frame mean BGR: [123.9, 117.4, 120.9]
    Cropped: shape=(480, 480, 3), y_start=0, x_start=80, size=480
    Cropped mean BGR: [119.3, 111.8, 118.2]
    Seg mask: shape=(480, 480), dtype=uint8
    Seg class distribution: {1: 14134, 2: 184404, 3: 25378, 4: 4677, 5: 1807}
    Disparity: shape=(480, 480), dtype=uint8, range=[0, 255]
    Seg resized: shape=(84, 84), classes={1: 452, 2: 5640, 3: 771, 4: 135, 5: 58}
    Disp resized: shape=(84, 84), range=[5, 252]
```

## Lessons Learned

1. When debugging "identical code different results", check ALL inputs including file paths
2. Debug logging at inference time (not just model loading) reveals input/output differences
3. Use the same checkpoint path for all scripts by relying on defaults or copying the exact path
