# Devlog 042: DrQ-v2 Encoder Synchronization Fix

**Date**: 2025-01-18
**Status**: Complete

## Problem

When running HIL-SERL with DrQ-v2 pretrained weights from Genesis, the robot arm behavior was identical to training from scratch - the pretrained policy weights had no effect.

Running `rl_inference.py` directly with the same checkpoint showed correct learned behavior, confirming the checkpoint was valid.

## Root Cause Analysis

The HIL-SERL architecture creates separate policy instances in learner and actor processes:

1. **Learner**: Creates policy → loads pretrained weights → pushes parameters to actor
2. **Actor**: Creates fresh policy (random weights) → receives parameters from learner

The issue: **encoder weights were not being synchronized**.

### Code Flow (Before Fix)

**Learner** (`learner.py:1341`):
```python
state_dicts = {"policy": policy.actor.state_dict()}  # Only actor weights!
```

**Actor** (`actor.py:850`):
```python
policy.actor.load_state_dict(actor_state_dict)  # Only loads actor
```

### Why This Matters for DrQ-v2 vs SAC

| Aspect | SAC | DrQ-v2 |
|--------|-----|--------|
| Encoder | Pretrained ResNet (frozen) | Custom CNN (trained end-to-end) |
| Encoder sync needed? | No - both start with same pretrained weights | **Yes - encoder is critical** |

For SAC, the encoder is frozen and both actor/learner initialize with the same pretrained ResNet weights, so no sync is needed.

For DrQ-v2, the encoder is trained end-to-end. The learner loads pretrained encoder weights from the Genesis checkpoint, but the actor's encoder remained **random** because encoder weights were never pushed.

## Fix Applied

### 1. Learner: Push encoder weights

**File**: `lerobot/src/lerobot/scripts/rl/learner.py`

```python
def push_actor_policy_to_queue(parameters_queue: Queue, policy: nn.Module):
    state_dicts = {"policy": policy.actor.state_dict()}

    # NEW: Add encoder if it exists (needed for DrQ-v2)
    if hasattr(policy, "encoder") and policy.encoder is not None:
        state_dicts["encoder"] = policy.encoder.state_dict()

    # ... rest of function
```

### 2. Actor: Wait for initial parameters

**File**: `lerobot/src/lerobot/scripts/rl/actor.py`

Added blocking wait after policy creation:

```python
# Wait for initial parameters from learner (critical for pretrained policies)
logging.info("[ACTOR] Waiting for initial parameters from Learner...")
try:
    bytes_state_dict = parameters_queue.get(block=True, timeout=timeout_seconds * 5)
    state_dicts = bytes_to_state_dict(bytes_state_dict)

    # Load actor state dict
    policy.actor.load_state_dict(state_dicts["policy"])

    # Load encoder if present (needed for DrQ-v2)
    if hasattr(policy, "encoder") and "encoder" in state_dicts:
        policy.encoder.load_state_dict(state_dicts["encoder"])
        logging.info("[ACTOR] Loaded encoder parameters from Learner.")

    logging.info("[ACTOR] Loaded initial parameters from Learner.")
except Empty:
    logging.warning("[ACTOR] Timeout waiting for initial parameters, using random weights")
```

### 3. Actor: Load encoder during policy updates

**File**: `lerobot/src/lerobot/scripts/rl/actor.py`

Modified `update_policy_parameters()`:

```python
def update_policy_parameters(policy, parameters_queue, device):
    # ... existing actor loading ...

    # NEW: Load encoder if present (needed for DrQ-v2)
    if hasattr(policy, "encoder") and "encoder" in state_dicts:
        encoder_state_dict = move_state_dict_to_device(
            state_dicts["encoder"], device=device
        )
        policy.encoder.load_state_dict(encoder_state_dict)
        logging.info("[ACTOR] Loaded encoder parameters from Learner.")
```

## Verification

After the fix, actor logs should show:
```
[ACTOR] Waiting for initial parameters from Learner...
[ACTOR] Loaded encoder parameters from Learner.
[ACTOR] Loaded initial parameters from Learner.
```

## Files Modified

- `lerobot/src/lerobot/scripts/rl/learner.py` - Push encoder in `push_actor_policy_to_queue()`
- `lerobot/src/lerobot/scripts/rl/actor.py` - Wait for initial params, load encoder in `update_policy_parameters()`

## Commits

- `13d1dc13` - Add encoder parameter synchronization for DrQ-v2 policies

## Key Insight

The HIL-SERL codebase was designed for SAC with frozen pretrained encoders. DrQ-v2's end-to-end encoder training requires explicit encoder synchronization between learner and actor processes.

## Related

- Devlog 041: DrQ-v2 HIL-SERL Integration
- Devlog 037: Genesis DrQ-v2 inference (sim2real transfer)
