# Teleop Lag Fix Implementation

Branch: `fix/teleop-lag` off `so101` in lerobot repo

## Root Cause Analysis

Two independent sources of blocking compounded in the record loop:

### 1. Per-frame camera synchronization (30-66ms/iteration)

`async_read()` clears `new_frame_event` after each read, forcing the next call to block on `new_frame_event.wait()` until the background thread delivers a fresh frame. At 30fps that's up to 33ms per camera. With 2 cameras read sequentially in `get_observation()`, that's 30-66ms of camera-induced blocking — eating the entire 33.3ms frame budget before any motor command is sent.

### 2. Force-reconnect in the main thread (1-3s stalls)

When `async_read()` times out (200ms) and detects the background thread stuck >0.5s, it calls `_force_reconnect()` in the main record loop thread:

```
_force_reconnect():
  thread.join(timeout=0.5s)                          → 0.5s
  _find_working_camera() scans ALL /dev/video*       → 2-5s
  VideoCapture(device) + configure                   → 0.2s
  start new read thread                              → instant
```

Total: 1-3 seconds of zero teleop forwarding. Leader moves freely, then when the loop resumes it reads the current (far-ahead) leader position and sends a huge position jump → burst/jerk.

### Why camera_preview.py doesn't lag

Only does `cap.read()` → `get_action()` → `send_action()`. No `async_read()` (no watchdog/reconnect), no dataset writing, no observation building.

## Fix

### read_latest() on both camera classes

Added to `OpenCVCamera` and `RealSenseCamera`. Returns `self.latest_frame` under `frame_lock` immediately — no `new_frame_event.wait()`, no watchdog, no reconnect. Returns `None` if no frame captured yet (background thread hasn't delivered one).

```python
def read_latest(self) -> np.ndarray | None:
    if not self.is_connected:
        raise DeviceNotConnectedError(...)
    if self.thread is None or not self.thread.is_alive():
        self._start_read_thread()
    with self.frame_lock:
        return self.latest_frame
```

The background `_read_loop()` still handles its own reconnection autonomously (consecutive failure counter at lines 470-488 in opencv camera). By never calling `async_read()` from the main loop, `_force_reconnect()` never blocks teleop.

### SO101Follower.get_observation(blocking_cameras=False)

When `blocking_cameras=False`, uses `read_latest()` instead of `async_read()`. Falls back to `async_read()` only when `read_latest()` returns `None` (first frame before background thread has captured anything).

### Record loop reorder for teleop mode

Before (every branch):
```
observation = robot.get_observation()     ← blocks 30-66ms+ on cameras
action = teleop.get_action()
sent_action = robot.send_action(action)
dataset.add_frame(...)
```

After (teleop branch only):
```
action = teleop.get_action()              ← 3-5ms serial read leader
sent_action = robot.send_action(action)   ← 3-5ms serial write follower
observation = robot.get_observation(      ← 3-5ms serial read follower
    blocking_cameras=False)                    + instant frame grab
dataset.add_frame(...)                    ← <1ms queue put
```

Policy mode is unchanged — policy needs observation before it can predict an action.

### Robot.get_observation(**kwargs) interface

Updated the abstract base class and all 11 concrete implementations to accept `**kwargs` so `blocking_cameras=False` propagates without TypeError on any robot type, not just SO101Follower.

## Timing instrumentation

Added per-operation timing to record_loop. Logs a warning on any iteration >50ms:

```
WARNING: Slow loop: 87ms (obs=52 action=55 send=60 frame=87)
```

The timestamps are cumulative from `start_loop_t` (except `frame` which is relative to `t_send`), so you can see exactly which phase ate the time.

## Edge cases

- **First frame**: `read_latest()` returns `None` → `get_observation()` falls back to blocking `async_read()` for first call only
- **Dead camera**: `read_latest()` returns last good frame forever. Stale frame is acceptable — background thread handles reconnection, and a stale frame is better than blocking teleop for seconds
- **Frame duplication**: If control loop runs faster than camera fps, same frame appears in consecutive dataset rows. Motor positions still differ each timestep so the data is still useful
- **Dataset=None (reset phase)**: Teleop branch still runs action-first for smooth movement; observation is captured but not written

## Files changed

| File | Change |
|---|---|
| `cameras/opencv/camera_opencv.py` | `read_latest()` |
| `cameras/realsense/camera_realsense.py` | `read_latest()` |
| `robots/so101_follower/so101_follower.py` | `blocking_cameras` param |
| `record.py` | Loop reorder + timing instrumentation |
| `robots/robot.py` + 11 implementations | `**kwargs` on `get_observation()` |

## Test results (v3 recording, 30 episodes)

The fix eliminated all software-induced blocking. Zero `Slow loop` warnings across 30 episodes — every loop iteration completed under 50ms. Worst individual read was 6.7ms (camera), typical was <2ms.

However, 0.5-2 second lag was still felt at episodes 18-19. Log analysis confirmed it's not from the record loop, camera reads, or dataset writing. Not from video encoding (happens between episodes) or the rerun viewer.

### Remaining lag source: motor hardware degradation

The follower arm has been in use for ~6 months. During the same session:
- Motor 6 (gripper) threw `Overload error!` on disconnect
- `sync_read` failed twice with `no status packet` — serial bus couldn't reach any motor
- Both crashes happened during `get_observation()` motor reads, not camera reads

The intermittent lag likely comes from the STS3215 servo motors themselves:
- **Worn plastic gears** — increased backlash means the motor physically can't track fast leader movements, creating perceived lag even though commands arrive on time
- **Degraded motor controller** — a flaky motor on the daisy chain can delay or corrupt serial packets for all 6 motors, causing brief communication stalls
- **Gripper motor (id=6)** is the most suspect — it's last in the daisy chain and takes the most mechanical abuse during grasping. A bad motor at the end can still disrupt the whole bus

**Diagnosis steps:**
1. Power off, wiggle each joint by hand — noticeable play = worn gears
2. Run teleop for 2 min, feel each motor for heat — hot to touch = dying
3. Reseat all daisy-chain connectors, check for frayed wires
4. If gripper motor is suspect, swap it first (it's cheapest to test since it's at the chain end)

**Action:** Order 2-3 spare STS3215 motors. Swap gripper first and retest.
