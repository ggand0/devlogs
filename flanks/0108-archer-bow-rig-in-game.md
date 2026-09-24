# Archer bow rig in game

Written by Claude Opus 5.5.

2026-09-24. Engine side of Astra's approved archer shot animations: the raise, release and reload from `assets_dev/archer/reload_v11/`, handoff `tmp/handoffs/HANDOFF-archer-approved-ranged-animations-for-claude-2026-09-24.md`. The model's checksum matches the approved file (`cf9c79d9...`). It is installed as `assets/units/archer.glb`, and its tables as `assets/units/archer.shoot.json` (Astra's `archer.raise.json`, renamed to the model's shot file). Nothing is committed until Gota's in-game check.

## Loader (`unit_glb.rs`)

- New parts:
  - 11 and 12: the bow forearm and hand
  - 13 and 17: the string halves
  - 14 and 15: the limbs
  - 16: the held arrow
  - 18 and 19: head and torso
- The archer's 9 and 10 are named `forearm_draw` and `hand_draw`, aliases of the spearman's ids.
- The bow rig loads only when all 14 rig parts are present and the shot file (version 3) is valid:
  - its joints match the `pivot_<name>` empties it names, to within 1 mm
  - the raise, release and reload join end to start, the arrow excepted at the loose
  - the upper string's mesh reaches the rig's string length
- Otherwise the parts fold into the drawing arm, the bow arm or the body, and the archer shoots rigid as before.
- The figure's height is now measured over the trunk parts (0, 2, 3, 18, 19), not part 0 alone. The v11 lower body stops at the waist, and part 0 alone would have scaled the archer 74% too big.
- The last level keeps the head and torso.
- Two authoring constants are not in the JSON and are mirrored from `motion.py`:
  - the string half length, 0.87 m
  - the arrow's 20 mm offset beside the grip

  Astra should add them to the file.

## Signals (both render paths)

- Anim z carries a shot from `RANGED_BASE` 16 as clip * 2 + progress: clip 0 the raise (the 1.5 s draw), 1 the release (0.6 s after the loose), 2 the reload (5.8 s). Then comes a hold of up to 2.5 s in the drawn pose, which covers the longest reload the sim draws (8.4 s).
- A real loose and a cancelled draw both go from wind-up to recovery without a stagger. The loose sets the reload, 192 or more ticks, and a lost target sets `missile::CANCEL_TICKS` (20, formerly an inline literal in `movement.rs`). A cancelled or staggered draw rewinds.
- Bow readiness rides the instance's fourth position channel, which was the scale and was always 1.0 for soldiers. It is the arrows' scale again only for part 7.
  - It rises over 0.4 s while an archer shoots standing.
  - It falls over 0.5 s when he stops, walks faster than 0.6 m/s, dies or starts a melee blow.
  - The pose blends from the carry by it. The quaternions blend straight from identity, the scalars scale, and the arrow shows past half.
  - Corpses write 0.
- The GPU smoothing record grew to 40 bytes. BuildParams now carries the shot timings.
- A legacy archer (v2 or code-built) reads the raise as a stab wind-up and the release as a follow-through, which matches its old look.

## Shader

- `bow_pose` ports `motion.py transforms`:
  - both arm chains, parent times child
  - the bow grip with its own yaw and pitch
  - limbs bending opposite ways
  - the nock from the two fixed-length string halves, and the strings turned onto it
  - the held arrow, plus the reload's free arrow settling onto the string
  - body yaw on everything, torso pitch about the waist on the upper parts, and the head counter-yawing

  Normals turn with each part.
- Tables live in a per-kind storage buffer, group 3 binding 7, in both draw paths.
- While the bow is down, the drawing arm keeps the old rigid march, cheer and melee swings, and the bow arm its march and cheer swings.
- The bow arm no longer lifts during a melee blow, which fixes the bug noted in the handoff.
- The shoulder twist while running now reaches parts 11 to 19. The torso eases in by height like part 0, so the waist seam holds.
- Derived levels: L1 is 768 triangles and L2 is 420.

## Checks

- Clippy clean. `dir` and `arch` fingerprints are 19/19 equal to the pipelined-18ada02 baselines, with the new archer loaded.
- In `FL_TEST_ARCHERY` I recorded the GPU path and the CPU path (`FL_GPU_SYNC=0`): drawn aim with the string to the hand at the jaw, the bow lowered through the reload, the next arrow nocked on the way up. Nothing detached. The v2 archer still shoots with its old motion.
- Not checked yet:
  - melee from the drawn pose
  - walking while reloading
  - death mid-shot
  - performance at scale

## Backups

`refs/backup/archer-bow-rig-2026-09-24`, `tmp/backups/archer-bow-rig-2026-09-24.patch`, `tmp/backups/archer.shoot.json`.
