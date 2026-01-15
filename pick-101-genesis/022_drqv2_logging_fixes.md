# Devlog 022: DrQ-v2 Logging Fixes

**Date:** 2026-01-12

## Problem

RoboBase's default logging output was not matching the SB3-style format used in PPO training, making it difficult to track training progress at a glance.

Issues:
1. No progress bar showing training advancement
2. Metrics only printed at `log_every` intervals (not per-step)
3. RoboBase's console output format was verbose and hard to parse
4. No real-time FPS tracking in the progress bar

## Solution

### 1. SilentLogger

Created `SilentLogger` class that extends RoboBase's `Logger` to suppress console output while preserving CSV/TensorBoard logging:

```python
class SilentLogger(RoboBaseLogger):
    """Logger that suppresses console output but keeps CSV/TensorBoard."""

    def _dump(self, step, prefix=None):
        """Override to skip console output, only CSV/TB."""
        if prefix is None or prefix == "eval":
            self._dump_csv_only(self._eval_mg)
        if prefix is None or prefix == "train":
            self._dump_csv_only(self._train_mg)
        # ... other prefixes

    def _dump_csv_only(self, mg: MetersGroup):
        """Dump MetersGroup to CSV only, skip console."""
        if len(mg._meters) == 0:
            return
        data = mg._prime_meters()
        if mg._save_csv:
            mg._dump_to_csv(data)
        mg._meters.clear()
```

### 2. tqdm Progress Bar

Added tqdm progress bar that updates every training step:

```python
pbar = tqdm(
    total=self.cfg.num_train_frames,
    initial=self.global_env_steps,
    desc="Training",
    unit="steps",
    dynamic_ncols=True,
)

# In training loop:
pbar.update(steps_per_iter)
pbar.set_postfix({
    'fps': f"{current_fps:.0f}",
    'rew': f"{agent_0_prev_reward:.1f}" if agent_0_prev_reward is not None else "N/A",
    'suc': f"{agent_0_prev_success:.0%}" if agent_0_prev_success is not None else "N/A",
    'buf': len(self.replay_buffer),
}, refresh=True)
```

Output:
```
Training:  10%|████      | 100000/1000000 [32:15<4:51:20, 51.5steps/s, fps=52, rew=45.2, suc=0%, buf=12500]
```

### 3. SB3-Style Statistics Table

Implemented `_print_sb3_stats_to_pbar()` for periodic detailed logging:

```
------------------------------------------------------------
| rollout/                                 |
------------------------------------------------------------
| ep_len_mean                              |      200.0 |
| ep_rew_mean                              |     45.234 |
| ep_success_rate                          |        0.0% |
------------------------------------------------------------
| time/                                    |
------------------------------------------------------------
| fps                                      |         52 |
| iterations                               |      12500 |
| time_elapsed                             |   00:32:15 |
| total_timesteps                          |     100000 |
------------------------------------------------------------
| train/                                   |
------------------------------------------------------------
| actor_loss                               |    0.12345 |
| critic_loss                              |    0.23456 |
| alpha                                    |    0.10000 |
------------------------------------------------------------
```

Uses `pbar.write()` to print above the progress bar without disrupting it.

### 4. Real-time FPS Tracking

Added continuous FPS calculation instead of only computing when `agent.logging=True`:

```python
# Track FPS
fps_start_time = time.time()
fps_start_steps = self.global_env_steps

# In loop:
elapsed = time.time() - fps_start_time
current_fps = (self.global_env_steps - fps_start_steps) / max(elapsed, 1e-6)
```

## Config Updates

Updated `genesis_lift.yaml` for 100k eval/checkpoint intervals:

```yaml
# Training settings
num_train_frames: 1000000
eval_every_steps: 100000
num_eval_episodes: 10

# Checkpointing
save_snapshot: true
snapshot_every_n: 12500  # Save every ~100k steps (12500 iters * 8 envs)

# Eval video recording
log_eval_video: true
log_eval_video_every_n_evals: 1  # Save video on every eval
```

## Key Implementation Details

### Progress Bar Timing

The iteration counter is incremented BEFORE updating the progress bar to ensure accurate step counts:

```python
# Increment iteration counter BEFORE progress update
self._main_loop_iterations += 1

# Calculate FPS
elapsed = time.time() - fps_start_time
current_fps = (self.global_env_steps - fps_start_steps) / max(elapsed, 1e-6)

# Update progress bar every step
pbar.update(steps_per_iter)
```

### Episode Metrics

Episode reward/success only shown after first episode completes:
- `rew=N/A` until first episode ends (~200 steps per env)
- `buf=0` until first episode adds to replay buffer

This is expected behavior - DrQ-v2 adds full episodes to the replay buffer, not individual transitions.

## Files Modified

- `src/training/genesis_workspace.py`: SilentLogger, tqdm, SB3-style output
- `src/training/cfgs/genesis_lift.yaml`: 100k eval/checkpoint intervals

## Commit

`49fae7b` - Add SB3-style logging and progress bar for DrQ-v2 training
