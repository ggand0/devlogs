# Devlog 060: Reward Classifier Research for HIL-SERL

Comprehensive research on reward classifiers for Human-in-the-Loop Reinforcement Learning, including HIL-SERL's approach and alternative online training methods.

## Context

Training DrQ-v2 from scratch with terminal-only sparse rewards was not working (~50 intervention episodes, robot shaking). The HIL-SERL paper uses a reward classifier to provide dense per-step rewards instead of terminal-only rewards.

## What is a Reward Classifier?

A vision-based binary classifier that predicts success probability from camera images at every timestep.

**Terminal-only reward (problematic):**
```
Step 0:   reward = 0
Step 1:   reward = 0
...
Step 99:  reward = 0
Step 100: reward = 1 (success) or 0 (failure)
```

**Reward classifier (dense):**
```
Step 0:   reward = classifier(image) → 0.02
Step 1:   reward = classifier(image) → 0.03
...
Step 50:  reward = classifier(image) → 0.45 (getting closer)
...
Step 100: reward = classifier(image) → 0.98 (success state)
```

The classifier outputs continuous probability [0, 1], not binary. This provides a smooth gradient of "how close to success" each frame is.

## HIL-SERL Paper Approach

**Source**: [HIL-SERL Paper (arXiv 2410.21845)](https://arxiv.org/html/2410.21845v1)

### Training Data Collection
- **200 positive data points** (success states - task completed)
- **1000 negative data points** (everything else)
- Collected through ~10 human trajectories
- Takes approximately 5 minutes

### Classifier Specifications
- Binary classifier taking camera images as input
- Outputs probability of success state
- Target accuracy: **>95%** on evaluation data

### Key Finding: NO Online Fine-tuning
> "The classifier does not get fine-tuned during RL training. Instead, it serves as a fixed component that provides sparse reward signals."

The classifier is trained **offline before RL begins** and remains fixed during training.

## LeRobot HIL-SERL Implementation

**Source**: [LeRobot HIL-SERL Documentation](https://huggingface.co/docs/lerobot/hilserl)

### Configuration
```json
{
  "env": {
    "processor": {
      "reward_classifier": {
        "pretrained_path": "path_to_your_pretrained_model",
        "success_threshold": 0.7,
        "success_reward": 1.0
      }
    }
  }
}
```

### Training Workflow
1. Collect labeled dataset with `terminate_on_success: false` to get more positive examples
2. Train classifier offline with `lerobot-train`
3. Set `reward_classifier.pretrained_path` in config
4. Run RL training - classifier provides per-step rewards

### Optional Manual Annotation
> "Training a reward classifier is optional. You can start the first round of RL experiments by annotating the success manually with your gamepad or keyboard device."

## Problem: Classifier Reliability

Even with 95% offline accuracy, the classifier can produce false positives/negatives during online deployment due to:
- Distribution shift between training and deployment
- Novel states not seen during offline training
- Lighting/environmental changes

## Research: Online Reward Classifier Training

### Ever-Correcting Reward Classifier

**Source**: [Real-world RL from Suboptimal Interventions (arXiv 2512.24288)](https://arxiv.org/html/2512.24288)

Key finding:
> "Even when a reward classifier achieves up to 95% precision after offline training, it can still produce many false-negative or false-positive examples when deployed in online environments."

**Solution - Ever-Correcting Module:**
> "The classifier is retrained whenever newly corrected labels are added to the reward buffer, whether they correct false negatives or false positives."

This creates a feedback loop:
1. Human operator flags misclassifications during training
2. Corrected labels added to reward buffer
3. Classifier retrained with new data
4. Continues adapting to deployment conditions

### Unified Learning from Human Feedback

**Source**: [ACM Transactions on Human-Robot Interaction](https://dl.acm.org/doi/full/10.1145/3623384)

> "At every iteration, the human can provide demonstrations, corrections, or preferences, and the robot updates its reward model and identifies an optimal trajectory in real-time."

This approach:
- Makes no assumptions about tasks
- Learns reward model from scratch
- Compares human input to nearby alternatives
- Trains ensemble of reward models

### General RLHF Literature

**Source**: [RLHF Wikipedia](https://en.wikipedia.org/wiki/Reinforcement_learning_from_human_feedback)

> "The reward function being dynamically refined and adjusted to distributional shifts as the agent learns."

Both offline and online data collection models have been mathematically studied.

### Other Relevant Papers

| Paper | Approach | Source |
|-------|----------|--------|
| Hi-ORS | Human-in-the-loop Online Rejection Sampling | [arXiv 2510.26406](https://arxiv.org/html/2510.26406) |
| Interactive Imitation Learning | Real-time human guidance to correct behavior | [Frontiers](https://www.frontiersin.org/journals/robotics-and-ai/articles/10.3389/frobt.2025.1682437/full) |
| LILAC | Language corrections during execution | [ACM](https://dl.acm.org/doi/10.1145/3568162.3578623) |

## Comparison: Offline vs Online Classifier Training

| Aspect | Offline (HIL-SERL) | Online (Ever-Correcting) |
|--------|-------------------|--------------------------|
| Training time | Before RL starts | Continuous during RL |
| Adaptation | None | Adapts to deployment |
| Complexity | Simpler | More complex |
| RL stability | Stable (fixed reward) | May destabilize Q-learning |
| False positive handling | Collect more data pre-training | Fix on-the-fly |

## Why HIL-SERL Uses Fixed Classifier

1. **RL Stability**: Changing reward function mid-training can invalidate learned Q-values
2. **Simplicity**: Fewer moving parts, easier to debug
3. **Sufficient for their tasks**: 95% accuracy was enough with human interventions

## Implementation Gap

The LeRobot HIL-SERL implementation does NOT support:
- Online classifier fine-tuning during RL
- Using manual success annotations to update classifier
- Ever-correcting mechanism

This is a valid improvement that could be added but requires careful handling of RL stability.

## Practical Recommendations

### If classifier is unreliable:
1. **Collect more diverse training data** - Edge cases, different lighting, etc.
2. **Raise success_threshold** - More conservative, fewer false positives
3. **Use manual annotation** - Skip classifier, press success button yourself
4. **Hybrid approach** - Use classifier but override with manual input

### For training classifier:
1. Set `terminate_on_success: false` during data collection
2. Collect ~200 positive, ~1000 negative frames
3. Target >95% accuracy before deployment
4. Test on held-out data before RL training

## Available Datasets for Classifier Training

| Dataset | Frames | Notes |
|---------|--------|-------|
| `gtgando/so101_pick_lift_cube_locked_wrist_labeled` | 1,574 | 84x84, has videos |
| `gtgando/so101_pick_lift_cube_rl_labeled` | 1,000 | 640x480, full resolution |
| `outputs/hilserl_drqv2_scratch/dataset/` | 5,073 | Online collected with interventions |

## Next Steps

1. Train offline reward classifier on existing labeled dataset
2. Evaluate accuracy on held-out data
3. If >95% accuracy, use for RL training
4. If unreliable, consider:
   - Collecting more diverse data
   - Manual annotation during RL
   - Implementing ever-correcting mechanism

## References

1. HIL-SERL Paper: https://arxiv.org/html/2410.21845v1
2. LeRobot HIL-SERL Docs: https://huggingface.co/docs/lerobot/hilserl
3. Ever-Correcting Reward Classifier: https://arxiv.org/html/2512.24288
4. Unified Learning from Human Feedback: https://dl.acm.org/doi/full/10.1145/3623384
5. RLHF Wikipedia: https://en.wikipedia.org/wiki/Reinforcement_learning_from_human_feedback
6. Hi-ORS: https://arxiv.org/html/2510.26406
7. Interactive Imitation Learning Survey: https://www.frontiersin.org/journals/robotics-and-ai/articles/10.3389/frobt.2025.1682437/full
