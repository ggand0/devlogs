# CNN (Pixel-Based) SAC Experiment Results

## Experiment Overview

**Objective**: Test whether a CNN policy trained on top-down rendered observations could learn the dodge task, compared to MLP policies with explicit state vectors.

**Date**: December 28-29, 2024

**Duration**: ~22 hours 42 minutes

---

## Configuration

```yaml
algorithm: SAC
observation_mode: topdown          # 256x256 grayscale top-down camera
image_obs_width: 256
image_obs_height: 256
image_grayscale: true
frame_stack: 4                     # Stack 4 frames for temporal info

total_timesteps: 500000
learning_rate: 0.0003
buffer_size: 50000                 # ~24GB with 256x256x4 observations
batch_size: 128
learning_starts: 10000
train_freq: 4
gradient_steps: 1

level: 2                           # Hard difficulty
action_space_type: basic_3d        # Continuous [vx, vy, sprint]

transport: grpc                    # Unix socket for environment communication
```

**Hardware**: GPU training with gRPC transport achieving 225 FPS during data collection, dropping to ~6 FPS during training (GPU-bound).

---

## Results

### Final Metrics

| Metric | Value |
|--------|-------|
| Total timesteps | 500,000 |
| Training time | 22h 42m 37s |
| Final training reward | -75.69 |
| Peak training reward | 129.84 |
| Final eval reward | 89.71 ± 20.73 |
| Best eval reward | 164.35 |
| Final episode length | 25 steps |
| Peak episode length | 227 steps |
| Final eval episode length | 185.80 ± 17.84 |

### Training Dynamics

```
Eval num_timesteps=500000, episode_reward=89.71 +/- 20.73
Episode length: 185.80 +/- 17.84
---------------------------------
| eval/              |          |
|    mean_ep_length  | 186      |
|    mean_reward     | 89.7     |
| train/             |          |
|    actor_loss      | 59       |
|    critic_loss     | 31.6     |
|    ent_coef        | 0.0986   |
|    n_updates       | 489999   |
---------------------------------
```

### Performance Over Time

- **0-10K steps**: Data collection only (225 FPS), no training
- **10K-50K steps**: FPS dropped to ~6 as gradient updates began
- **Peak performance**: Reached ~164 eval reward mid-training
- **Final performance**: Degraded to ~90 eval reward (instability)

---

## Analysis

### What Worked

1. **gRPC transport**: Achieved 225 FPS during data collection phase, validating the transport optimization
2. **Basic learning**: Agent did learn to dodge to some extent (eval reward 89.7 > random policy)
3. **Frame stacking**: 4-frame stack provided temporal information for velocity inference

### What Didn't Work

1. **Sample efficiency**: 500K steps over 22+ hours to achieve mediocre performance
2. **Training stability**: Performance peaked mid-training then degraded (164 → 90)
3. **Training throughput**: Only 6 FPS during training (GPU-bound on CNN backprop)

### Root Cause: Redundant Learning

The CNN must learn to:
1. Identify the player position from pixels
2. Identify projectile positions from pixels
3. Infer velocities from frame differences
4. Map these inferred states to actions

With explicit state vectors, all of this information is directly available in a 65-69 dimensional vector. The CNN spends most of its capacity on perception rather than policy learning.

---

## Comparison: CNN vs MLP (Expected)

| Aspect | CNN (Observed) | MLP (Expected) |
|--------|----------------|----------------|
| Observation | 256×256×4 pixels | 65-69 floats |
| Data per step | ~262 KB | ~260 bytes |
| Training FPS | 6 | 100-200 |
| Time to 500K steps | 22+ hours | ~1-2 hours |
| Sample efficiency | Low | High |
| Stability | Unstable (peaked then dropped) | Generally stable |
| Final performance | 89.7 eval reward | TBD |

---

## Key Insights

### 1. State-based outperforms pixel-based when state is available

This is well-established in RL literature:

- **Yarats et al. (2021)**: State-based SAC serves as upper bound that pixel methods struggle to match
- **Laskin et al. (2020)**: Pixel-based can approach state-based efficiency only with contrastive learning (CURL)
- **Ceron & Castro (2021)**: "Sample efficiency is a lot better for state-based continuous control than for the pixel-based version"

### 2. CNN makes sense only when state is unavailable

Use CNN/pixel-based when:
- Real robot with camera (no ground-truth state)
- Egocentric first-person view
- Visual features are the task (object recognition)

Use MLP/state-based when:
- Simulation with full state access
- Positions and velocities available
- Speed and sample efficiency matter

### 3. gRPC optimization benefits MLP more than CNN

| Setup | Data Collection | During Training |
|-------|-----------------|-----------------|
| CNN + gRPC | 225 FPS | 6 FPS (GPU-bound) |
| MLP + HTTP | ~50 FPS | ~40-50 FPS |
| MLP + gRPC | 225 FPS | ~100-200 FPS (expected) |

