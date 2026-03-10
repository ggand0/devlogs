# Depth Recording Implementation in LeRobot

Implementation of optional depth capture for RealSense cameras in the LeRobot fork (`feat/depth-recording` branch off `so101`).

## What Changed (4 files)

### 1. `cameras/realsense/camera_realsense.py`

**Problem:** The async read loop only captured color. `read_depth()` existed but was synchronous-only, and calling it separately would hit `try_wait_for_frames()` a second time, getting a different frameset (desynchronized from color).

**Solution:** Added `_read_color_and_depth(timeout_ms)` that calls `try_wait_for_frames()` once and extracts both color and depth from the same frameset. Modified `_read_loop()` to use it, storing both `latest_frame` and `latest_depth_frame` under the same lock.

Added `async_read_depth()` that returns the stored depth frame directly (no event wait — it's called after `async_read()` which already ensured a frame was captured, and both were written in the same lock section).

```python
# In _read_loop:
color_image, depth_map = self._read_color_and_depth(timeout_ms=500)
with self.frame_lock:
    self.latest_frame = color_image
    if depth_map is not None:
        self.latest_depth_frame = depth_map
```

`async_read()` is unchanged — still returns color only.

### 2. `robots/so101_follower/so101_follower.py`

`_cameras_ft` now checks `getattr(cam_config, "use_depth", False)` and adds `depth_{cam}: (H, W, 1)` alongside the RGB entry. `get_observation()` calls `cam.async_read_depth()` and reshapes `(H, W)` → `(H, W, 1)` via `np.newaxis`.

Duck-typing (`getattr`) avoids importing `RealSenseCameraConfig` — works for any camera that has `use_depth`.

### 3. `datasets/image_writer.py`

`write_image()` detects `np.uint16` dtype and writes directly via `cv2.imwrite()` as 16-bit PNG. This bypasses `image_array_to_pil_image()` entirely (PIL's uint16 PNG support is unreliable). Squeezes `(H, W, 1)` → `(H, W)` before writing.

### 4. `datasets/utils.py`

`hw_to_dataset_features()` forces `dtype: "image"` for keys starting with `"depth_"`, regardless of the `use_video` flag. This prevents video encoding (8-bit lossy codecs would destroy 16-bit depth precision).

## What Didn't Need to Change

- **`RealSenseCameraConfig`** — `use_depth: bool = False` already existed
- **`validate_feature_image_or_video()`** — Already handles `(H, W, 1)` shapes correctly
- **`add_frame()` / `save_episode()`** — depth uses `dtype: "image"`, which follows the existing PNG storage path
- **`build_dataset_frame()`** — Already handles `dtype: "image"` features
- **`record.py`** — No changes needed at the recording loop level

## On-disk Layout

```
dataset/
├── videos/chunk-000/
│   └── observation.images.wrist/
│       └── episode_000000.mp4          # RGB (H.264/AV1, as before)
├── images/
│   └── observation.images.depth_wrist/
│       └── episode_000000/
│           ├── frame_000000.png        # 16-bit PNG, uint16, (H, W)
│           └── ...
├── data/chunk-000/
│   └── episode_000000.parquet          # metadata + embedded depth PNG bytes
```

## Storage Budget

At 848x480 depth @ 30fps, ~300 KB/frame compressed:
- 10 sec episode: ~90 MB depth + ~5 MB RGB video
- 75 episodes (~5 min): ~2.7 GB depth + ~150 MB RGB
- Depth dominates because per-frame PNG vs temporally-compressed video

## Sync Guarantee

Color and depth are extracted from the same `try_wait_for_frames()` call in `_read_color_and_depth()`. RealSense hardware delivers synchronized framesets — both frames share the same hardware timestamp.

## Reading Back Depth

```python
import cv2
# IMREAD_UNCHANGED preserves uint16 — default would truncate to uint8
depth = cv2.imread("frame_000042.png", cv2.IMREAD_UNCHANGED)  # uint16, shape (H, W)
depth_meters = depth.astype(np.float32) * 0.001  # D405 default scale: 1mm per unit
```

## Depth Quality: D405 Sensor vs Neural Estimation

### Test Recording

Recorded 3 episodes (5 sec each, 30fps) with `configs/test_depth.yaml` (`wrist_cam_use_depth: true`). Dataset at `~/.cache/huggingface/lerobot/gtgando/so101_depth_test/`.

- 640x480 @ 30fps
- D405 depth: uint16, ~80KB/frame avg (smaller than 300KB estimate — close-range tabletop compresses well)
- 145-147 frames per episode

### D405 Raw Depth Issues

**Black regions (value=0):** No depth return. The D405 has a minimum range of ~7cm. Gripper fingers are only a few cm from the lens — below minimum range. Fingertips at ~70mm show faint readings.

**Stereo quantization artifacts:** Visible structured "banding" patterns on smooth surfaces (cube, fingers, table). At 700-1600mm distance, each stereo disparity step maps to several mm, creating discrete depth bands. This is NOT random noise — it's the fundamental resolution limit of stereo IR depth at these distances.

**Post-processing filters tested:**
- Bilateral filter (d=7, sigmaColor=30, sigmaSpace=5): Nearly invisible difference — noise is only 1-3mm, sub-pixel in the 960mm visualization range
- Stronger median + bilateral (d=9, sigmaColor=150, sigmaSpace=10): 58% of pixels changed, max diff 1636mm, but visually marginal — the structured banding patterns persist because they're quantization, not noise

### Depth Anything V2 Comparison

Ran Depth Anything V2 Small (via transformers pipeline, CPU) on the same RGB frames for comparison.

Side-by-side images: `depth_estimated_viz/episode_000000/` (RGB | D405 | Depth Anything V2, every 10th frame).

**Key finding: neural depth estimation produces dramatically smoother depth maps.** The network acts as a natural regularizer — no stereo disparity quantization, no minimum-range black regions on the gripper, smooth contours on all surfaces.

| Property | D405 Sensor | Depth Anything V2 |
|---|---|---|
| Surface smoothness | Stereo banding artifacts on flat surfaces | Smooth everywhere |
| Metric accuracy | Yes (uint16 mm) | No (relative depth only) |
| Min range | ~7cm (fingers = black) | No limit (estimates depth for everything) |
| Gripper depth | Mostly missing (too close) | Present and smooth |
| Latency | Zero (hardware) | ~1-2s CPU / ~20ms GPU |
| VRAM | Zero | ~400MB (Small) |

**This explains SpatialVLA's design choice:** they use ZoeDepth (similar monocular estimator) for Ego3D Position Encoding rather than sensor depth. Smooth spatial embeddings are more useful for vision transformers than noisy-but-metric stereo depth. The entire system is designed around estimated depth.

### Implications

- **For SpatialVLA fine-tuning:** ZoeDepth at runtime is the right call. No need to feed sensor depth.
- **For inference VRAM savings:** Replacing ZoeDepth with D405 sensor depth saves ~1-2GB VRAM but introduces noisier spatial embeddings. May or may not hurt accuracy — worth ablating.
- **For other projects (salt, future RL):** Sensor depth is still valuable where metric accuracy matters (contact detection, safety bounds, sim-to-real transfer). Neural estimation is better for visual feature extraction.
- **Depth fusion (sensor + estimated):** An interesting middle ground — use neural estimation for smooth contours and sensor depth for metric scale alignment. See note below.

### Depth Fusion: Smooth Neural Structure + Metric Sensor Scale

The idea: use neural depth estimation for smooth contours/surfaces, then align to metric scale using D405 sensor readings. This is a well-studied problem with several practical approaches.

**Scale-and-shift alignment (simplest, sub-ms)**

Fit `aligned = s * mono_depth + t` to match sensor depth via least squares on valid D405 pixels. VI-Depth (Intel ISL, ICRA 2023) showed this works with as few as 150 sparse metric points. Add RANSAC for robustness to D405 outliers (flying pixels, reflections).

```python
def align_depth(mono_depth, sensor_depth, valid_mask):
    d_mono = mono_depth[valid_mask]
    d_sensor = sensor_depth[valid_mask]
    A = np.stack([d_mono, np.ones_like(d_mono)], axis=1)
    s, t = np.linalg.lstsq(A, d_sensor, rcond=None)[0]
    return s * mono_depth + t
```

Limitation: single global scale+shift assumes linear relationship. Can fail with large depth range or non-linear sensor distortion. Region-aware variants (per-segment scale+shift using SAM) address this.

**PromptDA — Prompt Depth Anything (CVPR 2025)**

Built by the Depth Anything team for exactly this use case. Takes RGB + sparse/noisy sensor depth as "prompts" fused into the DPT decoder. Small variant (25.1M params) should run at 30fps with TensorRT. Architecturally designed for sensor fusion rather than post-hoc alignment.

**Prior Depth Anything (May 2025)**

Coarse-to-fine: kNN-based alignment of sparse sensor points to monocular depth, then a conditioned MDE network refines. Handles noisy/incomplete sensor depth well but ~157ms/frame on A100 (~6fps) — kNN bottleneck (123ms) needs GPU optimization for real-time.

**MetricAnything (Jan 2026)**

Teacher model accepts image + sparse metric depth prompt (masked depth maps). Trained on 20M image-depth pairs from diverse noisy sources. State-of-the-art on depth completion, but 876M params — not real-time.

**Real-time feasibility on RTX 3090 (alongside VLA policy)**

| Method | Est. latency | 30fps? |
|---|---|---|
| DA V2 Small + scale/shift | ~5-8ms | Yes |
| DA V2 Base + scale/shift | ~10-15ms | Yes |
| PromptDA Small | ~10-20ms | Likely |
| Prior Depth Anything | ~200ms+ | No |
| MetricAnything Teacher | ~100ms+ | No |

DA V2 + global scale/shift is the pragmatic starting point. PromptDA Small is the most promising purpose-built option if it hits 30fps.

None of ZoeDepth, Metric3D, UniDepth, or Depth Pro support sparse depth input natively — they're RGB-only metric estimators. To use them with sensor depth, post-hoc scale+shift alignment is the only option.

**Key references:**
- [VI-Depth (ICRA 2023)](https://github.com/isl-org/VI-Depth) — scale+shift alignment baseline
- [PromptDA (CVPR 2025)](https://github.com/DepthAnything/PromptDA) — sparse depth as decoder prompts
- [Prior Depth Anything (2025)](https://github.com/SpatialVision/Prior-Depth-Anything) — coarse-to-fine fusion
- [MetricAnything (2026)](https://github.com/metric-anything/metric-anything) — universal metric depth with depth prompts
- [DA V2 TensorRT benchmarks](https://github.com/spacewalk01/depth-anything-tensorrt) — real-time feasibility

## Branch

`feat/depth-recording` on the lerobot fork, branched from `so101` at commit `4c499ada`.
