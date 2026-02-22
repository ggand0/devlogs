# Recording Resolution Fix

## Problem

HIL-SERL training failed with image size mismatch error:
```
RuntimeError: Sizes of tensors must match except in dimension 1. Expected size 84 but got size 128
```

The offline replay buffer was cached with 84x84 images, but the training config expected 128x128.

## Root Cause

The reward classifier dataset (`gtgando/so101_reach_grasp_cube_reward`) was recorded with `resize_size: [84, 84]` in the recording config. This resized the camera's native 640x480 frames down to 84x84 at capture time, destroying the original resolution.

**Recording config before fix** (`reward_classifier_record_config.json`):
```json
"resize_size": [84, 84],
"features": {
    "observation.images.gripper_cam": {
        "shape": [3, 84, 84]
    }
}
```

This meant:
1. Camera captures at 640x480
2. Wrapper immediately resizes to 84x84
3. 84x84 frames saved to dataset
4. Original 640x480 frames are lost forever

## Why This Is Wrong

Resizing at recording time is destructive - you can never recover the original resolution. The correct approach:
1. Record at native camera resolution (640x480)
2. Resize to training resolution (128x128) at runtime during training/inference

This preserves flexibility to experiment with different training resolutions without re-recording.

## Fix Applied

### 1. Recording Config (`reward_classifier_record_config.json`)

Changed to record at native 640x480:
```json
"resize_size": null,
"features": {
    "observation.images.gripper_cam": {
        "shape": [3, 480, 640]
    }
}
```

### 2. Training Config (`reach_grasp_hilserl_train_config.json`)

Set `target_entropy: -2.0` (see devlog 063 for details on this fix).

Training config already had `resize_size: [128, 128]` which will resize 640x480 frames to 128x128 at runtime.

## Action Required

The reward classifier dataset needs to be re-recorded at 640x480 resolution using the updated recording config.

## Lesson Learned

Always record datasets at the highest available resolution. Downscaling can be done at training time, but upscaling cannot recover lost information.
