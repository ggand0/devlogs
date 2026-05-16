# 010 — LeRobot v3.0 Data Recording Setup

2026-05-16

## Overview

Set up LeRobot v3.0 dataset recording in the Isaac Sim teleop pipeline. Goal: collect sim teleoperation data in the same format used by the real SO-101 robot so datasets can be mixed for real2sim policy training.

## LeRobot Version

- **Target:** lerobot 0.4.4 (PyPI stable, v3.0 dataset format)
- **Local lerobot checkout:** `/home/gota/ggando/ml/lerobot` is v0.3.2 (v2.1 format) with custom SO-101 patches — left untouched for real robot work
- **Sim install:** Separate venv at `/data/lerobot-v3/.venv` to avoid dependency conflicts with Isaac Sim's Python environment. Imported via `sys.path` insertion in `sim_teleop.py`.

### Why separate venv

Isaac Sim bundles its own Python (3.10) with specific numpy, torch, and USD library versions. Installing lerobot directly into the Isaac Sim env risks version conflicts (lerobot 0.4.4 pulls torch 2.x, numpy 2.x, etc.). The separate venv isolates lerobot and its dependencies. Only the lerobot site-packages path is added to `sys.path` at runtime — Isaac Sim's own packages take priority.

## LeRobot v3.0 Format (vs v2.1)

v3.0 was introduced in lerobot v0.4.0 (2025-10-23). Key differences from v2.1:

| Aspect | v2.1 | v3.0 |
|---|---|---|
| Episodes per file | 1 parquet + 1 MP4 each | Multiple episodes concatenated per file |
| File naming | `episode_000000.parquet` | `file-000.parquet` (size-based rolling) |
| Episode metadata | `episodes.jsonl` | `meta/episodes/chunk-NNN/file-NNN.parquet` |
| Task metadata | `tasks.jsonl` | `meta/tasks.parquet` |
| Stats | `episodes_stats.jsonl` | `meta/stats.json` |
| Video path | `videos/chunk-NNN/{camera}/episode_NNN.mp4` | `videos/{camera}/chunk-NNN/file-NNN.mp4` |
| Size thresholds | N/A | 100 MB parquet, 200 MB MP4 before rolling |
| Finalization | Implicit | Must call `dataset.finalize()` |

v3.0 is NOT backward compatible with v2.1. Conversion script exists: `lerobot.datasets.v30.convert_dataset_v21_to_v30`.

### v3.0 File Layout

```
dataset_root/
  meta/
    info.json
    stats.json
    tasks.parquet
    subtasks.parquet
    episodes/
      chunk-000/
        file-000.parquet
  data/
    chunk-000/
      file-000.parquet        # rows from multiple episodes
  videos/
    observation.images.wrist/
      chunk-000/
        file-000.mp4          # frames from multiple episodes
    observation.images.overhead/
      chunk-000/
        file-000.mp4
```

## Recording Architecture

### Approach: LeRobot API in sim_teleop

Use `LeRobotDataset.create()` / `add_frame()` / `save_episode()` / `finalize()` directly. LeRobot handles parquet writing, video encoding (ffmpeg), metadata, and stats computation. This guarantees format compatibility — the v3.0 multi-episode file concatenation and chunked metadata parquet are complex enough that reimplementing them would be error-prone.

Alternative approaches considered and rejected:
- **Write v3.0 files manually** — high risk of subtle format incompatibilities with the multi-episode concatenation and chunked metadata
- **Record raw, convert post-hoc** — two-step workflow adds friction; no benefit since LeRobot API overhead is minimal

### Feature Schema

Matches the real SO-101 teleop recording format (from `so101_pick_place_demo_org`):

```yaml
observation.state:
  dtype: float32
  shape: [6]
  names: [shoulder_pan, shoulder_lift, elbow_flex, wrist_flex, wrist_roll, gripper]
  units: degrees

action:
  dtype: float32
  shape: [6]
  names: [shoulder_pan, shoulder_lift, elbow_flex, wrist_flex, wrist_roll, gripper]
  units: degrees

observation.images.wrist:
  dtype: video
  shape: [480, 640, 3]

observation.images.overhead:
  dtype: video
  shape: [480, 640, 3]
```

Note: IRL datasets use normalized [-100, 100] motor range. Sim uses raw degrees from the leader arm for simplicity. This difference must be accounted for when mixing real and sim datasets for training.

## Recording Config

YAML config at `configs/recording.yaml` defines all recording parameters:

```yaml
dataset:
  repo_id: "gtgando/sim_so101_pick_place"
  robot_type: "so101"
  task: "Pick up the red cube and place it in the green bowl"
  fps: 30

cameras:
  wrist:
    prim_path: "{robot}/gripper_link/D405_Camera"
    width: 640
    height: 480
  overhead:
    prim_path: "/World/OverheadCamera"
    width: 640
    height: 480

features:
  observation_state:
    - shoulder_pan
    - shoulder_lift
    - elbow_flex
    - wrist_flex
    - wrist_roll
    - gripper
  action:
    - shoulder_pan
    - shoulder_lift
    - elbow_flex
    - wrist_flex
    - wrist_roll
    - gripper
```

## Overhead Camera

Placed programmatically in `sim_teleop.py` when `overhead` is in the camera config.

- **Position:** (0.11, -0.29, 1.315) m — aligned with robot base X/Y, 58cm above table surface (table Z = 0.735 m)
- **Orientation:** Looking straight down (-90 pitch)
- **Resolution:** 640x480 (matching wrist cam and IRL overhead)
- **FOV:** Tuned to cover workspace (~50cm x 50cm at table level)
- **Renderer:** Same Replicator render product approach as wrist cam

IRL overhead cam is mounted ~58cm above the cube placement area on the table.

## Module Structure

```
scripts/
  sim_teleop.py          # main loop — swaps EpisodeRecorder for LeRobotRecorder
  lerobot_recorder.py    # LeRobotRecorder class wrapping LeRobot v3.0 API
  recording_config.py    # YAML config loading
  leader_reader.py       # unchanged
configs/
  recording.yaml         # default recording config
```

## Install Commands

```bash
# Create separate venv for lerobot v3.0
uv venv /data/lerobot-v3/.venv --python 3.10

# Install lerobot 0.4.4
uv pip install --python /data/lerobot-v3/.venv/bin/python lerobot==0.4.4

# Verify
/data/lerobot-v3/.venv/bin/python -c "from lerobot.datasets import LeRobotDataset; print('OK')"
```

## Open Questions

- Video codec: lerobot 0.4.4 prefers AV1 when available, falls back to H.264. Isaac Sim's system may not have AV1 encoder — need to check.
- Whether `sys.path` insertion for lerobot's site-packages causes any import conflicts with Isaac Sim's bundled packages.
- Overhead camera FOV/focal length may need tuning after visual inspection in the sim viewport.
