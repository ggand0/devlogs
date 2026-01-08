# Devlog 026: HIL-SERL Actor-Learner Integration Fixes

## Overview

Fixed numerous issues preventing HIL-SERL training from running end-to-end with the SO-101 robot. This devlog covers fixes for resume logic, MuJoCo integration, wrapper configuration, and state dimension handling.

---

## Fix 1: Resume Logic - No Checkpoint Graceful Handling

### Problem

Learner crashed when `resume=true` but no checkpoint existed:
```
RuntimeError: No model checkpoint found in outputs/hilserl_drqv2_so101/checkpoints/last for resume=True
```

### Solution

Made resume logic graceful - if no checkpoint exists, continue without loading:

**`lerobot/src/lerobot/configs/train.py`:**
```python
elif self.resume:
    config_path = parser.parse_arg("config_path")
    if not config_path:
        if self.output_dir and Path(self.output_dir, TRAIN_CONFIG_NAME).exists():
            config_path = str(Path(self.output_dir, TRAIN_CONFIG_NAME))
        else:
            pass  # No config found, handle gracefully
    if config_path:
        model_file = policy_path / "model.safetensors"
        if model_file.exists():
            self.policy.pretrained_path = policy_path
            self.checkpoint_path = policy_path.parent
```

---

## Fix 2: Model Loading - Check File Exists

### Problem

```
FileNotFoundError: No such file or directory: ./model.safetensors
```

### Solution

Only set `pretrained_path` if model file actually exists (see Fix 1 code above).

---

## Fix 3: Ctrl+C Unresponsive

### Problem

Learner wouldn't respond to Ctrl+C. Had to kill the process forcefully.

### Solution

**`lerobot/src/lerobot/scripts/rl/learner.py`:**
```python
# Changed from time.sleep to event-based wait
if shutdown_event is not None:
    shutdown_event.wait(timeout=0.1)
else:
    time.sleep(0.1)
```

**`lerobot/src/lerobot/utils/process.py`:**
```python
# Force shutdown on second Ctrl+C
if self._counter > 1:
    logging.info("Force shutdown")
    os._exit(1)  # Immediately terminate all threads
```

---

## Fix 4: urdf_path Attribute Error

### Problem

```
AttributeError: 'SO101FollowerEndEffectorConfig' object has no attribute 'urdf_path'
```

SO101FollowerEndEffector uses MuJoCo for IK, not URDF-based kinematics.

### Solution

**`lerobot/src/lerobot/scripts/rl/gym_manipulator.py`:**

Skip EEObservationWrapper for MuJoCo-based robots:
```python
# Only add EE observation wrapper for URDF-based robots
if hasattr(robot_config, 'urdf_path') and robot_config.urdf_path:
    env = EEObservationWrapper(env, ...)
```

---

## Fix 5: State Dimension Mismatch (12 vs 18)

### Problem

```
RuntimeError: The size of tensor a (12) must match the size of tensor b (18)
```

Policy expected 18-dim state (joint_pos + joint_vel + ee_xyz + ee_euler) but environment provided 12-dim.

### Solution

Added `FullProprioceptionWrapper` with MuJoCo FK support:

**`lerobot/src/lerobot/scripts/rl/gym_manipulator.py`:**
```python
class FullProprioceptionWrapper(gym.ObservationWrapper):
    """
    Provides full 18-dim proprioceptive state:
    - joint_pos (6): Joint positions
    - joint_vel (6): Computed from position delta
    - ee_xyz (3): End-effector position from FK
    - ee_euler (3): End-effector orientation from FK
    """
    def __init__(self, env, fps=30, num_dof=6):
        # Initialize kinematics - support both URDF and MuJoCo
        if hasattr(env.unwrapped.robot.config, 'urdf_path') and env.unwrapped.robot.config.urdf_path:
            self.kinematics = RobotKinematics(...)
        elif hasattr(env.unwrapped.robot, 'mj_data'):
            self.use_mujoco_fk = True

    def observation(self, observation):
        joint_pos = observation["agent_pos"][:self.num_dof]
        joint_vel = (joint_pos - self.last_joint_positions) / self.dt

        if self.use_mujoco_fk:
            ee_xyz = robot.mj_data.site_xpos[robot.ee_site_id].copy()
            xmat = robot.mj_data.site_xmat[robot.ee_site_id].reshape(3, 3)
            ee_euler = rotation_matrix_to_euler(xmat)

        full_state = np.concatenate([joint_pos, joint_vel, ee_xyz, ee_euler])
        observation["agent_pos"] = full_state
        return observation
```

