# Devlog 039: Moving Finger Visibility Issue in Image-Based RL

## Discovery

During image-based RL training, the agent learned to use only the static finger to tilt/push the cube instead of grasping with both fingers. Investigation revealed a potential partial observability issue.

## The Problem

At stage 3 curriculum reset, the gripper is at ~1.07 rad (61°), which is quite open. From the wrist camera's perspective:

| Gripper Angle | Degrees | Both Fingers Visible? |
|---------------|---------|----------------------|
| 1.07 rad | 61° | Moving finger barely/not visible |
| 0.79 rad | 45° | Starting to appear |
| 0.40 rad | 23° | Both visible |
| 0.00 rad | 0° | Both clearly visible |

**The agent might not realize it has a second (moving) finger because it's not visible in the initial frames.**

## Sim vs Real Camera FOV

### Simulation Camera (MuJoCo)
- FOV: 75° (fovy parameter)
- Position: Front of gripper, looking toward fingertips

### Real Camera (innoMaker USB Camera)
- Diagonal FOV: **130°**
- Horizontal FOV: **103°**
- Resolution: 1080p @ 30fps
- Size: 32x32mm

**The real camera has 1.7x wider FOV than simulation!** This means on real hardware, the moving finger would likely be visible even when open.

## Camera Specs (Real Hardware)

```
innoMaker 1080P USB2.0 UVC Camera
- Compatibility: Windows, Linux, Android, Mac OS, Raspberry Pi, Jetson Nano
- Resolution: 1080P x 30fps
- Formats: YUY2/MJPEG
- Diagonal FOV: 130°
- Horizontal FOV: 103°
- Size: 32 x 32 mm
- Interface: USB 2.0 UVC standard
```

## Potential Fixes

### 1. Match Real Camera FOV
Update sim camera to match real hardware:
```xml
<camera name="wrist_cam" pos="0.0 -0.055 0.02" euler="0 0 3.14159" fovy="100"/>
```
(Note: MuJoCo fovy is vertical FOV; 100° vertical ≈ 130° diagonal for 4:3 aspect)

### 2. Start with Gripper More Closed
Modify stage 3 reset to close gripper to ~0.4 rad (23°) so both fingers are initially visible.

### 3. Use Curriculum Stage 2 First
Stage 2 starts with cube already in gripper (closed), so agent sees both fingers from the start. Then transfer to stage 3.

## Implications

- Agent trained with narrow FOV may not transfer well to real hardware
- The "exploit" of using only static finger might be a consequence of partial observability
- v13 reward fix addresses reward hacking, but doesn't fix the visual observability issue

## Test Images

Saved to `runs/`:
- `wrist_cam_stage3_reset.png` - View at stage 3 reset (61°)
- `gripper_45deg.png` - View at 45°
- `gripper_partial.png` - View at 23°
- `gripper_closed.png` - View when fully closed

## Next Steps

1. Continue current v13 training to 500k-1M to see if reward fix alone helps
2. If still failing, try increasing camera FOV to match real hardware
3. Consider curriculum from stage 2 → stage 3

## References

- Devlog 038: Image RL with camera fix (v11 exploit behavior)
- Real camera: Amazon innoMaker 1080P USB Camera
