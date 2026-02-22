# Devlog 009: From Imitation Learning to Reinforcement Learning - Roadmap

**Date**: December 13, 2025

## Overview

This devlog documents the journey from successfully training an IL policy for pick-and-place, to defining the roadmap for RL-based robot learning. The ultimate goal is building a **desktop cleanup robot** that can autonomously pick up bottles/cans and place them in a trash can.

---

## Context: The Journey So Far

### What I've Built

1. **SO-101 Robot Setup**: Leader-follower arm configuration with gripper-mounted camera
2. **IL Pipeline**: LeRobot-based imitation learning with ACT policy
3. **MuJoCo Simulation**: State-based RL environment achieving 100% grasp+lift success

### The IL Experience

I successfully trained two ACT policies via imitation learning:
- Green cube → white paper (November 2025)
- Red cube → green bowl (December 2025)

Both achieved "partial success" - the policies work sometimes but aren't robust. The process involved:
1. Manually recording 30 teleoperation demonstrations (~15 minutes of tedious work)
2. Training for 3-4 hours
3. Evaluating and finding it only works ~50% of the time
4. Realizing I'd need to record MORE demos to improve...

### The Problem with IL

After going through this process twice, the limitations became clear:

**The Data Treadmill**: Every improvement requires more manual demonstrations. To go from 50% to 80% success, I might need 50+ more episodes. To handle varied cube positions? More demos. Different objects? More demos. It never ends.

**Can't Compete at Scale**: Companies like Google (RT-2), OpenAI, and others are collecting millions of demonstrations with teams of operators. As an individual, I can record maybe 50 demos before losing patience. The data gap is insurmountable.

**No Self-Improvement**: If the policy fails in a new way, it can't learn from that failure. It only knows what I showed it.

---

## Motivation: Why Sim-to-Real RL?

### The Core Insight

**With RL, I never have to manually record demonstrations again.**

The entire training loop becomes:
1. Define reward function (once)
2. Let sim train overnight (automated)
3. Transfer to real robot (automated)
4. Fine-tune with HIL-SERL (semi-automated)

### What "Semi-Automated" Means

HIL-SERL (Human-in-the-Loop SERL) still requires human presence, but it's fundamentally different from IL:

| Aspect | Imitation Learning | HIL-SERL Fine-tuning |
|--------|-------------------|---------------------|
| **Human role** | Demonstrate perfect behavior | Intervene only when robot fails |
| **Reward signal** | None (just copy demos) | Automated by reward classifier |
| **Success detection** | Manual labeling | Classifier determines automatically |
| **Learning** | Only from demos | From exploration + interventions |
| **Time commitment** | Record every episode | Watch and occasionally correct |

### The Reward Classifier: Key to Automation

In HIL-SERL, a learned **reward classifier** watches the camera feed and automatically determines:
- Is the cube in the gripper? → reward
- Is the cube in the bowl? → success, end episode
- Did the cube fall? → failure, reset

This means I don't have to manually label every timestep. The classifier (trained on a small set of labeled images) handles it. My job is just to:
1. Intervene when the robot does something dangerous/stuck
2. Occasionally correct the robot to help exploration
3. Watch the success rate go up

### The Training Pyramid

```
                    ┌─────────────────┐
                    │  HIL-SERL       │  ← 1-2 hours, semi-automated
                    │  (real robot)   │     human intervenes when needed
                    └────────┬────────┘     classifier gives rewards
                             │
                    ┌────────▼────────┐
                    │  Sim-to-Real    │  ← instant, automated
                    │  Transfer       │     just load weights
                    └────────┬────────┘
                             │
         ┌───────────────────▼───────────────────┐
         │          MuJoCo RL Training           │  ← overnight, fully automated
         │  (image-based, domain randomization)  │     no human involvement
         └───────────────────────────────────────┘
```

Most of the learning happens in sim (free, unlimited, overnight). Real robot time is minimal and semi-automated.

### Comparison: IL vs Sim-to-Real RL

