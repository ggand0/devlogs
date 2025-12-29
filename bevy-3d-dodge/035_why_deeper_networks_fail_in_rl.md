# Why Deeper Networks Fail in RL (Unlike Supervised Learning)

## The Paradox

In supervised learning (vision, NLP), deeper networks with millions/billions of parameters consistently outperform smaller ones. **This trend has largely eluded deep RL** - larger networks do not lead to performance improvement and often make things worse.

---

## Root Cause: Non-Stationarity

### Supervised Learning: Static World
- Training data distribution is **fixed**
- Target outputs are **constant**
- More capacity = better function approximation
- Deeper networks can learn hierarchical features

### Reinforcement Learning: Moving Target
- Data distribution **changes as policy improves**
- Target (future reward) is **non-stationary** and depends on current policy
- Agent generates its own training data through exploration
- What was "correct" early in training may be wrong later

---

## Key Failure Modes

### 1. Primacy Bias
Deep RL agents have a **tendency to overfit to early experiences** and ignore useful evidence encountered later.

> "An agent impacted by the primacy bias tends to be incapable of leveraging subsequent data because of the accumulated effect of the initial overfitting."

- Larger networks = more parameters to overfit
- Early random experiences get "locked in"
- Network loses ability to adapt to better strategies discovered later

**Source**: [The Primacy Bias in Deep Reinforcement Learning (ICML 2022)](https://arxiv.org/abs/2205.07802)

### 2. Loss of Plasticity
Neural networks gradually **lose the ability to learn from new experiences** - analogous to solidification of neural pathways in biological brains.

- RL's non-stationary nature exacerbates this
- Networks "crystallize" around early representations
- Deeper networks have more layers that can crystallize

**Source**: [Neuroplastic Expansion in Deep Reinforcement Learning](https://arxiv.org/html/2410.07994v3)

### 3. The Deadly Triad
RL algorithms rely on three components that each introduce instability:
1. **Off-policy learning** - using old data that may not reflect current policy
2. **Bootstrapping** - updating estimates based on other estimates (moving target)
3. **Function approximation** - neural networks that can diverge

> "Conventional optimization techniques from supervised learning are not designed to handle the non-stationary nature of RL."

**Source**: [Training Larger Networks for Deep Reinforcement Learning](https://arxiv.org/abs/2102.07920)

### 4. Memory Effects
> "Neural networks exhibit a memory effect, where transient non-stationarities can permanently impact the latent representation and adversely affect generalisation performance."

The network "remembers" the confusing early phase of training and this corrupts later learning.

**Source**: [Transient Non-stationarity and Generalisation in Deep RL](https://openreview.net/forum?id=Qun8fv4qSby)

---

## Why Width > Depth for RL

| Aspect | Deeper Network | Wider Network |
|--------|---------------|---------------|
| Gradient flow | More layers = vanishing/exploding gradients | Single layer = stable gradients |
| Representation locking | Each layer can "crystallize" | Fewer layers to crystallize |
| Optimization landscape | More complex, more local minima | Simpler, easier to navigate |
| Primacy bias impact | Affects all layers cumulatively | Less cumulative effect |

**Empirical finding**: "A deeper MLP does not consistently improve performance, while a wider MLP generally provides better results."

---

## Our Experiment Confirms This

| Architecture | Best Eval | Success Rate |
|-------------|-----------|--------------|
| [256, 256] (2-layer) | 1007.27 | 40% |
| [256, 256, 256] (3-layer) | 860.22 | 0% |

The 3-layer network likely:
1. Overfitted to early random exploration
2. Lost plasticity before learning good dodging strategy
3. Locked in suboptimal representations across 3 layers instead of 2

---

## Proposed Solutions from Research

### 1. Periodic Resets
Periodically re-initialize the last few layers while preserving replay buffer.

> "Applying resets to SAC alleviates the effects of the primacy bias."

**Source**: [The Primacy Bias in Deep RL](https://proceedings.mlr.press/v162/nikishin22a/nikishin22a.pdf)

### 2. Neuroplastic Expansion
Start with a smaller network and dynamically grow it during training.

> "Dynamically growing the network from a smaller initial size maintains learnability throughout training."

### 3. Iterated Relearning
Periodically transfer knowledge to a freshly initialized network.

> "A fresh network experiences less non-stationarity during training."

### 4. Representation Decoupling
Separate representation learning from RL training (e.g., contrastive learning on observations).

---

## Practical Recommendations

1. **Stick with 2 layers** - This is the proven architecture for SAC/continuous control
2. **Prefer width over depth** - [512, 512] over [256, 256, 256]
3. **Use standard hyperparameters** - They're tuned for typical network sizes
4. **Consider resets** - If training for very long, periodic layer resets can help
5. **Don't scale up naively** - More capacity requires special techniques

---

## References

- [The Primacy Bias in Deep Reinforcement Learning (Nikishin et al., ICML 2022)](https://arxiv.org/abs/2205.07802)
- [Training Larger Networks for Deep Reinforcement Learning](https://arxiv.org/abs/2102.07920)
- [A Framework for Training Larger Networks for Deep RL](https://link.springer.com/article/10.1007/s10994-024-06547-6)
- [Neuroplastic Expansion in Deep Reinforcement Learning](https://arxiv.org/html/2410.07994v3)
- [Deep Reinforcement Learning amidst Lifelong Non-Stationarity](https://openreview.net/pdf?id=P1OwHAhDVbd)
- [Transient Non-stationarity and Generalisation in Deep RL](https://openreview.net/forum?id=Qun8fv4qSby)
- [It's Time to Move On: Primacy Bias and Why It Helps to Forget](https://iclr-blogposts.github.io/2024/blog/primacy-bias-and-why-it-helps-to-forget/)
- [SAC - Stable Baselines3 Documentation](https://stable-baselines3.readthedocs.io/en/master/modules/sac.html)
- [Soft Actor-Critic - Spinning Up](https://spinningup.openai.com/en/latest/algorithms/sac.html)
