# Spearman v3 in game: jointed arm and attack tables

Written by Claude Opus 5.5.

2026-09-23. Engine side of Astra's v3 spearman (devlog 0093, handoff
`tmp/handoffs/HANDOFF-spearman-v3-for-claude-2026-09-23.md`). Gota checked the
stab in game: it works.

## Contract

- Parts: 4 upper arm, 9 `forearm_spear`, 10 `hand_spear`, 8 weapon. Each is
  rigid and rotates about its own `pivot_*` empty (shoulder, elbow, wrist, grip
  contact).
- The attack tables ship next to the model as `<model>.stab.json`, in the format
  `motion.py export_clip` writes (`spear_stab.json`, renamed). There are 33
  wind-up and 33 follow-through poses, each `[shoulder, elbow, wrist, level]`.
- The loader uses the jointed path only when all four parts, their pivots and a
  valid table file are present. The tables must start and end in the guard, and
  their joints must match the empties to within 1 mm.
- Fallbacks: without the tables, the forearm and hand join the arm, which then
  bends at the elbow as v2 did. Without `joint_elbow` or `pivot_weapon`, the
  weapon joins the arm and turns with it as one piece. Each fallback logs why.

## Shader

- Pose = table sample x a blend weight. The rest mesh is the carry, so blending
  from carry is a straight scale of the joint turns.
- The weapon keeps its own pitch, `-pi/2 * level`, and slides 0.715 m
  (authoring) through the fist as it levels.
- Chain: elbow' = S + R(sh)(E - S), wrist' = elbow' + R(sh+el)(W - E),
  grip' = wrist' + R(sh+el+wr)(G - W).
- The whole arm swings a little about the shoulder on the march and in the
  cheer.
- The wind-up reaches the guard in its first third when readiness is lower.
  The follow-through settles to readiness over its last 8%.

## Signals (both render paths)

- The stance band moved to anim2.x, so readiness survives an attack. Leg length
  moved from anim2.x into the per-kind `Rig` uniform, and corpses are told
  apart by fx = 2. The `gait::Legs` resource is gone.
- anim z holds only the attack and the cheer:
  - attack = digit * 2 + progress, where digit is the style, plus 3 on a charge
  - the cheer starts at 12
- The wind-up is now linear and reaches 1 on the tick the blow lands:
  `(w - swing_t + alpha) / (w + 1)`. The old quadratic lunge and the charge
  amplitude are rebuilt in the shader, so the code-posed arms keep their look.
- Follow-through lasts 0.6 s after a blow. A blow is detected when a wind-up
  turns into recovery without a stagger. The follow-through keeps playing if
  the soldier is staggered or killed.
- Rewind: a wind-up cut short by a stagger or death plays backwards to 0 over
  0.25 s.
- While a soldier's melee attack shows, his band rises to at least 0.65 (the
  confident-fighting tier), so the arm settles into the guard, not the carry.
- The GPU smoothing record grew to 32 bytes. BuildParams now carries
  FOLLOW_S, FOLLOW_BASE, FOLLOW_SPAN, REWIND_S and BAND_FIGHTING.

## Incident

This work began without a request, after a context compaction. The first run
drew no units at all: `from` is a reserved WGSL word, and cargo does not check
shaders. Astra stashed the tree. It was restored with `git stash pop`.

- Backups: `refs/backup/v3-integration-2026-09-23` and
  `tmp/backups/v3-integration-2026-09-23.patch`.
- Lesson: check every run's log for `failed to process shader`.

## Gates

- Fingerprints: `dir` 19/19 and `arch` 19/19 equal to the pipelined-18ada02
  baselines.
- `FL_GPU_CHECK`: only the usual off-by-one LOD counts.
- Clippy clean.

## Next

- Man-at-arms v2 integration.
- Sword kinds stay on the old bent arm until Astra authors their tables.
- Shieldwall shoulder motion is still separate work.
