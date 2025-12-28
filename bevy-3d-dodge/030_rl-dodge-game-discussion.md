# RL Dodge Game: Architecture and Training Discussion Summary

## MLP vs CNN for State-Based Observations

When explicit state information (positions, velocities) is available, MLPs outperform CNNs both theoretically and empirically.

### Why MLPs win with explicit velocity

- Velocity vectors directly encode projectile trajectories—exactly what the agent needs
- CNNs would need to *learn* to extract this from pixels or stacked frames
- No spatial structure to exploit; convolution's inductive bias doesn't help
- Lower sample complexity, faster training

### When CNNs become necessary

- Egocentric vision (camera attached to agent)
- Pixel-based observations without ground-truth state
- Agent must learn to infer positions and velocities from raw frames

### References

1. **Yarats et al. (2021)** - "Improving Sample Efficiency in Model-Free Reinforcement Learning from Images" (AAAI)
   - Demonstrates significant performance gap between pixel-based and state-based SAC
   - State-based serves as upper bound that pixel methods struggle to match
   - https://ojs.aaai.org/index.php/AAAI/article/view/17276

2. **Laskin et al. (2020)** - "CURL: Contrastive Unsupervised Representations for Reinforcement Learning" (ICML)
   - Found that pixel-based RL can be "nearly as data-efficient as state-based RL"—but only with sophisticated contrastive learning
   - Confirms state-based is the default efficiency winner
   - https://arxiv.org/abs/2004.04136

3. **Ceron & Castro (2021)** - "Measuring Progress in Deep Reinforcement Learning Sample Efficiency"
   - Documents that "sample efficiency is a lot better for state-based continuous control than for the pixel-based version"
   - https://arxiv.org/abs/2102.04881

4. **Raffin et al. (2020)** - "Low Dimensional State Representation Learning with Reward-shaped Priors"
   - Shows low-dimensional representations are "easier to associate with the reward signal"
   - https://arxiv.org/abs/2007.14808

---

## OpenAI Hide-and-Seek Architecture

The emergent behaviors came from multi-agent competition, not architectural cleverness.

### Architecture used

1. **Entity embeddings** - MLPs embedding each object's state vector
2. **Masked self-attention** - Transformer-style attention over entities (not pixels)
3. **LSTM** - Temporal memory for multi-step planning
4. **MLP action heads** - Separate heads for different action types

### Key insight

They used state-based observations, not pixels. The critic had access to full unmasked environment state. Interesting behaviors emerged from the autocurriculum induced by competition, not from the network architecture.

### Reference

- Baker et al. (2019) - "Emergent Tool Use From Multi-Agent Autocurricula" (ICLR 2020)
- https://arxiv.org/abs/1909.07528
- https://openai.com/index/emergent-tool-use/

---

## Why Agents Learn Boring "Shuffling" Behaviors

The agent found a local optimum: stay back, minimal movement, avoid risk. This satisfies survival but looks uninteresting.

### Root cause

This is primarily a **reward/environment design issue**, not an architecture problem. The MLP can represent complex policies—it's just not incentivized to.

### Architecture changes that won't help

- Bigger networks (no capacity problem)
- CNNs (state is already compact)
- Attention (no entity-relationship complexity)

### Architecture changes that might help marginally

- **LSTM/GRU** - For anticipating patterns or planning multi-step maneuvers
- **Action space redesign** - Continuous vs discrete can affect movement fluidity

---

## Solutions for Encouraging Interesting Behaviors

### Option 1: Reward shaping

- Bonus for close calls (projectile passes within threshold)
- Reward forward positioning or center-of-arena time
- Penalize stationary behavior
- Reward velocity/acceleration variance

### Option 2: Environment pressure

- Projectiles from all directions (including behind)
- Projectiles that predict agent position
- Shrinking safe zone
- Required pickups in dangerous areas

### Option 3: Scripted adversarial thrower

Predictive aiming heuristic:

```
target_position = agent_position + agent_velocity * lead_time
aim_direction = normalize(target_position - thrower_position)
```

Variations:
- Linear prediction with tuned lead time
- Noise injection to prevent counter-prediction
- Multi-shot patterns covering dodge directions
- Curriculum: basic aim → predictive aim as agent improves

### Option 4: Multi-agent setup

Full adversarial training with learned thrower agent.

---

## Multi-Agent Setup Design

### Reward structure

- **Dodger**: +1 survive, -1 if hit
- **Thrower**: +1 hit, -1 if dodger survives

### Thrower action space options

| Option | Action space | Complexity | Notes |
|--------|--------------|------------|-------|
| Aim only | `aim_angle` (continuous) | Low | Fixed fire rate, learns direction only |
| Aim + timing | `[aim_angle, fire_bool]` | Medium | Can learn to hold shots, bait movement |
| Full control | `[aim_angle, speed, fire]` | High | Most expressive, harder to train |
| Discrete | `{left, center, right, predicted, wait}` | Low | Simpler training, less nuanced |

**Recommendation**: Start with aim + timing (Option 2).

### Training approaches (increasing complexity)

1. **Naive simultaneous** - Both update every step; simple but potentially unstable
2. **Alternating** - Freeze one, train other for N steps; more stable
3. **Self-play with checkpoints** - Train against past versions; prevents overfitting
4. **Population-based** - Multiple agents; most robust, most complex

### Stabilization tricks

- Update thrower less frequently than dodger (e.g., every 4th step)
- Train thrower against mixture of current dodger (70%) and random policy (30%)

### Same algorithm for both?

Yes, both can use SAC or PPO. No need to handicap the thrower—balanced competition drives improvement. If unstable, slow down thrower updates first.

---

## Solving Sparse Reward for Thrower

Random aim rarely hits early in training, causing sparse signal.

### Solutions

**1. Distance-based shaping (recommended)**

```python
min_distance = closest_approach(projectile, dodger)
reward = 1.0 - (min_distance / max_distance)
if hit:
    reward += 10.0
```

**2. Aim accuracy reward**

```python
aim_error = angle_between(aim_direction, direction_to_dodger)
reward = -aim_error
```

**3. Curriculum on dodger difficulty**

- Phase 1: Dodger frozen
- Phase 2: Dodger slow
- Phase 3: Full speed

**4. Imitation warmstart**

Pre-train thrower with supervised learning on heuristic policy, then switch to RL.

**5. High fire rate early**

More shots = more accidental hits = more signal. Decay fire rate over training.

---

## Recommended Path Forward

1. **Start with scripted predictive thrower** - Quick to implement, tests if dodger can learn interesting counter-behavior
2. **Add reward shaping if needed** - Close-call bonuses, positioning rewards
3. **Move to multi-agent only if scripted approach plateaus** - Significant complexity jump, but enables open-ended skill progression
4. **Use distance-based reward shaping for thrower** - Solves cold-start problem without curriculum complexity
