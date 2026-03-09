# Depth Recording for SpatialVLA and Future Models

## Motivation

SmolVLA (RGB-only, 450M params) achieves 60-80% pick-and-place but is brittle under lighting changes. SpatialVLA (4B params, PaLiGemma2 backbone) adds Ego3D Position Encoding — it runs ZoeDepth on RGB to produce depth, back-projects to 3D point clouds via camera intrinsics, and encodes spatial positions into vision tokens. This gives explicit 3D scene understanding invariant to lighting (shadows, fluorescents, booth LEDs).

We also want depth recording for a separate project (salt). This devlog captures research on depth storage formats and what needs to change in LeRobot.

## Key Finding: SpatialVLA Doesn't Need Depth in the Dataset

SpatialVLA runs ZoeDepth on RGB **at runtime** — both training and inference. The pipeline:

```
RGB → ZoeDepth → depth map D → back-project with intrinsics (fx, fy, cx, cy) → 3D XYZ per pixel
    → sinusoidal encoding → MLP → Ego3D position embeddings added to SigLIP vision tokens
```

So for SpatialVLA fine-tuning, you only need RGB in RLDS format. ZoeDepth handles depth generation during the forward pass.

**However**, recording real D405 depth is still valuable:
- At inference, real depth can **replace ZoeDepth** — saves ~1-2GB VRAM and reduces latency
- Future models (ACT with depth, diffusion policy, RL) may consume sensor depth directly
- Real metric depth from D405 (7-50cm range) is more accurate than monocular estimation
- Needed independently for salt project

## LeRobot Depth Support Status

LeRobot has partial infrastructure but no end-to-end depth support:

| Component | Status |
|---|---|
| `RealSenseCamera.use_depth` config | Exists, default `False` |
| `RealSenseCamera.read_depth()` | Works (synchronous only) |
| `RealSenseCamera.async_read()` | Color only — depth not implemented |
| Image writer | TODO comment: "handle 1 channel and 4 for depth images" |
| Dataset pipeline | Assumes 3-channel uint8 everywhere |
| Video encoding for depth | Not investigated by the team |

The camera layer can capture depth, but the recording/dataset pipeline can't store or load it.

## Depth Storage Format Survey

How major IL/RL frameworks store depth:

| Framework | Format | Details |
|---|---|---|
| **RLDS / Open X-Embodiment** | 16-bit PNG in TFRecord | `tfds.features.Image(dtype=uint16)`, shape `(H, W, 1)` |
| **Robomimic** | float32 in HDF5 | Normalized [0,1], gzip compression |
| **DROID** | ZED SVO files (raw) | Stereo-computed depth, stripped in RLDS conversion |
| **ROS** | 16-bit PNG or float32 | Standard `sensor_msgs/Image` encoding |
| **LeRobot** | Not supported | RGB only (MP4 video or JPEG/PNG fallback) |

Nobody encodes depth as video — H.264/H.265 are 8-bit per channel, which destroys 16-bit precision.

### Format Comparison

| Format | Bit depth | Compression | Size/frame (848x480) | Lossless? |
|---|---|---|---|---|
| **16-bit PNG** | uint16 | DEFLATE (lossless) | ~200-400 KB | Yes |
| float32 HDF5 | float32 | gzip (lossless) | ~400-800 KB | Yes |
| float32 .npy | float32 | None | 1.6 MB | Yes |
| Raw uint16 | uint16 | None | 815 KB | Yes |
| H.264 video | uint8 | Lossy temporal | N/A | **No** |

## Recommendation: 16-bit PNG Per Frame

### How It Works

The D405 outputs a **Z16 depth frame** — a 2D array of `uint16` values where each pixel = distance in millimeters (default scale 0.001m). Pixel value `342` = 342mm = 34.2cm. Range is 0–65535mm (~65m), well beyond D405's useful 7–50cm.

A 16-bit PNG stores this uint16 array as a single-channel grayscale image with lossless DEFLATE compression. Each PNG file = one frame of depth data.

**Recording:**
```python
import pyrealsense2 as rs
import cv2
import numpy as np

# Get depth frame from D405
depth_frame = frames.get_depth_frame()
depth_array = np.asanyarray(depth_frame.get_data())  # uint16, shape (480, 848)

# Save as 16-bit PNG — lossless, ~300KB
cv2.imwrite("depth_000042.png", depth_array)
```

**Loading:**
```python
# IMREAD_UNCHANGED is critical — default reads as 8-bit, losing precision
depth_uint16 = cv2.imread("depth_000042.png", cv2.IMREAD_UNCHANGED)  # uint16
depth_meters = depth_uint16.astype(np.float32) * 0.001  # convert to meters
```

### Why 16-bit PNG

- **Lossless**: exact uint16 values preserved, no quantization or temporal artifacts
- **Industry standard**: RLDS/OXE, ROS, and most real-robot datasets use this
- **Good compression**: depth maps compress well (smooth surfaces, uniform regions) — ~2-4x over raw
- **Universal tooling**: OpenCV, PIL, ffmpeg, numpy all handle 16-bit PNG
- **Compatible with RLDS conversion**: `tfds.features.Image(dtype=np.uint16)` expects this

### Storage Budget

At 848x480 @ 30fps with ~300 KB/frame average:

| Duration | Depth storage | RGB video (H.264) | Total |
|---|---|---|---|
| 10 sec episode | ~90 MB | ~5 MB | ~95 MB |
| 75 episodes (~5 min total) | ~2.7 GB | ~150 MB | ~2.85 GB |
| 1 hour recording | ~32 GB | ~1.8 GB | ~34 GB |

Depth dominates storage since it's per-frame PNG vs temporally-compressed video. Acceptable for our dataset sizes (50-100 episodes).

## What Needs to Change in LeRobot

To add depth recording to the pipeline:

1. **Camera layer** (`cameras/realsense/camera_realsense.py`):
   - Implement depth in `async_read()` — return paired RGB + depth from synchronized frameset
   - Or add `async_read_depth()` that returns uint16 array

2. **Robot observation_features** (`robots/so101_follower/so101_follower.py`):
   - Support `(H, W, 1)` shapes for depth alongside `(H, W, 3)` for RGB
   - Return depth frames from `get_observation()` keyed as `depth_{camera_name}`

3. **Image writer** (`datasets/image_writer.py`):
   - Handle uint16 single-channel arrays → 16-bit PNG
   - The existing TODO: `# TODO(aliberts): handle 1 channel and 4 for depth images`

4. **Dataset features** (`datasets/utils.py`):
   - `hw_to_dataset_features()` must recognize 1-channel image features
   - Convention: `observation.images.depth_{camera_name}` (e.g., `observation.images.depth_wrist`)
   - Depth features should never use video encoding — always individual PNGs

5. **Recording pipeline** (`record.py`):
   - No video encoding for depth channels
   - Store as image sequence alongside video-encoded RGB

## D405 Camera Specs (Relevant)

| Property | Value |
|---|---|
| Depth format | Z16 (uint16, millimeters) |
| Optimal resolution | 848x480 |
| Optimal range | 7-50cm |
| Max FPS | 90 fps @ 848x480 |
| Depth scale | 0.001m (1mm per unit, configurable) |
| Serial | `335122272499` |
| Intrinsics | Available via `pyrealsense2` (`fx, fy, cx, cy`) |

## Next Steps

1. Implement depth recording in LeRobot (camera → dataset pipeline)
2. Record new dataset with RGB + depth from D405
3. For SpatialVLA: convert to RLDS (depth optional — ZoeDepth handles it)
4. For inference optimization: swap ZoeDepth with real D405 depth to save VRAM