For CNN, the bottleneck shifts to GPU compute during training, making transport optimization less impactful. For MLP, the transport remains significant even during training.

---

## Foundational Literature

The performance gap between state-based and pixel-based RL is well-documented in the literature. Key papers:

### DeepMind Control Suite (Tassa et al., 2018)

The foundational benchmark that established the standard comparison between state-based and pixel-based continuous control.

- **Paper**: "DeepMind Control Suite"
- **Link**: https://arxiv.org/abs/1801.00690
- **Contribution**: Created benchmark tasks with both `state` and `pixels` observation modes, making this comparison a standard evaluation in the field

### DrQ-v2 (Yarats et al., 2021)

Demonstrated that pixel-based methods can *approach* state-based performance, but only with heavy data augmentation—implicitly confirming the gap.

- **Paper**: "Mastering Visual Continuous Control: Improved Data-Augmented Reinforcement Learning"
- **Link**: https://arxiv.org/abs/2107.09645
- **Key finding**: Data augmentation is critical for pixel-based RL; without it, the gap to state-based is substantial

### CURL (Laskin et al., 2020)

Highly cited (~2000+ citations) paper that explicitly framed its contribution as closing the efficiency gap.

- **Paper**: "CURL: Contrastive Unsupervised Representations for Reinforcement Learning"
- **Link**: https://arxiv.org/abs/2004.04136
- **Key quote**: Pixel-based can be "nearly as data-efficient as state-based RL"—the framing confirms state-based as the efficiency benchmark

### Why This Matters

The existence of entire research programs (DrQ, CURL, RAD, etc.) dedicated to closing the state-pixel gap confirms that:

1. The gap is real and significant
2. State-based is the default upper bound
3. Pixel-based requires special techniques to approach state-based performance

For tasks where ground-truth state is available (like this dodge game), using pixel observations is solving a harder problem than necessary.

---

## Visual Evaluation Observations

Running the trained CNN model visually reveals characteristic pixel-based agent behavior:

### Observed Behavior

1. **Jittery/twitching movement**: Agent makes rapid, erratic adjustments rather than smooth dodges
2. **Corner-seeking**: Agent gravitates toward left edge, then bottom-left corner
3. **Reactive rather than predictive**: Dodges a few balls but gets caught by subsequent ones
4. **Sprint oscillation**: Rapid back-and-forth sprinting, suggesting last-moment reactions

The overall effect is comedic—the agent looks panicked and confused, twitching frantically at the edge of the arena before inevitably getting hit. It's entertaining to watch despite (or because of) its ineffectiveness.

### Likely Causes

**1. Latency in visual processing**

By the time the CNN processes a frame and the 4-frame stack updates, fast-moving projectiles have already traveled significant distance. The agent is always "behind" the ball.

**2. Velocity inference limitations**

Frame stacking provides temporal information, but inferring velocity from pixel differences is:
- Noisy (small pixel changes between frames)
- Delayed (need multiple frames to estimate motion)
- Imprecise (no sub-pixel accuracy)

In contrast, MLP agents receive explicit velocity vectors with perfect accuracy.

**3. Corner strategy as local optimum**

Corners reduce the angle of incoming threats. The agent learned this defensive positioning but lacks the predictive capability to actively dodge in open space.

### Comparison to MLP Behavior

| Aspect | CNN Agent | MLP Agent (expected) |
|--------|-----------|---------------------|
| Movement | Jittery, reactive | Smooth, predictive |
| Positioning | Corner-seeking | Center-capable |
| Dodge timing | Last-moment | Anticipatory |
| Success mode | Avoid by position | Avoid by prediction |

### Visual Evaluation Commands

```bash
# Terminal 1: Game with rendering
VK_LOADER_DEBUG=error VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json cargo run --release -- --fps 60

# Terminal 2: Evaluation
uv run --directory python python eval_sac.py ../results/sac_topdown_cnn/20251228_014639/models/final_model.zip --observation-mode topdown --image-grayscale --frame-stack 4 --episodes 5
```

---

## Recommendations

1. **Use MLP for this task**: With full state access, there's no benefit to pixel-based learning
2. **Reserve CNN for future egocentric experiments**: If implementing first-person camera view without state access
3. **Consider smaller images if CNN needed**: 64×64 instead of 256×256 would be 16× faster
4. **Focus on reward shaping and environment design**: As noted in devlog 030, the "shuffling" behavior is a reward/environment issue, not an architecture issue

---

## Files

- Config: `python/configs/sac_topdown_cnn.yaml`
- Results: `python/results/sac_topdown_cnn/20251228_014639/`
- Training script: `python/train_sac.py`
