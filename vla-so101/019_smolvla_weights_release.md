# 019: SmolVLA v3 Dual Weights Released to HuggingFace

Date: 2026-08-14

## Trigger

Received an email from **Abu Shahid**, who found the blog post on fine-tuning SmolVLA for SO-101
pick-and-place. He is evaluating SmolVLA for pick-and-place in a **MuJoCo simulation** environment and
asked whether the fine-tuned checkpoint could be shared — noting that the dataset and training recipe
were published but the weights were not, and that having them would save re-running training and help
validate fine-tuning as a path for his sim setup.

He also mentioned finding a normalization pipeline bug in the post-migration SmolVLA base model and
filing a GitHub issue on `huggingface/lerobot`. Not verified on our side; possibly worth following up
since our config uses `VISUAL: IDENTITY` / `STATE: MEAN_STD` / `ACTION: MEAN_STD`.

His read was correct: all four training scripts in this repo run with `--policy.push_to_hub=false`, so
nothing was ever uploaded. Only the dataset (`gtgando/so101_pick_place_10cm_v3`) was public.

## What was released

**Repo**: https://huggingface.co/gtgando/smolvla-so101-pick-place-10cm-v3-dual (public)

Source: `outputs/train/smolvla_so101_10cm_v3_dual/checkpoints/020000/pretrained_model/`
(identical to `checkpoints/last`) — the run documented in [011](011_smolvla_v3_dual_training.md), the
same checkpoint the blog post's 60-80% success rate refers to.

### Why this checkpoint and not another

Confirmed against the head-to-head in [012](012_act_and_wrist_only_training.md) — all three v3 models at
20k, 5 real-robot rollouts each:

| Model | Cameras | Success |
|---|---|---|
| SmolVLA dual | wrist + overhead | **100% (5/5)** |
| SmolVLA wrist-only | wrist | 80% (4/5) |
| ACT dual | wrist + overhead | 80% (4/5) |

Dual-cam is the best model by eval, by final loss (0.005 vs 0.006), and is the `scripts/infer.py` default.
The 100% there is one lucky 5-episode run; 011's "typical 60-80%" is the honest number and is what both the
blog and the model card report.

| File | Size |
|---|---|
| `model.safetensors` | 906.71 MB |
| `config.json` | 2.1 KB |
| `train_config.json` | 5.5 KB |
| `README.md` (model card) | written for this release |

`training_state/` (optimizer + RNG + scheduler, 413 MB) was deliberately **not** uploaded — it is only
needed to resume training, not for inference.

### Integrity check

SHA256 of local vs. remote `model.safetensors` verified matching:

```
34e1d3d54239df5a4790b7c561e1fc45c04b6a1d7e9b3ca445be5b75057839fe
```

## Pre-upload review

Checked `config.json` and `train_config.json` before publishing:

- No API tokens or credentials
- No absolute local filesystem paths (`output_dir` is relative)
- `wandb.entity` is null, wandb disabled
- Only external references are the public dataset repo ID and `HuggingFaceTB/SmolVLM2-500M-Video-Instruct`

Clean, uploaded as-is with no redaction.

## Model card contents

Documented base model, dataset, full training hyperparameters, observation/action spec
(both camera keys required, `(3,480,640)` each, 6-D state and action), the 60-80% success rate with its
measurement protocol, and a usage snippet.

Limitations section states explicitly:

- **Real-robot only** — zero-shot sim transfer is untested, no domain randomization. Called out
  directly because Abu's stated use case is MuJoCo, and it would be misleading to let him assume
  the reported success rate carries over.
- Camera placement baked in; fixed ~10cm workspace
- **Color-specific visual features** — the 012 color test (1/3 on orange/blue/green vs 5/5 on red; grasp
  generalizes, place does not) is the strongest concrete evidence that appearance shift breaks this policy.
  Directly load-bearing for the sim question, so it is quoted in the card.
- Action-chunk boundary jitter (visible shaking), no temporal ensembling
- Undertrained — loss still dropping at 20k, 75 episodes is small

License: Apache 2.0, inherited from `lerobot/smolvla_base`. Repo itself is MIT.

## Not released

- **Wrist-only variant** (`smolvla_so101_10cm_v3_wrist`) — mentioned in the model card as available on
  request rather than uploaded. It *does* have a documented number, 80% (4/5) in 012; an earlier draft of
  the card wrongly claimed it had none, corrected before anyone fetched it. Withheld simply because it is
  the weaker model and nobody asked for it, though its single-camera requirement may make it the easier
  one to port to a new setup.
- Note there are two wrist-only v3 runs: `_v3_wrist` (012, the one evaluated at 80%) and
  `_v3_wristonly_tb2` (013, retrained on a physically filtered wrist-only dataset for a 44% speedup,
  identical final loss 0.006, never separately evaluated). If the wrist-only model is ever released,
  `_v3_wrist` is the one with the number attached.
- **ACT baseline** (`act_so101_10cm_v3_dual`, 12 GB) — not asked for.
- Earlier runs (`smolvla_so101_10cm`, `_v2_wrist`, `_pick_place`) — superseded by v3.

## Follow-ups

- [ ] Reply to Abu Shahid with the model URL; flag the sim-transfer caveat and ask for his lerobot
      normalization issue link.
- [ ] Rotate the HuggingFace token stored in `~/.cache/huggingface/` (profile `[so101]`) — it was
      printed in plaintext during this session while checking the logged-in account.
- [ ] Consider adding the model link to the `vla-so101` README Results section.
- [ ] If the normalization bug is real, re-check whether it affects this checkpoint's inference path.
