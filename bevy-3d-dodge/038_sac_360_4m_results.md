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

---

## Experiment 2: No Dodge Bonus

Testing whether disabling close-call rewards helps or hurts learning on 360°.

### Configuration
- Same as above, but with `dodge_bonus_multiplier: 0.0`
- Pure survival reward only

### Results

| Metric | Value |
|--------|-------|
| Training time | 4h 25m |
| Final eval | 344.28 ± 351.65 |
| Best eval | 764.63 |
| Final ep length | 425 ± 323 |
| Peak ep length | 289 (training) |
| Throughput | ~251 fps |

### Comparison: Dodge Bonus vs No Dodge Bonus

| Variant | Best Eval | Final Eval | Ep Length |
|---------|-----------|------------|-----------|
| With dodge bonus | 726 | 416 ± 385 | ~488 |
| **No dodge bonus** | **765** | 344 ± 352 | ~425 |

### Analysis

1. **Slightly better best eval**: 765 vs 726 (+5%) without dodge bonus
2. **Worse final eval**: 344 vs 416 (-17%) suggests less stable learning
3. **Similar variance**: Both have high variance (~350-385)
4. **Shorter episodes**: 425 vs 488 steps on average

### Conclusion

Removing dodge bonus slightly improved peak performance but hurt stability. The difference is marginal - neither reward shaping significantly helps with 360° difficulty. The core challenge is spatial awareness of threats from all directions.

## Overall Conclusion

360° full coverage is substantially harder than 120° fan. Neither reward variant achieved stable learning. Consider curriculum learning or intermediate difficulty steps (90°, 120°) before attempting full coverage.
