# Devlog 025: Action Smoothness Research

## Problem

RL-trained policies exhibit oscillation/twitching behavior during execution. In our curriculum stage 1 experiments (devlog 024), the agent shows minor twitching even when trying to hold a stable position.

## Research Findings

### The Oscillation Problem in RL

High-frequency oscillations are a known problem in deep RL, especially problematic when deploying to real hardware where they can damage actuators. The issue stems from the policy learning a non-smooth mapping from states to actions.

**Source**: [Benchmarking Smoothness and Reducing High-Frequency Oscillations in Continuous Control Policies](https://arxiv.org/html/2410.16632v1) (October 2024)

### Two Classes of Solutions

#### 1. Loss Regularization Methods

Add regularization terms to the policy gradient loss to encourage smooth action outputs.

**CAPS (Conditioning for Action Policy Smoothness)** - ICRA 2021

Two components:
- **Temporal smoothness** (L_T): Minimize distance between actions at consecutive states
  ```
  L_T = ||π(s_t) - π(s_{t+1})||²
  ```
- **Spatial smoothness** (L_S): Minimize distance between actions at nearby states
  ```
  L_S = ||π(s) - π(s + noise)||²
  where noise ~ N(0, σ)
  ```

Combined loss:
```
L_CAPS = λ_T · L_T + λ_S · L_S
```

Hyperparameters (environment-dependent):
- σ: 0.1 - 0.2
- λ_T: 0.01 - 0.1
- λ_S: 0.05 - 0.5

**Source**: [Regularizing Action Policies for Smooth Control with Reinforcement Learning](https://cs-people.bu.edu/rmancuso/files/papers/ICRA21_1616_FI.pdf)

**L2C2 Method**

Applies spatial regularization to both actor and value networks, sampling nearby states based on consecutive state differences rather than fixed hyperparameters.

#### 2. Architectural Methods

Modify network architecture to constrain smoothness.

**Spectral Normalization (Local SN)**
```
W_SN = δ · W / σ(W)
```
Applied only to output layer for improved performance.

**LipsNet**

A plug-and-play module:
```
y = K(x) · f(x) / (||∇f(x)|| + ε)
```
Where K(x) models input-dependent Lipschitz value.

**Hybrid Approach**

Combining LipsNet with CAPS or L2C2 achieves 26.8% smoothness improvement while maintaining task performance.

### Reward-Based Approaches

Simpler to implement, added directly to reward function.

#### Action Rate Penalty

Penalize change between consecutive actions:
```python
action_penalty = -λ * ||a_t - a_{t-1}||²
```
Typical λ = 0.01 to 0.1

#### Jerk Penalty

Penalize acceleration changes (second derivative):
```python
jerk_penalty = -λ * ||a_t - 2*a_{t-1} + a_{t-2}||²
```

#### Energy Penalty

Penalize action magnitude to reduce energy consumption and encourage smaller movements:
```python
energy_penalty = -λ * ||a_t||²
```

**Source**: [Learning to Recover: Dynamic Reward Shaping](https://arxiv.org/html/2506.05516)

### SAC-Specific Considerations

SAC's entropy regularization naturally helps smooth exploration:
- Higher entropy early keeps exploration systematic
- Prevents getting stuck in local optima
- Can tune `ent_coef` to prevent early collapse

**Source**: [RL in Continuous Action Spaces](https://medium.com/@amit25173/reinforcement-learning-in-continuous-action-spaces-4fc60897fa55)

### Ornstein-Uhlenbeck Noise

For exploration, OU noise provides time-correlated noise (smoother than Gaussian):
- Movements flow more naturally
- Useful for robotic arm tasks

### Trade-offs

> "The learning algorithm's tendency to exploit the reward function can lead to policies where subpar performance is preferred in favor of smoothness."

Action penalties can cause the agent to prefer doing nothing over completing the task. Requires careful tuning of penalty coefficient.

## Practical Implementation Options

### Option 1: Action Rate Penalty (Simplest)

Add to reward function:
```python
# In step():
action_delta = action - self._prev_action
action_penalty = -0.05 * np.sum(action_delta**2)
reward += action_penalty
self._prev_action = action.copy()
```

Pros:
- Simple to implement
- No changes to training algorithm
- Directly penalizes twitching

Cons:
- May slow down task completion
- Requires tuning λ

### Option 2: CAPS Loss Regularization

Requires modifying SAC training:
```python
# In policy update:
temporal_loss = ((policy(s_t) - policy(s_t1))**2).mean()
spatial_loss = ((policy(s) - policy(s + noise))**2).mean()
caps_loss = λ_T * temporal_loss + λ_S * spatial_loss
total_loss = policy_loss + caps_loss
```

Pros:
- More principled approach
- Doesn't affect reward structure

Cons:
- Requires custom SAC implementation
- More hyperparameters to tune

### Option 3: Lower Action Scale

Reduce `action_scale` from 0.02 to 0.01:
- Finer control granularity
- May need more steps to complete task

### Option 4: Action Filtering (Post-hoc)

Apply low-pass filter to actions during execution:
```python
filtered_action = α * action + (1 - α) * prev_action
```

Pros:
- No retraining needed
- Can tune α at deployment

Cons:
- Introduces lag
- May hurt performance on fast tasks

## Recommendation for Our Case

Start with **Option 1 (Action Rate Penalty)** as v11:
1. Simple to implement
2. Directly addresses observed twitching
3. Easy to tune (start with λ=0.05)

If insufficient, combine with:
- Lower action scale (0.01 instead of 0.02)
- Adjust target bonus to `z > 0.08` to fix loophole

## References

1. [Benchmarking Smoothness and Reducing High-Frequency Oscillations in Continuous Control Policies](https://arxiv.org/html/2410.16632v1) - Oct 2024
2. [CAPS: Regularizing Action Policies for Smooth Control](https://cs-people.bu.edu/rmancuso/files/papers/ICRA21_1616_FI.pdf) - ICRA 2021
3. [Deep RL for Robotics: Survey of Real-World Successes](https://www.annualreviews.org/content/journals/10.1146/annurev-control-030323-022510) - Annual Reviews
4. [Learning to Recover: Dynamic Reward Shaping](https://arxiv.org/html/2506.05516) - 2025
5. [RL in Continuous Action Spaces](https://medium.com/@amit25173/reinforcement-learning-in-continuous-action-spaces-4fc60897fa55)
6. [Reward Shaping for Robotic Hand Manipulation](https://www.sciencedirect.com/science/article/abs/pii/S0925231225008768) - ScienceDirect
7. [DiffSkill: Online Momentum Smoothing](https://www.sciencedirect.com/science/article/abs/pii/S0950705124008244) - 2024
