# SAC 360° Full Coverage - 4M Steps Results

## Experiment
Testing full 360° spawn coverage (projectiles from any direction).

## Configuration
- `spawn_angle_degrees: 180` (±180° = 360° full circle)
- Level 2 difficulty
- 4M training steps
- [256, 256] MLP architecture
- 4 parallel environments

## Results

| Metric | Value |
|--------|-------|
| Training time | 4h 36m |
| Final eval | 415.55 ± 384.84 |
| Best eval | 725.84 |
| Final ep length | 488 ± 348 |
| Peak ep length | 244 (training) |
| Throughput | ~241 fps |

## Comparison to 60° (120° fan)

| Setup | Best Eval | Final Eval | Ep Length |
|-------|-----------|------------|-----------|
| 60° @ 2M | 1007 | 660 ± 307 | ~715 |
| 360° @ 4M | 726 | 416 ± 385 | ~488 |

## Analysis

1. **360° is significantly harder**: Despite 2x training steps, performance is ~40% worse than 60°

2. **High variance**: ±385 reward variance indicates inconsistent survival - agent sometimes lasts but often dies quickly

3. **Short episodes**: Average 144-488 steps vs 700+ for 60° setup

4. **Learning plateau**: Final reward (416) close to best (726) suggests the agent hit a ceiling

## Possible Improvements

1. **Longer training**: May need 8M+ steps for this difficulty
2. **Easier intermediate step**: Try 90° or 120° before jumping to 360°
3. **Architecture changes**: Might need larger network for more complex spatial reasoning
4. **Curriculum learning**: Start with narrow angles, gradually widen

## Conclusion

360° full coverage is substantially harder than 120° fan. The agent learned some dodging behavior but struggles with attacks from behind. Consider curriculum learning or intermediate difficulty steps.
