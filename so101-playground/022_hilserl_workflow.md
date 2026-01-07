# Devlog 022: HIL-SERL Workflow for SO-101

## What is HIL-SERL?

**Human-in-the-Loop Sample Efficient Reinforcement Learning** combines:
1. **Sim-pretrained policy**: DrQ-v2 trained in simulation
2. **Real robot fine-tuning**: Online RL with human corrections
3. **Learned reward classifier**: Predicts success from images

The key insight is that humans can correct the robot when it makes mistakes, providing high-quality demonstrations that are added to the replay buffer for training.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      HIL-SERL System                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Actor      │    │   Learner    │    │   Reward     │  │
│  │  (Real Robot)│◄──►│   (GPU)      │◄───│  Classifier  │  │
│  └──────┬───────┘    └──────┬───────┘    └──────────────┘  │
│         │                   │                               │
│         ▼                   ▼                               │
│  ┌──────────────┐    ┌──────────────┐                      │
│  │ Leader Arm   │    │ Replay       │                      │
│  │ (Human)      │    │ Buffer       │                      │
│  └──────────────┘    └──────────────┘                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Components

1. **Actor (Real Robot)**
   - Runs policy inference at 10 Hz
   - Sends observations to learner
   - Receives updated policy weights

2. **Learner (GPU)**
   - Trains DrQ-v2 with incoming data
   - Updates actor's policy periodically
   - Uses UTD (Update-to-Data) ratio for sample efficiency

3. **Reward Classifier**
   - CNN trained on labeled success/failure images
   - Predicts reward in real-time
   - Terminates episode on success detection

4. **Human Intervention**
   - Leader arm tracks follower during policy execution
   - Human can grab leader to override policy
   - Intervention data marked and added to buffer

## Prerequisites

1. **Sim-trained checkpoint** at `pick-101/runs/image_rl/20260106_221053`
2. **Reward classifier** at `gtgando/so101_reward_classifier`
3. **URDF** for FK at `models/so101_new_calib.urdf`
4. **Hardware**: SO-101 follower + leader arms, wrist camera

## Step-by-Step Guide

### Step 1: Convert Sim Checkpoint

Once sim training completes, convert from RoboBase to LeRobot format:

```bash
# Check sim training status
ls /home/gota/ggando/ml/pick-101/runs/image_rl/20260106_221053/snapshots/

# Convert checkpoint (script TBD)
python scripts/convert_robobase_to_lerobot.py \
  --input /path/to/best_snapshot.pt \
  --output models/drqv2_sim_pretrained
```

### Step 2: Verify Configuration

Check `train_hilserl_drqv2.json`:

```json
{
  "policy": {
    "type": "drqv2",
    "pretrained_path": "models/drqv2_sim_pretrained",
    "frame_stack": 3,
    "input_features": {
      "observation.images.gripper_cam": {"shape": [9, 84, 84]},
      "observation.state": {"shape": [54]}
    }
  },
  "reward_classifier": {
    "repo_id": "gtgando/so101_reward_classifier"
  },
  "env": {
    "wrapper": {
      "add_full_proprioception": true,
      "control_mode": "leader"
    }
  }
}
```

### Step 3: Connect Hardware

```bash
# Check robot connections
ls /dev/ttyACM*

# Should see:
# /dev/ttyACM0 - Follower arm
# /dev/ttyACM1 - Leader arm

# Check camera
ls /dev/video*
# /dev/video0 - Wrist camera
```

### Step 4: Start Training

```bash
cd /home/gota/ggando/ml/so101-playground

# Run HIL-SERL training
python -m lerobot.scripts.rl.train_hilserl \
  --config train_hilserl_drqv2.json
```

### Step 5: Human Intervention Protocol

During training:

1. **Policy executes**: Robot moves autonomously
2. **Monitor performance**: Watch for mistakes
3. **Intervene when needed**:
   - Grab leader arm to take control
   - Guide robot to correct behavior
   - Release to return control to policy
4. **Episode ends**: On success (reward=1) or timeout

The system automatically:
- Detects intervention via leader-follower tracking error
- Records human corrections
- Adds intervention data to replay buffer
- Trains on mixed policy + human data

### Step 6: Monitor Training

```bash
# Watch logs
tail -f outputs/hilserl_drqv2_so101/train.log

# Or use wandb
# Enable in config: "wandb": {"enable": true}
```

Key metrics to watch:
- `success_rate`: Should increase over time
- `intervention_rate`: Should decrease as policy improves
- `critic_loss`: Should stabilize
- `actor_loss`: Should decrease

## How Intervention Works

### Automatic Detection

The `GearedLeaderAutomaticControlWrapper` monitors:

```python
# Tracking error between leader and follower
error = np.linalg.norm(follower_pos - leader_pos)

# Start intervention when variance spikes (human grabbed leader)
if np.var(error_queue[-2:]) > intervention_threshold:
    is_intervention = True

# End intervention when error stabilizes (human released)
if np.var(error_queue) < release_threshold:
    is_intervention = False
```

### Data Flow

```
Policy Action ──┬──► [Intervention?] ──► Robot Executes
                │         │
                │         ▼
                │   Human Override ──► Leader Arm
                │
                ▼
        Replay Buffer ◄── (action, is_intervention flag)
```

## Troubleshooting

### Robot not moving
- Check USB connections
- Verify motor torque is enabled
- Check URDF path is correct

### No reward signal
- Test reward classifier separately
- Check camera is connected
- Verify image preprocessing matches training

### High intervention rate
- Policy may need more sim pretraining
- Check state observation matches sim
- Verify action scaling is correct

### Training unstable
- Reduce learning rate
- Increase batch size
- Check reward normalization

## Expected Timeline

1. **0-1k steps**: High intervention, policy adapting
2. **1k-5k steps**: Decreasing intervention, improving success
3. **5k-20k steps**: Refinement, consistent success
4. **20k+ steps**: Policy converges, minimal intervention needed

## Files

- `train_hilserl_drqv2.json` - Main config
- `lerobot/scripts/rl/train_hilserl.py` - Training script
- `lerobot/scripts/rl/gym_manipulator.py` - Environment wrappers
- `lerobot/policies/drqv2/` - DrQ-v2 policy implementation