Config change:
```json
"add_full_proprioception": true
```

---

## Fix 6: torchcodec FFmpeg Error

### Problem

```
RuntimeError: Could not load libtorchcodec
```

### Solution

Pass `video_backend` from config when loading dataset:

**`lerobot/src/lerobot/scripts/rl/learner.py`:**
```python
offline_dataset = LeRobotDataset(
    repo_id=cfg.dataset.repo_id,
    root=dataset_offline_path,
    video_backend=cfg.dataset.video_backend,  # Added this
)
```

---

## Fix 7: Auto-Save Config to Output Dir

### Problem

Actor required manual copying of config to output dir. User complained: "it should be automated bitch"

### Solution

Auto-save config on learner startup:

**`lerobot/src/lerobot/scripts/rl/learner.py`:**
```python
# Save config to output dir for actor to use
config_save_path = os.path.join(cfg.output_dir, "train_config.json")
if not os.path.exists(config_save_path):
    import json
    with open(config_save_path, "w") as f:
        json.dump(cfg.to_dict(), f, indent=4, default=str)
```

---

## Fix 8: Camera Device Error

### Problem

```
ConnectionError: Failed to open OpenCVCamera(/dev/video0)
```

### Solution

Changed to correct camera device after testing with cv2:
```json
"cameras": {
    "gripper_cam": {
        "type": "opencv",
        "index_or_path": "/dev/video1",
        ...
    }
}
```

---

## Fix 9: Image Size Mismatch (20 vs 3)

### Problem

```
RuntimeError: The size of tensor a (20) must match the size of tensor b (3) at non-singleton dimension 3
```

`crop_params_dict` was null but images still needed resizing.

### Solution

Added `ImageResizeWrapper` for resize without crop:

**`lerobot/src/lerobot/scripts/rl/gym_manipulator.py`:**
```python
class ImageResizeWrapper(gym.Wrapper):
    """Wrapper that resizes image observations without cropping."""
    def __init__(self, env, resize_size):
        super().__init__(env)
        self.resize_size = tuple(resize_size)
        for key in list(self.observation_space.keys()):
            if "image" in key:
                new_shape = (3, self.resize_size[0], self.resize_size[1])
                self.observation_space[key] = gym.spaces.Box(...)

    def _resize_images(self, obs):
        for key in obs:
            if "image" in key:
                img = torch.from_numpy(obs[key]).float()
                img = F.interpolate(img.unsqueeze(0), size=self.resize_size, ...)
                obs[key] = img.squeeze(0).numpy().astype(np.uint8)
        return obs
```

---

## Fix 10: Annoying Voice Sounds

### Problem

`log_say()` calls with `play_sounds=True` were playing annoying audio during training.

### Solution

Changed all `play_sounds=True` to `play_sounds=False` in gym_manipulator.py (7 occurrences):
- Manual reset announcements
- Human intervention step
- Continuing with policy actions
- Intervention started/ended
- Recording episode

---

## Fix 11: Offline Buffer State Dimension Mismatch (6 vs 18)

### Problem

```
RuntimeError: The size of tensor a (6) must match the size of tensor b (18) at non-singleton dimension 0
```

Offline buffer was built from dataset with 6-dim state, but actor sends 18-dim states (with full proprioception).

### Root Cause

Dataset only has `observation.state` with 6-dim joint positions. When `add_full_proprioception=True`, actor sends 18-dim states but offline buffer expects 6-dim.

