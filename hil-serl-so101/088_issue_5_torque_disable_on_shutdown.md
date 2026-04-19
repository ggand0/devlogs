# Devlog 088: GitHub Issue #5 — Torque disable failure on shutdown

**Date:** 2026-04-19
**Issue:** https://github.com/ggand0/hil-serl-so101/issues/5
**Reporter:** @Manu752

## Summary

External user opened an issue titled "Recording data for offline buffer fails after 10 Episodes" — claimed recording fails after 9 episodes with a torque error. Actual cause: all 10 episodes recorded successfully; the error fires during `env.close()` shutdown when disabling torque on motor id=4 (wrist_flex). Data is on disk; the failure is in the shutdown cleanup path.

## What the user reported

Running `grasp_only_record_angled_10ep_config.json`. After seeing `Recording episode 9` in the log, got:

```
RuntimeError: Failed to write 'Torque_Enable' on id_=4 with '0' after 6 tries.
```

User's paste ends at `"after 6 tries."` with no decoded error string (Overload / Overheat / etc. unknown).

## What actually happened

Per OP's log, episodes are written per-episode before the next starts. Each episode block in the log shows:

```
Recording episode N → Episode ended → Map: 100%|...| → Creating parquet... 1/1 → SVT-AV1 encoder block
```

Pattern repeats 10 times for N=0..9. Episode 9's block (lines 539–564) completes fully, then the traceback fires on line 565. So the parquet and video were flushed to disk for all 10 episodes before shutdown.

## Traceback path

```
main() (gym_manipulator.py:3310)
  → env.close()
  → RobotEnv.close() (gym_manipulator.py:646)
  → robot.disconnect() (so101_follower.py:224)
  → bus.disconnect(disable_torque_on_disconnect) (motors_bus.py:472)
  → disable_torque(num_retry=5) (feetech.py:298)
  → write("Torque_Enable", id=4, value=0) (motors_bus.py:1023)
  → _write (motors_bus.py:1049)
  → raise RuntimeError
```

The shutdown path calls `bus.disable_torque()` directly. It does not go through the `_robust_sync_write` / `_robust_sync_read` helpers (at `gym_manipulator.py:150` and `:176`) that we added for in-episode ops like IK reset, teleop leader sync, and goal-position writes. Those helpers do port-clear, exponential backoff, and USB reconnect; `disconnect()` does packet-level retry only.

## Why the motor rejected the write

Feetech servos return a status packet with an error byte on every write. If a fault flag is set, the motor NACKs the write with that error code; packet-level retry just resends the same packet and gets the same NACK. Fault flags are latched — they survive retries and need a power-cycle or reboot instruction to clear.

Which fault flag was set is unknown from OP's log (paste truncated). Devlog 040 had an identical trace (`id_=4`, "after 6 tries", shown to be Overload), but that's a separate incident and not evidence for this one.

## Related history

- **Devlog 040** (`safe_return_fixes.md`, commit `13309b3`): same error string on `rl_inference.py` shutdown after Ctrl+C. Fixed there by doing an IK-lift + interpolated REST transition + 0.5s serial recovery delay + wrapping `safe_return()` in try/except. Prevented the strain, didn't add recovery for a latched motor.
- **Devlog 087** (`placo_ee_consolidation.md`): documents the `_robust_sync_read/write` helpers and their placement — in-episode ops, not shutdown.

## Gap

`_robust_sync_write` covers runtime ops but not `env.close()` → `disconnect()`. If the motor ends a session in a latched fault state, the shutdown write will fail regardless of how many times we retry at packet level. A defensive fix would be to wrap `env.close()` in try/except so the shutdown failure doesn't mask a successful recording run — but this is upstream LeRobot code, not our repo.

## Reply posted

> Judging from the log, all 10 episodes were recorded and written to parquet before the error. I've run into the same error before, and there's retry logic during recording to reconnect to the motors. The error suggests the wrist_flex motor was in an error state and it couldn't be disabled. Power-cycling the arm usually clears it. Hope that helps!

## Open questions (not pursued)

- Which specific fault did wrist_flex latch? Would need OP to paste the full error line.
- Why wrist_flex specifically? Top-down IK reset holds it at 90° for the whole episode, which is a sustained-load pose; could be thermal, not investigated.
