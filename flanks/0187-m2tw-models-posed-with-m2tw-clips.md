# M2TW's own models posed with M2TW's clips

Written by Claude Opus 5.5.

Gota asked for real M2TW soldiers in the mined poses, not stick figures: the stick figures draw no weapon, so Astra's first versions put the sword at angles M2TW never uses. This entry decodes M2TW's unit model format, poses the English Dismounted Feudal Knights with the standing and ready clips, and renders them beside our standing v2 with Astra's review cameras. Output: `refs/animation/m2tw_model_refs/` (local only, its README). Scripts: `assets_dev/_m2tw_refs/` (private assets repo, scripts only). M2TW's data they read: `/data/ggando/m2tw/`. Nothing in `src/` changed.

## Getting the files

`data/unit_models/battle_models.modeldb` (unpacker, `--filter=*.modeldb`) names each unit's meshes, textures and skeletons. Dismounted Feudal Knights: `unit_models/_Units/EN_Lmail_Hmail/dismounted_feudal_knights_lod0..3.mesh`, texture `EN_Lmail_Hmail_england.texture`, shields and weapons from `attachmentsets/final heater_england_diff.texture`, skeleton `MTW2_Swordsman`, weapon skeletons `MTW2_Sword_Primary` and `fs_test_shield`. The unpacker takes a filename glob (`--filter=*dismounted_feudal_knights_lod*`), about 10 s per call. A `.texture` is a 48-byte header and a plain DDS that Pillow reads.

## The .mesh format (`assets_dev/_m2tw_refs/m2mesh.py`)

A boost binary archive. First the triangle groups, each a u32-length category, a u32-length variant name, a u32 triangle count and u16 indices (the first group has two extra zero bytes of class header before its count). The knight file holds variants per category for the game to mix per soldier: Arms (5), Legs (4), Hands, Body (3), Head (3), `teeth`, `primaryactive0` and `primaryactive1` (3 swords each, the blade in hand plus the scabbard), `shield0` (10 heater patterns).

Then one shared vertex buffer as parallel arrays, each a u32 count and the data: UV (2 f32), skin weights (2 f32), position (3 f32), bone indices (4 u8), normal, tangent and binormal (4 u8 each). Then the 26 bone names (the 20 body bones of the skeleton in its order, then weapon and shield bones).

Three things that were not obvious:

