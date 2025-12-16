# Phase 4: RL Training Setup

## Goal

Set up PyTorch with ROCm and implement DQN training pipeline using Stable-Baselines3.

## Tasks

### 1. PyTorch ROCm Setup
- Install PyTorch 2.4.0 with ROCm 6.4.3 support
- Verify GPU detection and availability
- Test basic tensor operations on GPU

### 2. Stable-Baselines3 Setup
- Install stable-baselines3 and dependencies
- Verify DQN algorithm imports correctly
- Configure for AMD GPU usage

### 3. Training Script
- Implement `train.py` with DQN agent
- Configure hyperparameters (network size, learning rate, buffer size, etc.)
- Set up TensorBoard logging
- Add model checkpointing
- Training loop with progress tracking

### 4. Evaluation Script
- Implement `eval.py` for model evaluation
- Load trained models and run test episodes
- Visualize agent performance
- Record metrics (avg reward, success rate, episode length)

### 5. Baseline Training
- Run initial training for 100k steps
- Monitor TensorBoard for learning progress
- Verify reward curve trends upward
- Save best performing model

## Success Criteria

- PyTorch successfully uses AMD GPU
- Training runs without errors
- Agent performance improves over time (reward increases)
- Can load and evaluate trained models
- TensorBoard shows clear learning signal

## Timeline

Estimated 4-6 hours for complete setup and initial training run.
