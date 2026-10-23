# Archer model, the knife, and the knight's clips on the archer

Written by Claude Opus 5.5. Branch `feat/archer-rework`, 2026-10-22. Integrates handoff 141 (the approved model of handoff 140, the knight's clips on the archer, the knife-out state of plan 026 §5).

## What changed

### The model

- `assets/units/archer.glb` is the approved `quiver_v3` build (SHA-256 `e65c90af…`) with one change: the knife's 150 L0 vertices (triangle rows 2654 to 2703, 50 triangles, none sharing a vertex with the hand) carry part 20 instead of the hand's part 10. Every other byte is the approved file. New tool `tools/blender/archer/knife/tag_knife.py` does it, refuses any other source checksum, and checks the rows are hand triangles with no shared vertex before writing. Output SHA-256 `c63e0e88…`.
- The ranged clip files stay beside it. `archer.melee_stand.json` and `archer.melee_ready.json` come with Astra's commit 8aac4a9 (cherry-picked as 03b20e0).
- Only L0 is new; L1 to L3 are the old meshes (handoff 140). L3's bow is in the body part and cannot be hidden. Follow-up before the asset checklist passes.

### The loader (`src/unit_glb.rs`)

- Clips map tracks by joint name: each model joint takes the clip's track of its name, a joint the clip does not name keeps its rest pose, a track for a joint the model lacks is dropped. The clip must name the model's root. The log lists the joints held at rest once (`arrow`, `limb_u`, `limb_l`, `string_u`, `string_l` for the archer).
- `CLIP_KIND[archer]` is the knight: his own file first, else `knight.<clip>.json` beside the model. So he plays the knight's shuffles, cut, stab, charge, charge blow and parry.
- A kind without a shield loads no block clip (the archer). A bow-rig kind loads the cut clip although his sim has no slow cut: he has no weapon arm to draw the quick overhead blow, so the cut clip plays it at the quick blow's pace.
- `SKEL_CLIPS` grows to 18: `melee_stand` (16) and `melee_ready` (17). `SKEL_CLIP_SLOTS` 18 and the WGSL `Skel` arrays with it.
- `PART_NAMES` gains `knife` (20). A model whose bow rig fails folds it into the draw arm with the forearm and hand.

Checked before relying on the mapping: the archer and the knight share the rest frame (arms down, hands forward); in the standing clips the legs, pelvis, chest and head match to 0.0°, and the arms differ only because each holds his own weapon.

### The knife-out state (sim)

- `sight` bits 10 to 14 are a countdown and bit 15 is `SIGHT_KNIFE` (`sim/soldier.rs`). Per archer per tick, in `scan` after the reach marks: an enemy in his reach, an enemy block's edge within `KNIFE_EDGE` (10 m) of him, or an empty quiver with an enemy block in his regiment's bow range sets the count to 31; otherwise it drops by one every fourth tick (about 4 s to put the knife away). The knife is out while the count is above zero or his regiment has its swords out.
- The blocks a man measures come from prep (`sim/job.rs` `sidearm_reasons`, `Sidearm`): per archer regiment, the unbroken enemy blocks whose edge gap to its own block is within 10 m, as rectangles (`formation::BlockRect`, which `block_reach_along` now uses, same arithmetic). No slack constant: the gap along the centre line never exceeds the distance from any point of one block to any point of the other, so every man inside his block is covered; a man outside it is fighting (his reach covers him) or dressing back. No grid query.
- With the knife out he starts no draw, and a draw in progress lets go (the target-gone branch, `CANCEL_TICKS`).

### The renderer

- GPU record: bit 7 of the kind byte carries `SIGHT_KNIFE` (`render_units_gpu.rs` `pack_chunk`, also from the job's columns). CPU path reads the column.
- The fourth position float carries the knife below zero: `-1 - steps` (unit_build.wgsl, with `params.steps.z` naming the bow-rig kinds), so a knife-out archer steps in the knight's shuffles; with the bow out it stays his bow.
- `unit_pose.wgsl`: `pose_begin` decodes the knife (bow weight 0 while it is out); `put_skeleton` uses `melee_stand`/`melee_ready` in place of stand/ready; the archer's rigid arm swing no longer runs over a blow a clip plays. New pose slot `P_KIT` (45; bow stride 46) holds the knife for the pulled draw. `place` gathers the put-away kit at the soldier's position (zero-area triangles): with the knife out parts 8, 13 to 17 (bow grip, string halves, limbs, held arrow); with the bow out part 20. The knife is posed as the hand (part 10) on the part path, so shadows and far levels follow.

## What the scenes showed (opt-dev)

| Scene | Result |
|---|---|
| `FL_ARCHERY_SWORDS=1`, 60 a regiment, close-up | knife in the right hand, no bow, quiver on the back; the regiments march, charge and fight in the low knife crouch |
| plain, 60 a regiment, close-up | bows up and shooting, no knife |
| `FL_ARCHERY_ATTACK=1 FL_ARCHERY_NOSKIRMISH=1`, 500 a regiment | bows on the attacker until it closes; at contact the men within 10 m have knives out, those further along keep their bows. Blue struck by its own shafts: 7 (devlog 0264's run of this scene: 62); orange struck 2820 by t=56 |

Screens in `tmp/runs/archer-model/shots/`, logs in `tmp/runs/archer-model/`.

## Checks

- opt-dev build, strict clippy (`--all-targets -D warnings`) and the 50 tests pass.
- `FL_TEST_DIR` fingerprint, 45 s, branch against main's binary (`tmp/runs/archer-rework/main-src`): identical, 21 fingerprints, last `1260 3222 99dc6fe27e005c91`.

## Commits

Astra's source commits cherry-picked by Gota (a36be4c, 03b20e0, 3da75a3), then c905987 (the model and `tag_knife.py`) and 93cdd11 (the code). The shipped model loads with all clips without `FL_GLB_ARCHER`.

## Open

- Far levels L1 to L3 (handoff 140 §4).
- The back-quiver reload clip (Astra's next).
- The 200k flank-view measurement against main, at the end.
- Gota plays.

## Astra's distance levels (handoff 143, 2026-10-23)

Candidate `assets_dev/archer/rework_lods_v1/archer.glb` (SHA-256 `f344e311…`, the same bytes as `../flanks-gfx/assets/units/archer.glb`, still uncommitted there when checked). The knife is part 20 on all four levels and the bow limbs are their own parts down to L3, so the existing hide rule covers them with no code change. Checked in game at forced L1, L2 and L3 (`FL_LOD_PX`), knife and bow each: no shader or asset errors; bows show with the bow out, knife shows and the bow hides with it out at every level. L1 matches L0 pose for pose (the marching knife-out archers hold their arms forward at both levels: the knight's charge pose over the gait). Shots in `tmp/runs/archer-model/lods/`. Integration waits for Astra's commit of the GLB and `tools/blender/archer/lods/`; once it is in, `tag_knife.py` has no use (the builder writes part 20).

## History rebuilt (2026-10-23)

Astra's distance-model commit f34fb36 moved here from `feat/archer-hood-source`; `../flanks-gfx` switched to `main` (8810113, origin/main). Then, at Gota's order, the text audit: every commit message read in full and rewritten (process words such as "Ship" and "approved", internal terms, test counts), and every added line grepped. Fixes folded into the commits that introduced them: plan references and a non-ASCII sign in sim comments, an em dash in ai.rs, en dashes and "dagger" in the tool READMEs, "approved" in the distance-model build, "kit" in the shader names (`P_KNIFE`, `put_away`). `tag_knife.py` stays: the distance build needs its output as the source. The intermediate tagged model is gone from the history; the last commit carries the final model.

New branch, 9 commits on main 8810113: 7242126, 6ac7ef8, 4821693, bc934e0, 7f1302f, 29bdcae, bb5715c (`tag_knife.py`), 392694e (code), 74d0854 (model and distance meshes). Backups: `backup/archer-rework-2026-10-23` (old tip 93cdd11), `backup/archer-hood-source-2026-10-23` (f34fb36), bundle `work/backups/archer-rework-and-hood-source-2026-10-23.bundle` (verified). Clippy, build and the 50 tests pass; both render paths draw the knife scene with no errors.

## Gota's play, 2026-10-23: three issues

Gota played the branch at 74d0854 (the melee animations look good) and reported, in Gota's words:

> melee animations look good but i found bugs
> A: it does melee charge pose when doing bow attack move
> B: should remove spear wall ui icon for archers
> C: this bug
>
> 1. i order attack on distant enemy: refs/archer/attack_move_debug0.png
> 2. archer unit gets blocked by some unit: attack_move_debug1.png
> 3. some overflown soldiers go straight into the enemy line because of unit center is still at "safe" distance or whatever (no skirmish mode): attack_move_debug2.png
> 4. at some point it starts attacking with bow: attack_move_debug3.png
> 5. then soldiers suddenly back off
> 6. some guys still end up fighting in melee: attach_move_debug4.png
> Something similar also happens when there's no block units, though the unit starts backing off earlier. still awkward behavior

### A: the charge pose with the bow out (fixed, 05b627b)

The knight's charge clip, now on the archer, held his upper body in the charge pose whenever his regiment was confident and he ran, bow out or not. The pose holds now only for an archer with his knife out.

### B: the wall button for archers (fixed, abb2ff6)

`refresh_control_buttons` starts `all_wall` and `all_spear` true and clears them only for non-archer regiments, so a selection of archers alone showed the wall button lit with the spear wall icon. The button now greys out when nothing picked can form a wall, as the archer buttons do for a selection without archers.

### C: an attack-ordered archer regiment streams into the enemy line, then backs off

The pictures: (0) the order on a distant block; (1) a friendly block stands in the way and the archers stream around it into a long column; (2) the head of the column walks on toward the enemy while the regiment's block is still short of range; (3) the regiment starts shooting; then the men walk back; (4) a few of the head are caught in melee.

The cause, read from the code:

- An archer attack order's march goal is the target block's centre (`Groups::goal`, `Order::Attack` gives `t.centroid`; `resolve_orders` passes it to the kernel). Each man walks to that goal plus his own offset in the formation, so every man's destination lies inside or beside the enemy block. Nothing in the march holds a man back at shooting distance; only the regiment's stand-off stops the march.
- The stand-off test (`archer_fire_solutions`, `edge_gap`) measures the regiment as a rectangle of its files and ranks around its centroid. A column strung out around a friendly block is not that rectangle: its centroid lags far behind its head, so the test still reads "out of range" while the head is already at the enemy.
- When the stand-off begins, the regiment's frame snaps to its centroid (`gd.anchor = gd.centroid`), and every man walks to his place around it: the head of the column, far ahead of the centroid, walks back. That is step 5.
- Without a block in the way, the regiment stays in formation but its men still head for places at the enemy until the stand-off; the early back-off there is likely the same snap, not measured.

A first proposal (march to the point where the block gap is 96 m) was rejected by Gota: a fixed number tied to today's one bow range, and a point taken from the block rectangle, the same view of the regiment that hid the column's head.