- Bone indices: byte 2 is the bone carrying weight 0, byte 1 the bone carrying weight 1. Checked by distance from each vertex to its bones (2469 of 2577 single-bone vertices sit nearer byte 2's bone).
- Shield vertices have both weights zero and both indices 15 (`bone_Lelbow`): bound wholly to the forearm.
- UVs address the unit texture and the attachment texture side by side: u 0 to 0.5 is the unit texture, u 0.5 to 1 the attachment texture, v as is. Read as one texture, the knight wore the wrong parts of the sheet (a face on the side of his head).

Positions are in the bind pose: the skeleton's T pose with the pelvis at the origin, M2TW's left-handed axes (x his right, y up, z forward). The skeleton file stores an inverse bind matrix per bone; every rotation in it is identity, so skinning is `pos + R (v - bind)` per bone, blended by the two weights. The clips' per-bone translations equal the skeleton offsets.

## The weapon

The clips animate the 20 body bones only. The blade is bound to the weapon bone, which `MTW2_Sword_Primary` puts 0.085 m out from the right hand with no rotation, so the blade's angle is the hand's turn in the clip. The stick figures could never show this: a hand's turn moves no joint the figure draws. In M2TW's standing frame the hand points the blade forward 0.81, across to his left 0.49 and up 0.32 (unit vector), the tip 0.82 m ahead of his centre. In the ready frame the blade is held level at the waist, pointing forward, its tip at about the shield's front edge.

The rotation math was checked on the posed model: arms hang down, the head faces forward (head +z maps to (0.05, 0, 1.0)), knees bend forward. A conjugated rotation would swing the arms up.

## Numbers (posed, ground where the clip puts it; the first pass lifted the lowest sole to the ground, 2.4 cm higher)

| Pose | Height | Width | Ahead of centre | Behind |
|---|---:|---:|---:|---:|
| M2TW stand, frame 0 | 1.838 m | 0.851 m | 0.823 m | 0.389 m |
| M2TW ready, frame 0 | 1.634 m | 1.014 m | 0.650 m | 0.387 m |
| Ours, standing v2 | 1.800 m | 0.815 m | 0.573 m | 0.156 m |

M2TW's standing knight is about as wide as ours and reaches 25 cm further forward with the sword. Its ready stance is 1.01 m wide with the arms out, wider than handoff 085's 0.8 m target for our ready pose.

## Scripts

All in `assets_dev/_m2tw_refs/`, committed in the private assets repo, scripts only; its README lists each and the commands. The first pass ran from `work/scripts/m2tw_anim/`, where copies of the old stick-figure scripts and the first-pass ones remain. They read M2TW's data from `M2TW_DATA` (default `/data/ggando/m2tw/data`): `Animations/` copied from the install, `unit_models/` unpacked from its packs, 67 MB in all.

## Second pass, the same night: every basic clip, three models

Gota asked for a view from his left side (soldiers are not symmetric), the scripts committed in assets_dev, M2TW's files copied to /data, and the same sheet for every clip in `refs/animation/m2tw_basic_refs/`.

- Models per family (`m2units.UNITS`): the knight for sword and shield and the attacks; Armored Sergeants (England, `MTW2_Spear`, a long spear in the right hand and a kite shield) for the spear clips; Peasant Archers (England, `MTW2_Fast_Bowman`, the bow on the left hand, a quiver on the back) for the bow clips. The archer's knife is left out: part of it is bound to the same weapon bone as the bow, so here it would ride the bow hand.
- `Rig` maps the model's bones to the skeleton by name, weapon bones to the hand their weapon skeleton names (`MTW2_Sword_Primary` and `MTW2_Spear_primary`: right hand; `MTW2_Bowman_Primary`: left hand), shield bones to the left hand. A weapon bone's offset cancels in the skinning, so a weapon rides its hand rigidly. Skinning is vectorised (a frame in 40 ms) and matches the first pass to 6e-7 m.
- `render_m2_clips.py` builds the model once in Blender and moves its vertices per frame; five views every frame, seven on eight sampled frames, 256 px, EEVEE at 16 samples, about 5 s a clip. It waits before each clip while a flanks game runs. 78 clips in about 7 minutes; sheets and GIFs (`m2_clip_sheets.py`) in 3 more; 338 MB of output.
- Views are now front, his left side, his right side, back, above, three-quarter and the game camera.

Checked by eye: the knight's ready and shuffle sheets, the spearman's stand (spear upright in the right hand) and ready (spear levelled, inside the 3.6 m frame), the archer's stand (bow hanging in the left hand), ready and walk.

The noise Gota heard was not the install: it sits on `/data2`, a WD SSD. The spinning disks are `/data_hdd2` and sde. The likely source was this session's own background `find /` hunting for Proton's wine, which walked `/data_hdd2`.

## Third pass, 2026-10-04: lighter units, one folder per unit

Gota found the knight too armoured to read the legs from and named units with legs in sight from playing M2TW again: Spain's two early sword units, peasants, hand gunners in melee, Spain's conquistadors. Asked to confirm each one's moveset, render them, and put the obsolete references under one folder.

Movesets, from `battle_models.modeldb` (each model's skeletons) and `descr_skeleton.txt` (each type's clips and parent):

- Swordsmen Militia and Sword and Buckler Men (Spain's two early sword units) and Dismounted Conquistadores use `MTW2_Swordsman`, the knight's skeleton: the same moveset. Mounted conquistadores are cavalry (`MTW2_HR_Lance`, `MTW2_HR_Sword`).
- Peasants use `MTW2_Spear` with `MTW2_Spear_primary` and no shield: the spear moveset of the Armored Sergeants, a pitchfork in the right hand.
- Hand gunners in melee use `MTW2_Non_Shield`, whose parent is `MTW2_Swordsman`. It keeps the knight's stand, steps, shuffles, turns, transitions and attacks, and swaps in the Knifeman's ready stance, walk, run, charge and parries. Of the 26 basic clips, 4 differ from the knight's: ready, walk, run, charge. Pike militia carry the same skeleton for their sword.

New scripts: `m2modeldb.py` parses the model list (its strings are length-prefixed, so paths with spaces parse) for each model's meshes, textures per faction, attachment textures and skeletons; `m2clips.py` resolves any skeleton type's basic clips through its parent chain and reproduces the old hand-written indexes clip for clip, apart from one fix: Peasant Archers (`MTW2_Fast_Bowman`) run with `MTW2_Fast_Bowman_run`, where the first pass showed `MTW2_Bowman_run`. `m2units.UNITS` is keyed by model name; each unit names its faction, skeleton, the hand each weapon group rides (a hand gunner's sword is a `secondaryactive` group), its view width, whether it plays the attacks, and one variant per body part. The hand gunner's gun is left out in melee.

The unit table now has eight soldiers, all re-rendered for one consistent set: the four `MTW2_Swordsman` units and the hand gunner play 26 basic clips and three attacks, the two spear units 26, the archer 23. Output: `refs/animation/m2tw_model_refs/<unit>/`. Data grew to 105 MB with the Spanish textures and the five new models.

The obsolete references moved, unchanged, to `refs/animation/obsolete/`: the three stick-figure folders (`m2tw_basic_refs`, `m2tw_pose_refs`, `m2tw_shuffle_refs`) and the first model pass filed by clip family. The knight comparison moved to `m2tw_model_refs/compare_knight_standing_v2/`. `refs/animation/README.md` maps every old path to its new one, since older devlogs and handoffs name the old paths.

## Next, when asked

- The other attacks, defend and parry, hit reactions and deaths (their clips are in `descr_skeleton.txt`; the index format of `render_clips.py` takes any list).
- An armature with the skin weights and the clip as an action in a .blend, so Astra can scrub a clip and read joint angles directly.