### Solution

Added full proprioception computation to `ReplayBuffer.from_lerobot_dataset()`:

**`lerobot/src/lerobot/utils/buffer.py`:**
```python
@classmethod
def from_lerobot_dataset(
    cls,
    ...,
    compute_full_proprioception: bool = False,
    mujoco_model_path: str | None = None,
    ee_site_name: str = "gripper",
    fps: float = 30.0,
):
    # Initialize MuJoCo for FK
    if compute_full_proprioception:
        import mujoco
        mj_model = mujoco.MjModel.from_xml_path(mujoco_model_path)
        mj_data = mujoco.MjData(mj_model)
        ee_site_id = mujoco.mj_name2id(mj_model, mujoco.mjtObj.mjOBJ_SITE, ee_site_name)

    # In the conversion loop:
    if compute_full_proprioception and key == "observation.state":
        joint_pos = val.numpy()
        joint_pos_rad = np.deg2rad(joint_pos)

        # Compute velocity (finite difference)
        joint_vel = (joint_pos - prev_joint_pos) / dt

        # Compute FK
        mj_data.qpos[:num_dof] = joint_pos_rad
        mujoco.mj_forward(mj_model, mj_data)
        ee_xyz = mj_data.site_xpos[ee_site_id].copy()
        ee_euler = rotation_matrix_to_euler(xmat)

        # Concatenate to 18-dim
        full_state = np.concatenate([joint_pos, joint_vel, ee_xyz, ee_euler])
        val = torch.from_numpy(full_state).float()
```

**`lerobot/src/lerobot/scripts/rl/learner.py`:**
```python
# Get full proprioception config
compute_full_proprioception = getattr(cfg.env.wrapper, "add_full_proprioception", False)

if compute_full_proprioception:
    mujoco_model_path = getattr(cfg.env.robot, "mujoco_model_path", None)
    ee_site_name = getattr(cfg.env.robot, "end_effector_site", "gripper")

offline_replay_buffer = ReplayBuffer.from_lerobot_dataset(
    ...,
    compute_full_proprioception=compute_full_proprioception,
    mujoco_model_path=mujoco_model_path,
    ee_site_name=ee_site_name,
    fps=fps,
)
```

**Important:** Delete `outputs/hilserl_drqv2_so101/offline_buffer.pt` to force re-conversion with proper 18-dim state.

---

## Commits

| Hash | Description |
|------|-------------|
| `c7dfd62d` | Fix HIL-SERL resume logic and MuJoCo robot support |
| `a68fbc4a` | Fix offline dataset loading to use video_backend from config |
| `86a0aa01` | Add ImageResizeWrapper and auto-save config to output dir |
| `ded99c07` | Disable play_sounds in gym_manipulator log_say calls |
| `62ebd4e4` | Compute full proprioception when converting offline dataset |

---

## Files Changed

- `lerobot/src/lerobot/configs/train.py` - Resume logic
- `lerobot/src/lerobot/scripts/rl/learner.py` - Buffer init, config auto-save, full proprioception
- `lerobot/src/lerobot/scripts/rl/gym_manipulator.py` - Wrappers, play_sounds, MuJoCo FK
- `lerobot/src/lerobot/utils/buffer.py` - Full proprioception in from_lerobot_dataset
- `lerobot/src/lerobot/utils/process.py` - Force shutdown with os._exit
- `train_hilserl_drqv2.json` - Camera device, add_full_proprioception

---

## Usage

```bash
# Delete old cache if state dimension changed
rm outputs/hilserl_drqv2_so101/offline_buffer.pt

# Run learner (first terminal)
cd /home/gota/ggando/ml/lerobot
uv run python -m lerobot.scripts.rl.learner --config_path /path/to/train_hilserl_drqv2.json

# Run actor (second terminal)
uv run python -m lerobot.scripts.rl.actor --config_path /path/to/train_hilserl_drqv2.json
```
