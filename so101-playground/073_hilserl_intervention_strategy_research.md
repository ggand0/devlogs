# HIL-SERL Intervention Strategy Research

**Date**: 2026-01-26
**Purpose**: Clarify the correct intervention strategy for HIL-SERL training

---

## The Problem

The HIL-SERL documentation contains seemingly contradictory advice:

1. "Keep interventions **brief and frequent**"
2. "Help policy get reward in **~1/3+ of episodes**"
3. "Let policy explore **20-30 timesteps**, then guide"
4. "**AVOID** persistently providing long sparse interventions that lead to task successes"

These statements appear to conflict. How can interventions be "brief" while also guiding to success?

---

## Resolution: The Actual Strategy

From the official Franka walkthrough (`docs/franka_walkthrough.md` line 164):

> "For instance, for an insertion task (i.e. RAM insertion), in the beginning of training, the policy will have a lot of random motion. We would typically **let it explore these random motions for 20-30 timesteps** then **intervene to guide the object close to the insertion port**, where we would **let the policy practice the insertion**."

### The Three-Phase Episode Structure

| Phase | Steps | Duration @ 10Hz | What Happens |
|-------|-------|-----------------|--------------|
| 1. Policy Exploration | 0-25 | 0-2.5 sec | Let policy try on its own |
| 2. Human Guidance | 25-50 | 2.5-5.0 sec | Guide gripper to target region |
| 3. Policy Practice | 50-100 | 5.0-10.0 sec | Release control, let policy attempt to finish |

**Key insight**: The intervention in Phase 2 can be substantial (20-40 steps). The word "brief" refers to not taking over the **entire** episode from start to finish.

### Critical Nuance: You Do NOT Complete the Task Yourself

From line 164:
> "We also found it beneficial to intervene to help the policy finish the task and get the reward semi-frequently even in the beginning (i.e. **letting** 1/3 or more of the episodes get reward)"

The word **"letting"** is key. You:
1. Guide the robot toward the target region
2. **Release control** and let the policy attempt to finish
3. The policy succeeds ~1/3+ of the time on its own

You do NOT intervene all the way to task completion. The 1/3+ success rate is an expected outcome when you position the robot well, not a target you enforce by completing tasks yourself.

---

## HIL-SERL Episode Durations (from code)

| Task | MAX_EPISODE_LENGTH | Hz | Duration |
|------|-------------------|-----|----------|
| RAM insertion | 100 steps | 10 Hz | **10 seconds** |
| USB pickup/insertion | 120 steps | 10 Hz | **12 seconds** |
| Object handover | 200 steps | 10 Hz | **20 seconds** |
| Default (franka_env.py) | 100 steps | 10 Hz | **10 seconds** |

Source: `/hil-serl/examples/experiments/*/config.py` and `/hil-serl/serl_robot_infra/franka_env/envs/franka_env.py`

---

## What "Brief" Actually Means

**WRONG interpretation**: 1-2 step micro-corrections

**CORRECT interpretation**:
- Don't take over from step 0 to step 100 (entire episode)
- Intervene in the **middle** of the episode
- Let policy explore first, guide to region, then **release** for policy to practice

### The Anti-Pattern Explained

> "AVOID: Persistently providing **long sparse** interventions that lead to task successes"

This means:
- **WRONG**: Episode 1 → intervene 100% → success → Episode 2-5 → no intervention → fail → Episode 6 → intervene 100% → success
- **RIGHT**: Episode 1 → explore 25 steps → intervene 25 steps → release 50 steps → Episode 2 → same pattern → Episode 3 → same pattern

The problem is **sparse** (intervening rarely) combined with **long** (taking over entire episode). The fix is **frequent** (every episode) with **middle-of-episode** guidance.

---

## Updated Config

Changed episode duration from 8s to 10s to match HIL-SERL:

```json
"control_time_s": 10.0  // was 8.0
```

This gives 100 steps @ 10 fps:
- Steps 0-25: Policy exploration
- Steps 25-50: Human guidance to target
- Steps 50-100: Policy practice (grasp attempt)

---

## Correct Intervention Strategy for SO101 Grasp

### Early Training (Episodes 1-100)

1. **Every episode**: Intervene
2. **Structure**:
   - Let robot move randomly for ~2.5 seconds (25 steps)
   - Take spacemouse control, guide gripper toward cube (~2.5 seconds)
   - **Release spacemouse**, let policy try to grasp (~5 seconds)
3. **Target**: 1/3+ of episodes should get reward

### Mid Training (Episodes 100-300)

1. Intervene less frequently (every 2-3 episodes)
2. Only intervene if policy is clearly going wrong direction
3. Watch for success rate improvement

### Late Training (Episodes 300+)

1. Minimal intervention
2. Only intervene on repeated failures
3. Intervention rate should approach 0%

---

## Why 700+ Episodes Failed

The previous training sessions likely had inconsistent intervention patterns:

- Some episodes: 96.7% intervention (taking over almost entire episode)
- Other episodes: 0% intervention (letting policy fail completely)

This is the "long sparse" anti-pattern that causes value function overestimation.

**The fix**: Consistent middle-of-episode guidance with policy practice at the end.

---

## Key Metrics to Track

From `train_rlpd.py`:
```python
info["episode"]["intervention_count"] = intervention_count  # Number of intervention starts
info["episode"]["intervention_steps"] = intervention_steps  # Total timesteps intervened
```

Track:
- `intervention_steps / episode_length` → should decrease over time
- `success_rate` → should increase over time
- Both should correlate (as intervention rate drops, success rate rises)

---

## Paper Verification (arxiv 2410.21845)

### Verified from Paper

**On the anti-pattern:**
> "we should avoid persistently providing long sparse interventions that lead to task successes. Such an intervention strategy will cause the overestimation of the value function, particularly in the early stages of the training process; which can result in unstable training dynamics."

**On intervention philosophy:**
> "the policy improves faster when the human operator issues specific corrections while letting the robot explore on its own otherwise"

**On frequency:**
> "In the beginning of the training process, the human intervenes more frequently to provide corrective actions, gradually decreasing the frequency as the policy improves."

**On replay buffer handling:**
> "We store the intervention data in both the demonstration and RL data buffers. However, we add the policy's transitions (i.e., the states and actions before and after the intervention) only to the RL buffer."

### NOT in Paper (from docs only)

The following guidance is from `franka_walkthrough.md`, **not** the paper:
- "1/3 or more of episodes get reward"
- "Let it explore 20-30 timesteps then intervene"
- "Let the policy practice the insertion"

The paper does NOT prescribe a specific success rate quota. The "1/3+ episodes" guidance is operational advice from the Franka walkthrough documentation, not a research finding.

---

## Sources

- HIL-SERL Paper: arxiv 2410.21845
- Franka Walkthrough: `/hil-serl/docs/franka_walkthrough.md`
- Episode Config: `/hil-serl/examples/experiments/ram_insertion/config.py`
- Environment: `/hil-serl/serl_robot_infra/franka_env/envs/franka_env.py`
- Training Loop: `/hil-serl/examples/train_rlpd.py`
