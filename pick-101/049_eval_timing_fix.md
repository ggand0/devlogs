# Eval Timing Fix

**Date:** 2025-01-03

## Problem

Eval videos were not being saved during training. Multiple "fixes" were attempted that didn't work:

1. Changed from `log_eval_video_every` (env steps) to `log_eval_video_every_n_evals` (count)
2. Fixed `torch.load` to use `weights_only=False` for resume
3. Fixed `load_snapshot` to not overwrite `cfg` from snapshot

**None of these were the actual problem.**

## Root Cause

The config parameter `eval_every_steps` is named misleadingly. It's actually **iterations**, not env steps.

```
global_env_steps = main_loop_iterations * action_repeat * num_envs
                 = iterations * 2 * 8
                 = iterations * 16
```

With the original config:
```yaml
eval_every_steps: 10000  # This is ITERATIONS, not env steps!
```

Actual eval frequency: `10000 * 16 = 160,000 env steps`

With `log_eval_video_every_n_evals: 5`, videos only saved every 5 evals = **800,000 env steps**.

That's why only `0.mp4` existed - the next video would be at 800k.

## Fix

Changed config to eval every ~100k env steps:

```yaml
eval_every_steps: 6250  # 6250 * 16 = 100k env steps
log_eval_video_every_n_evals: 1  # Video at every eval
```

## Math Verification

```
Steps per iteration = action_repeat * num_envs = 2 * 8 = 16

eval_every_steps = 6250 iterations
Env steps per eval = 6250 * 16 = 100,000

Evals at iterations: 0, 6250, 12500, 18750, 25000, 31250, ...
Evals at env steps:  0, 100k, 200k, 300k, 400k, 500k, ...
```

## Resume Behavior

After resume from 500k (iteration 31250):
- `31250 % 6250 = 0` → Eval triggers immediately
- `eval_count` resets to 0 (local variable in `_online_rl`)
- First eval after resume saves video

## Tests Added

Created `tests/test_eval_timing.py` with tests for:

1. `Every` class behavior (triggers at 0, N, 2N, ...)
2. `Until` class behavior (True while step < limit)
3. Eval timing with realistic config values
4. Resume from 500k triggers eval immediately
5. Video saving with `video_every_n_evals=1` and `=5`
6. Full training simulation 500k→1500k

All tests pass:
```
=== Full Training Simulation 500k→1500k ===
PASS: 10 evals from 500k to 1400k
      Evals at: [500000, 600000, 700000, 800000, 900000, 1000000, 1100000, 1200000, 1300000, 1400000]
      Videos at: [500000, 600000, 700000, 800000, 900000, 1000000, 1100000, 1200000, 1300000, 1400000]
```

## Other Fixes Made

1. `workspace.py:819` - Added `weights_only=False` to `torch.load` for PyTorch 2.6 compatibility
2. `workspace.py:825-827` - Skip loading `cfg` from snapshot to allow resuming with different `num_train_frames`

## Files Changed

- `configs/drqv2_lift_s2_sanity.yaml` - Fixed eval timing
- `src/training/workspace.py` - Fixed resume logic
- `tests/test_eval_timing.py` - New tests

## Lesson Learned

**Read the fucking code and do the math before claiming something is fixed.**

The parameter name `eval_every_steps` is misleading - it should be `eval_every_iterations` or the code should convert env steps to iterations internally.
