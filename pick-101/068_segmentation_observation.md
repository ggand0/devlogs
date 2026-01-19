# 068: Segmentation Observation for Sim-to-Real

## Goal

Replace RGB images with semantic segmentation masks to eliminate visual domain gap between simulation and real robot.

## Approach

Instead of training on RGB (where textures/lighting differ between sim and real), train on segmentation masks that look identical in both domains.

## 5-Class Segmentation

| Class | ID | Color | Description |
|-------|-----|-------|-------------|
| Background | 0 | Dark gray | Arm, sky, walls |
| Ground | 1 | Tan | Table/floor surface |
| Cube | 2 | Red | Target object |
| Static finger | 3 | Green | Fixed gripper jaw |
| Moving finger | 4 | Magenta | Movable gripper jaw |

## MuJoCo Segmentation Rendering

MuJoCo 3.x provides native segmentation via `renderer.enable_segmentation_rendering()`:

```python
renderer = mujoco.Renderer(model, height=480, width=640)
renderer.enable_segmentation_rendering()
renderer.update_scene(data, camera="wrist_cam")
seg = renderer.render()  # (H, W, 2): [geom_id, geom_type]
```

## Geom ID Mapping

Key finding: finger pads (geom 29, 32) are collision-only boxes, not rendered. Must use visible mesh geoms instead:

| Part | Geom IDs | Body |
|------|----------|------|
| Floor | 0 | world |
| Static finger | 25, 26, 27, 28, 29 | gripper |
| Moving finger | 30, 31, 32 | moving_jaw_so101_v1 |
| Cube | 33 | cube |

## SegmentationWrapper

Added to `src/training/so101_factory.py`:

```python
class SegmentationWrapper(gym.ObservationWrapper):
    """Outputs segmentation mask instead of RGB."""

    NUM_CLASSES = 5
    STATIC_FINGER_GEOM_IDS = [25, 26, 27, 28, 29]
    MOVING_FINGER_GEOM_IDS = [30, 31, 32]

    def observation(self, obs):
        seg = self._renderer.render()
        geom_ids = seg[:, :, 0]

        class_map = np.zeros_like(geom_ids, dtype=np.uint8)
        class_map[geom_ids == 0] = 1   # floor
        class_map[geom_ids == 33] = 2  # cube
        for gid in self.STATIC_FINGER_GEOM_IDS:
            class_map[geom_ids == gid] = 3
        for gid in self.MOVING_FINGER_GEOM_IDS:
            class_map[geom_ids == gid] = 4

        # Center crop + resize to 84x84
        class_map = cv2.resize(class_map, (84, 84), interpolation=cv2.INTER_NEAREST)

        return {"rgb": class_map[np.newaxis], "low_dim_state": proprioception}
```

## Usage

Enable via config:

```yaml
env:
  obs_type: seg  # rgb or seg
```

## Output Format

- Shape: `(1, 84, 84)` single-channel class IDs
- Dtype: `uint8`
- Values: 0-4 (class IDs)
- Frame stacking: 3 frames → `(3, 84, 84)`

## Real Robot Pipeline

For real robot, need segmentation model to produce same class masks:

**Option A: Color-based (simplest)**
- Red cube → HSV threshold
- Colored finger tape → HSV threshold
- Table → fixed region or plane detection

**Option B: Trained segmentation model**
- Collect real images
- Label with SAM (SALT tool at `/home/gota/ggando/ml/salt`)
- Train lightweight model (MobileNet U-Net, ~3M params)

## Test Scripts

- `tests/test_segmentation.py` - Static pose visualization
- `tests/test_wrist_cam_seg_video.py` - Wrist cam video demo

## Output

- `outputs/seg_test.png` - 5 poses with RGB | Seg side by side
- `outputs/wrist_cam_seg.mp4` - Video demo

## Commit

`89fa171` on branch `feat/seg-obs`
