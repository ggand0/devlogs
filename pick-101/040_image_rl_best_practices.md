# Devlog 040: Best Practices for Image-Based Pick & Place RL

Research summary of current best practices for visual reinforcement learning in robotic manipulation tasks.

## 1. Data Augmentation (Critical)

DrQ-v2's key insight: **image augmentation prevents encoder overfitting**.

### Random Shift Augmentation
- Pad 84×84 image by 4 pixels (repeat boundary pixels)
- Random crop back to 84×84 (±4 pixel shift)
- Apply bilinear interpolation on shifted image
- This simple augmentation is the primary reason DrQ-v2 works

### Performance
- DrQ-v2 achieves 96 FPS throughput (3.4× faster than DrQ)
- Can solve humanoid locomotion from pixels (previously unattainable by model-free RL)
- Most tasks train in ~8 hours on single GPU

**Sources:**
- [DrQ-v2 Paper](https://arxiv.org/abs/2107.09645)
- [DrQ-v2 GitHub](https://github.com/facebookresearch/drqv2)
- [Original DrQ Paper](https://arxiv.org/abs/2004.13649)

## 2. Curriculum Learning

### Why It Helps
- Acts as scaffold directing exploration toward meaningful milestones
- Provides intermediate goals in delayed reward domains
- Without curriculum, agent might never grasp an object, stalling progress entirely

### Implementation Strategies
1. **Task decomposition**: Break complex tasks into sequential subtasks
2. **Two-stage reward curriculum**: First maximize simple reward, then transition to full complex reward
3. **Parameterized goals**: Gradually increase complexity (e.g., object distance)

### For Grasping
- Early training: Place objects closer to gripper
- Agent learns grasping before precise placement
- Example progression: Stage 2 (grasped) → Stage 3 (near) → Stage 4 (far)

**Sources:**
- [Curriculum Learning for RL Domains](https://jmlr.org/papers/volume21/20-212/20-212.pdf)
- [Sample-Efficient Curriculum RL](https://arxiv.org/html/2410.16790v1)
- [Curriculum Learning for Multi-Step Manipulation](https://repositories.lib.utexas.edu/items/d2251975-b0b3-44a1-aaad-6953ebe46387)

## 3. Reward Shaping

### Key Findings
- Dense rewards help but must avoid exploitation
- Performance most impacted by curriculum; reward shaping had less impact
- Multi-policy architecture: Train each subtask with separate policy

### Visual Reward Learning
- Demonstrations reduce need for laborious reward shaping
- MT-CNN reward models generate informative, smooth reward signals
- Trained policies outperform those with sparse oracle rewards

**Sources:**
- [Comparing Reward Shaping, Visual Hints, and Curriculum Learning](https://www.researchgate.net/publication/324983318_Comparing_Reward_Shaping_Visual_Hints_and_Curriculum_Learning)
- [Robot Policy Learning from Demonstrations and Visual Rewards](https://www.sciencedirect.com/science/article/abs/pii/S0921889025004087)

## 4. Sim-to-Real Transfer

### Domain Randomization
- Vary textures, lighting, camera pose, dynamics parameters
- RCAN (Randomized-to-Canonical Adaptation Networks): 70% zero-shot grasp success, 2× better than DR alone
- DROPO: Offline domain randomization with likelihood-based parameter estimation

### Visual Modality Choices
- **Point cloud > RGB** for sim-to-real (TRANSIC 2024)
- RGB suffers from lighting, texture differences
- Well-calibrated point clouds bypass these issues

### Key Challenge
Gap between simulated and real data degrades policy performance. Common approaches:
1. Domain randomization
2. Domain adaptation (requires real data)
3. Sim-to-sim adaptation (RCAN approach)

**Sources:**
- [TRANSIC: Sim-to-Real Policy Transfer](https://transic-robot.github.io/)
- [Sim-to-Real Transfer Survey](https://arxiv.org/abs/2009.13303)
- [RCAN Paper](https://arxiv.org/abs/1812.07252)
- [DROPO](https://www.sciencedirect.com/science/article/pii/S0921889023000714)
- [Awesome Sim2Real GitHub](https://github.com/LongchaoDa/AwesomeSim2Real)

## 5. Sample Efficiency

### Benchmarks
| Method | Training Time | Notes |
|--------|--------------|-------|
| VPG (Visual Pushing for Grasping) | ~2000 transitions (~5.5 hrs real) | Learns pushing + grasping synergy |
| 10 demos + DrQ | 15-50 min real training | Sparse reward manipulation |
| 800k grasps (14 robots, 2 months) | 82.5% accuracy | Brute force approach |

### Techniques
- Parallel environments (8+ recommended)
- n-step returns (used in DrQ-v2)
- Large replay buffer (~0.5× training steps)
- Demonstrations dramatically reduce exploration burden

**Sources:**
- [VPG: Visual Pushing for Grasping](https://vpg.cs.princeton.edu/)
- [RL for Pick and Place Survey](https://www.mdpi.com/2218-6581/10/3/105)

## 6. Architecture Considerations

### Action Space
- Joint position actions transfer better than velocity
- TRANSIC uses joint position for successful sim-to-real

### Visual Input
- 84×84 is standard for DrQ-v2
- Frame stack of 3 provides temporal information
- Consider wider FOV for manipulation (match real hardware)

### Network Size
- DrQ-v2 uses relatively small networks
- [256, 256] MLP works for manipulation
- 32-channel encoder sufficient for simple tasks

## 7. Recent Advances (2024-2025)

### Diffusion Models
- Primarily used with imitation learning for trajectory planning
- Being adapted for integration with RL
- [Diffusion Models for Robotic Manipulation Survey](https://www.frontiersin.org/journals/robotics-and-ai/articles/10.3389/frobt.2025.1606247/full)

### Human-in-the-Loop RL
- Precise and dexterous manipulation via human feedback
- [Science Robotics 2024](https://www.science.org/doi/10.1126/scirobotics.ads5033)

### Foundation Models for Sim-to-Real
- Survey of methods using foundation models for transfer
- [Embodied AI Paper List](https://github.com/HCPLab-SYSU/Embodied_AI_Paper_List)

## 8. Recommendations for SO-101 Project

### Current Status
- ✅ DrQ-v2 with augmentation
- ✅ Curriculum learning (stages)
- ✅ Dense reward shaping (v13)
- ✅ Parallel environments (8)

### Next Steps to Try
1. **Match camera FOV** (sim 75° → real 130°)
2. **Domain randomization**: lighting, textures, cube color/size
3. **Add demonstrations**: Even 5-10 demos can help dramatically
4. **Try stage 2 first**: Start with grasped cube, learn lifting
5. **Point cloud input**: Better sim-to-real than RGB

### Expected Timeline
- Simple manipulation: 300k-500k steps
- Pick & place: 1M-3M steps
- Complex tasks: 3M-10M steps

## References Summary

| Topic | Key Paper | Year |
|-------|-----------|------|
| Visual RL | DrQ-v2 | 2021 |
| Sim-to-Real | TRANSIC | 2024 |
| Curriculum | Narvekar et al. | 2020 |
| Grasping | VPG | 2018 |
| Domain Randomization | RCAN | 2018 |
| Survey | RL for Pick and Place | 2021 |
