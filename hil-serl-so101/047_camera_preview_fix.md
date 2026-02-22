# Devlog 047: Camera Dies When Reward Preview Enabled

**Date**: 2025-01-25
**Status**: Complete

## Problem

Camera consistently died ~5 seconds into HIL-SERL training when `display_reward_preview: true` was enabled. The camera worked fine when preview was disabled.

Error pattern:
```
WARNING: Camera gripper_cam timeout attempt 1/5
WARNING: OpenCVCamera(/dev/video0): Thread stuck for 0.8s, forcing reconnect...
ERROR: OpenCVCamera(/dev/video0): No working camera found during force reconnect
```

## Root Cause

**Python's `fork()` corrupts OpenCV internal locks when called while threads are running.**

The reward preview used `multiprocessing.Process` which calls `fork()`. When fork happens while the camera background thread holds an internal OpenCV lock, the lock is copied in an acquired state to the child process. This corrupts the parent process state, causing the camera thread to deadlock.

Evidence from Python bug tracker:
- [Issue 6721](https://bugs.python.org/issue6721): "If fork() happens at the wrong time, the lock is copied in an acquired state. In the child process it will never be released."
- [OpenCV Issue #5150](https://github.com/opencv/opencv/issues/5150): OpenCV crashes when used with Python multiprocessing due to fork issues

The gRPC warning in logs confirmed this:
```
Other threads are currently calling into gRPC, skipping fork() handlers
```

## Solution

### 1. Use `spawn` instead of `fork`

The `spawn` start method creates a fresh Python interpreter without copying parent memory, avoiding lock corruption:

```python
import multiprocessing as mp
ctx = mp.get_context('spawn')  # spawn avoids fork lock corruption
preview_queue = ctx.Queue(maxsize=1)
preview_proc = ctx.Process(target=_preview_display_loop, args=(preview_queue,), daemon=True)
preview_proc.start()
```

### 2. Start preview process BEFORE camera connects

Even with `spawn`, starting the process before any threads exist is safer:

```python
# In make_robot_env(), BEFORE robot = make_robot_from_config()
if display_reward:
    ctx = mp.get_context('spawn')
    preview_queue = ctx.Queue(maxsize=1)
    preview_proc = ctx.Process(target=_preview_display_loop, ...)
    preview_proc.start()

# THEN create robot (which starts camera thread)
robot = make_robot_from_config(cfg.robot)
```

### 3. Preview runs in isolated process

The preview display loop runs in a completely separate process, with its own OpenCV instance:

```python
def _preview_display_loop(queue):
    """Separate process for displaying preview."""
    import cv2
    cv2.namedWindow("Reward", cv2.WINDOW_NORMAL)
    while True:
        try:
            img_np, prob, threshold = queue.get(timeout=1.0)
            # ... display with cv2.imshow/waitKey
        except:
            cv2.waitKey(100)
```

Main process sends frames via queue (non-blocking), preview process displays them.

## Additional Camera Robustness Improvements

### Watchdog for stuck camera thread

Added detection for camera thread stuck on `read()`:

```python
# In async_read()
if thread_alive and self.last_successful_read > 0:
    time_since_last_read = time.perf_counter() - self.last_successful_read
    if time_since_last_read > 0.5:
        logger.warning(f"{self}: Thread stuck, forcing reconnect...")
        self._force_reconnect()
```

### Force reconnect mechanism

When camera thread is stuck, force reconnect by releasing VideoCapture (unblocks stuck read):

```python
def _force_reconnect(self):
    self.stop_event.set()
    if self.videocapture is not None:
        self.videocapture.release()  # Unblocks stuck read()
    # ... reconnect to working camera
```

### MJPG format for USB stability

Added MJPG format and buffer size settings to reduce USB bandwidth:

```python
self.videocapture.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
self.videocapture.set(cv2.CAP_PROP_BUFFERSIZE, 1)
```

## Files Modified

1. `lerobot/src/lerobot/scripts/rl/gym_manipulator.py`
   - Added `_preview_display_loop()` function for separate process
   - Modified `make_robot_env()` to start preview process before camera
   - Modified `RewardWrapper` to accept pre-created preview queue
   - Modified `_overlay_reward_on_preview()` to send to queue instead of cv2.imshow

2. `lerobot/src/lerobot/cameras/opencv/camera_opencv.py`
   - Added `last_successful_read` tracking for watchdog
   - Added `_force_reconnect()` method
   - Added watchdog check in `async_read()`
   - Added MJPG format and buffer size settings in `connect()`

## Key Takeaway

Never use `fork()` (default multiprocessing on Linux) when threads are running. Use `spawn` context:

```python
ctx = mp.get_context('spawn')
proc = ctx.Process(target=fn, args=(...))
```

## Commit

```
d07f3cc4 Fix camera dying when reward preview is enabled
```