| Metric | Pure IL | Sim-to-Real RL + HIL-SERL |
|--------|---------|---------------------------|
| **Manual demo recording** | 30-100 episodes | 0 episodes |
| **Manual labeling** | Every episode | Train classifier once |
| **Real robot time** | All training | Fine-tuning only (1-2 hrs) |
| **Can improve beyond demos** | No | Yes |
| **Overnight training** | No | Yes (in sim) |
| **Discovers novel strategies** | No | Yes |

### But Wait - Don't You Need Data for the Reward Classifier?

Yes, but it's fundamentally different:

**Reward Classifier Training** (one-time, ~20 minutes):
1. Record ~15-20 episodes with random/teleoperated behavior
2. Set `terminate_on_success: false` to collect frames after task completion
3. The classifier learns: "what does success look like?"
4. Just needs to classify frames, not learn motor skills

**Compare to IL Training**:
1. Record 50+ episodes of **perfect demonstrations**
2. Every demo must successfully complete the task
3. Policy learns the entire skill from these demos
4. Quality matters - bad demos = bad policy

The classifier is a simple binary classifier (success/failure on a frame). The RL policy is learning a complete sensorimotor skill. The data requirements are orders of magnitude different.

### What Does the Human Actually Do in HIL-SERL?

During HIL-SERL fine-tuning, the human:

1. **Watches** the robot attempt the task
2. **Intervenes** (press button) when robot is stuck or dangerous
3. **Guides** the robot past the stuck point using gamepad/leader arm
4. **Releases** control back to the policy
5. **Repeat** as policy improves

The key difference from IL:
- **IL**: "Let me show you exactly how to do this 50 times"
- **HIL-SERL**: "You try it, I'll help when you're stuck"

As training progresses, intervention rate drops. A good run might start at 80% intervention and end at 10%. The human effort decreases over time, unlike IL where every demo is full effort.

---

## Part 1: Imitation Learning Experiments

### Red Cube to Bowl Task

After successfully training a green cube to paper policy in November 2025, I trained a new policy for picking a red cube and placing it in a green bowl.

#### Data Collection

Recorded 30 demonstration episodes using leader-follower teleoperation:

```bash
uv run lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=ggando_so101_follower \
    --robot.cameras="{ gripper_cam: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}}" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=ggando_so101_leader \
    --display_data=true \
    --dataset.repo_id=gtgando/so101_red_cube_to_bowl \
    --dataset.num_episodes=30 \
    --dataset.episode_time_s=20 \
    --dataset.fps=30 \
    --dataset.single_task="Pick up the red cube and place it in the green bowl"
```

**Lessons learned**:
- Episode duration of 20s was too long - had ~5s idle time per episode
- Should use 12-15s for this task
- `--dataset.video_backend=pyav` required (torchcodec had FFmpeg issues)

#### Training

```bash
uv run lerobot-train \
    --dataset.repo_id=gtgando/so101_red_cube_to_bowl \
    --dataset.video_backend=pyav \
    --policy.type=act \
    --output_dir=outputs/train/act_so101_red_cube_bowl \
    --job_name=act_so101_red_cube_bowl \
    --policy.device=cuda \
    --wandb.enable=false \
    --policy.repo_id=gtgando/act_so101_red_cube_bowl_policy \
    --batch_size=8 \
    --steps=100000 \
    --save_freq=20000
```

**Results**:
- Training time: ~3.5 hours on AMD GPU (ROCm 6.4)
- Final loss: 0.043 (99% improvement from start)
- Epochs: 45.48
- Model: 52M parameters

#### Evaluation

```bash
uv run lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=ggando_so101_follower \
    --robot.cameras="{ gripper_cam: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}}" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=ggando_so101_leader \
    --display_data=true \
    --dataset.repo_id=gtgando/eval_so101_red_cube_bowl \
    --dataset.num_episodes=10 \
    --dataset.episode_time_s=15 \
    --dataset.reset_time_s=10 \
    --dataset.single_task="Pick up the red cube and place it in the green bowl" \
    --policy.path=gtgando/act_so101_red_cube_bowl_policy
```

