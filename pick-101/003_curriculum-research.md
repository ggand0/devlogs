# Devlog: Curriculum Learning Research for Pick-and-Place

**Date**: 2024-12-14

## Problem

HER training is running but may take a while to converge. Meanwhile, researched alternative approaches:
- Sim2Real techniques for robot manipulation
- Curriculum learning to make the task easier
- Intermediate tasks that encourage grasping behavior (not pushing)

## Key Insight: Why Pushing Happens

Even with curriculum learning, if the intermediate task allows pushing as a solution, the agent will push. Examples:
- **Slope to goal**: Agent pushes cube downhill
- **Closer target**: Agent still pushes, just shorter distance

The intermediate task must **require** grasping to be solved.

## Intermediate Tasks That Force Grasping

### 1. Lift Task (robosuite standard)
Lift cube above height threshold. No target placement.
- Success = `cube_z > 0.10m` for N steps
- Can't be solved by pushing
- This is the canonical "learn to grasp" benchmark

### 2. Lift and Hold
Lift cube and maintain height for 3-5 seconds.
- Requires stable grasp, not lucky flick
- Punishes drops immediately

### 3. Lift Over Obstacle
Wall/barrier between cube and target. Only path is up-and-over.
- Physically impossible to push to goal
- Forces lift behavior

### 4. Cube on Pedestal
Cube starts elevated. Target on ground.
- Pushing knocks it off randomly (negative reward)
- Grasping gives controlled placement

### 5. Vertical Slot Placement
Target is a vertical slot/hole. Cube must drop in from above.
- Horizontal pushing can't achieve this
- Forces approach from above

## Recommended Curriculum

```
Stage 1: Reach (gripper near cube) - dense reward
Stage 2: Lift (cube above height) - sparse, use HER
Stage 3: Lift + Hold (maintain for N steps)
Stage 4: Place (full pick-and-place)
```

From robosuite benchmarks: SAC solves Lift but struggles with longer-horizon tasks.

## Resources

### Frameworks
- [robosuite](https://github.com/ARISE-Initiative/robosuite) - MuJoCo benchmark with Lift task
- [robomimic](https://robomimic.github.io/) - Imitation learning datasets for robosuite tasks

### Papers
- [Curriculum Learning for Sparse Rewards (IEEE 2024)](https://ieeexplore.ieee.org/document/10480429/) - Environment shifts for manipulation
- [RHER: Curriculum + HER](https://www.sciencedirect.com/science/article/abs/pii/S0893608021004020) - Decomposes tasks into progressive subtasks
- [Tactile-based Sim2Real](https://www.researchgate.net/publication/382987316) - 52% improvement with contact detection
- [CurricuLLM](https://arxiv.org/html/2409.18382) - LLM-generated task curricula

### Key Techniques for Sim2Real
1. **Domain randomization**: Vary physics params, textures, lighting
2. **Tactile sensing**: Contact detection compensates for vision occlusion
3. **Curriculum learning**: Start simple, increase difficulty
4. **HER**: Learn from failures by relabeling goals

## SO-101 Specific Findings

No one has published MuJoCo RL training for SO-101 pick-and-place. The arm is primarily used with:
- [LeRobot (Hugging Face)](https://github.com/huggingface/lerobot) - Imitation learning via teleoperation
- Uses ACT (Action Chunking Transformer) or diffusion policies, not SAC/PPO

We're pioneering the pure RL approach for this arm.

## Next Steps

1. Wait for HER training to complete (~67% remaining)
2. If HER fails, implement **Lift task** as intermediate training target
3. Transfer lift policy weights to initialize pick-and-place training
