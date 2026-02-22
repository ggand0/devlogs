# Teleoperation Lag During Recording

## Problem

Teleoperation feels laggy during recording (`record.py`), causing jerky movements when the operator moves fast. The follower arm falls behind the leader, then catches up in a burst. This doesn't happen (or is much less noticeable) when using `camera_preview.py --teleop`.

## Root Cause

Lerobot's `record_loop` couples camera capture, dataset writing, and teleop in a single synchronous loop. Each iteration at 30fps does:

1. `robot.get_observation()` — sync_read all 6 motors + read all cameras (RealSense is slow)
2. `build_dataset_frame()` — convert to dataset format
3. `teleop.get_action()` — read leader motor positions
4. `robot.send_action()` — write goal positions to follower
5. `dataset.add_frame()` — write parquet + video encoding

The camera reads (especially RealSense over USB) and dataset I/O block the teleop action forwarding. During that dead time, leader arm movements queue up and the follower falls behind.

In contrast, `camera_preview.py --teleop` only does:
1. `cap.read()` (one camera)
2. `leader.get_action()`
3. `follower.send_action()`

Much tighter loop, much less latency.

## Impact

- Jerky movements during fast teleoperation corrupt demonstration quality
- Operator has to move slowly and deliberately to avoid lag-induced jerks
- Some recorded episodes have to be discarded due to jerky artifacts

## Possible Fix

Decouple camera capture and dataset writing into background threads so the teleop loop only does motor reads/writes. This would require refactoring lerobot's `record_loop`.

## Workaround

Move slowly during recording. The lag is small enough that deliberate movements aren't affected.