**Result**: Partial success - policy sometimes completes the task.

### IL Summary

| Task | Dataset | Model | Result |
|------|---------|-------|--------|
| Green cube to paper | [so101_pick_lift_cube](https://huggingface.co/datasets/gtgando/so101_pick_lift_cube) | [act_so101_cube_policy](https://huggingface.co/gtgando/act_so101_cube_policy) | Partial success |
| Red cube to bowl | [so101_red_cube_to_bowl](https://huggingface.co/datasets/gtgando/so101_red_cube_to_bowl) | [act_so101_red_cube_bowl_policy](https://huggingface.co/gtgando/act_so101_red_cube_bowl_policy) | Partial success |

---

## Part 2: The Case for Reinforcement Learning

### Why Not Just Improve IL?

Imitation learning has fundamental limitations:

1. **Data scaling problem**: Can't compete with big corporations (Google RT-2, OpenAI) who collect millions of demonstrations
2. **Ceiling effect**: Policy can only be as good as the demonstrations
3. **Manual effort**: Recording demonstrations is tedious and doesn't scale
4. **No self-improvement**: Can't learn from failures or discover novel strategies

### Why RL?

1. **Automated training**: Sim training runs overnight, no human involvement
2. **Unlimited data**: Simulation provides infinite samples
3. **Exploration**: Can discover strategies humans wouldn't think of (like the "bump and grab" behavior I observed in MuJoCo training)
4. **Self-improvement**: Learns from failures, not just successes
5. **Robustness**: RL policies often handle edge cases better

### Current RL Progress

Already achieved 100% success on grasp + lift task in MuJoCo with state-based RL:

- **Project**: `/home/gota/ggando/ml/explore-mujoco/`
- **Task**: Pick up cube, lift to target height, hold for 150 steps
- **Success rate**: 100% (10/10 deterministic evaluation)
- **Training**: 1M steps, ~4 hours on RTX 4090
- **Interesting behavior**: Agent discovered "bump and grab" strategy - nudges cube into graspable orientation before closing gripper

---

## Part 3: The Sim-to-Real Challenge

### The Big Question

State-based RL works in simulation, but how do we transfer to real robot?

### Sim-to-Real Options Explored

#### Option 1: State-based RL + Vision Pipeline

```
MuJoCo RL (state-based) → Add cube detection → Real robot
```

**Pros**: Uses working policy
**Cons**: Need to build/tune cube detection pipeline

#### Option 2: Image-based RL in Sim

```
MuJoCo RL (image-based) → Domain randomization → Real robot
```

**Pros**: End-to-end, no perception pipeline
**Cons**: Harder to train, needs more samples

#### Option 3: Sim RL → HIL-SERL Fine-tune

```
MuJoCo RL → Transfer to real → HIL-SERL fine-tuning
```

**Pros**: Best of both worlds - sim efficiency + real robot adaptation
**Cons**: Still needs some real robot time

### Decision: Sim + HIL-SERL Hybrid

The chosen approach:

1. **Sim training**: Automated, overnight, unlimited samples
2. **Sim-to-real transfer**: Try it, might work (~50% chance for pick-and-place)
3. **HIL-SERL fine-tune**: Close the gap with human interventions if needed

This balances automation (RL goal) with pragmatism (real robots are hard).

### Honest Assessment of Sim-to-Real

**When it works well**:
- Locomotion (walking robots)
- Coarse manipulation (pushing, reaching)

**When it struggles**:
- Contact-rich tasks (grasping)
- Precise manipulation
- Deformable objects

**For pick-and-place**: Expect ~50% initial success. The sim pre-training isn't wasted - it provides a much better starting point than training from scratch on real.

---

## Part 4: Image-Based RL - Next Milestone

### Why Image-Based?

For the desktop cleanup goal (bottles/cans), vision is essential:
- Objects vary in shape, size, color
- Can't hardcode positions
- Need to generalize to new objects

### Technical Requirements

#### 1. Add Camera to MuJoCo Scene

```xml
<!-- Gripper-mounted camera -->
<camera name="gripper_cam"
        mode="fixed"
        pos="0 0 0.05"
        euler="0 90 0"
        fovy="60"/>
```

#### 2. Modify Environment for Image Observations

```python
# Change observation space
self.observation_space = spaces.Box(
    low=0, high=255,
    shape=(3, 128, 128),
    dtype=np.uint8
)

def _get_obs(self):
    self._renderer.update_scene(self.data, camera="gripper_cam")
    image = self._renderer.render()
    return np.transpose(image, (2, 0, 1))  # CHW format
```

#### 3. Domain Randomization

- Lighting variations
- Texture/color randomization
- Camera pose perturbations
- Physics noise (friction, mass)

#### 4. Sample-Efficient Algorithm: DrQ-v2

**DrQ-v2** (Data-regularized Q-learning v2) is the state-of-the-art for pixel-based RL.

Key insight: Apply **random image shifts** as data augmentation.

```python
image = random_shift(image, pad=4)  # shift by up to 4 pixels
```

This simple trick provides 10-100x sample efficiency improvement over vanilla SAC with CNN.

| Algorithm | Sample Efficiency | Complexity |
|-----------|------------------|------------|
| SAC + CNN | Poor | Simple |
| DrQ-v2 | Excellent | Simple |
| CURL/RAD | Good | Medium |
| Dreamer/TD-MPC | Excellent | Complex |

---

## Part 5: Full Roadmap

### Phase 1: MuJoCo RL ✅
- [x] State-based RL for grasp + lift (100% success)

### Phase 2: Image-Based RL (Next)
- [ ] Add camera to MuJoCo scene
- [ ] Create image-based environment
- [ ] Implement DrQ-v2 or SAC with CNN
- [ ] Add domain randomization
- [ ] Train and validate in sim

### Phase 3: Sim-to-Real Transfer
- [ ] Deploy policy on real SO-101
- [ ] Test with real camera input
- [ ] Measure success rate

### Phase 4: HIL-SERL Fine-tuning (if needed)
- [ ] Use sim-trained weights as initialization
- [ ] Set up HIL-SERL environment
- [ ] Fine-tune with human interventions

### Phase 5: Generalize to Objects
- [ ] Add object detection (YOLO or custom)
- [ ] Train on varied objects (bottles, cans)
- [ ] Different grasp strategies per object type

### Phase 6: Desktop Cleanup System
- [ ] Detect → Pick → Place loop
- [ ] Handle multiple objects
- [ ] Error recovery
- [ ] Maybe: VLA for language conditioning ("pick up the red bottle")

---

## Key Decisions Made

1. **RL over IL**: Scalable, automated, self-improving
2. **Sim-first**: Cheap samples, no hardware risk, overnight training
3. **Image-based for real deployment**: Required for object generalization
4. **HIL-SERL as backup**: Closes sim-to-real gap if needed
5. **IL for bootstrapping only**: Can use IL demos to warm-start RL

---

## Resources

### Repositories
- IL experiments: `/home/gota/ggando/ml/so101-playground/`
- MuJoCo RL: `/home/gota/ggando/ml/explore-mujoco/`
- HuggingFace datasets: `gtgando/so101_*`

### Papers
- [ACT: Learning Fine-Grained Bimanual Manipulation](https://arxiv.org/abs/2304.13705)
- [DrQ-v2: Mastering Visual Continuous Control](https://arxiv.org/abs/2107.09645)
- [HIL-SERL: Human-in-the-Loop RL](https://arxiv.org/abs/2410.21845)

### Next Steps
1. Add camera to MuJoCo lift_cube.xml
2. Create PickCubeImageEnv with image observations
3. Train with DrQ-v2 or augmented SAC
4. Add domain randomization
5. Attempt sim-to-real transfer

---

*This devlog serves as context for continuing the RL journey and writing a blog post about the progress.*
