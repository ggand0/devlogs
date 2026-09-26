# 0120 — M2TW shuffle animations and engagement evidence

Written by Claude Opus 5.5. Date: 2026-09-25. Follows devlog 0119. No game code changed.

Gota asked for (1) a way to see M2TW's left/right shuffle animations, as reference for Astra, and (2) the exact M2TW logic for attacking and unit engagement. Source: his Steam install, `/data2/SteamLibraryFlatpak/SteamLibrary/steamapps/common/Medieval II Total War`.

## Reading the animations

`data/Animations/pack.idx` + `pack.dat` hold every animation; `skeletons.idx` + `skeletons.dat` hold the compiled skeletons. No unpacking tool is needed. Layout, decoded here (XIDX's source gave the index layout):

- `pack.idx`: 20-byte header (magic, version 9, count 2401), then per entry `i32 entry_size, i32 offset, i32 size, f32 scale, i16 frames, i16 bones, u8 type, name`.
- Payload at `offset` in `pack.dat`: `i16 frames, i16 bones, u8 type`, then `frames x bones` unit quaternions (x, y, z, w, local to the parent), then `frames x bones` local translations (bone 0 carries the pelvis height, so the bob; the rest are constant bone offsets), then `frames x 3` of a small per-frame offset, then `frames x 3` root motion (ground x and z), then 10 floats: duration, distance, displacement x/y/z, speed, two more.
- Skeleton payload: per bone a 76-byte record before its name: offset from the parent (3 floats) and the parent index, plus an inverse bind matrix. After the bones comes the animation state table.
- Axes: x = the soldier's right, y = up, z = forward. 20 fps. Human skeleton of 20 bones (thigh 0.46 m, shin 0.40 m, upper arm 0.30 m).

Scripts: `work/scripts/m2tw_anim/m2anim.py` (readers + forward kinematics) and `render_shuffle.py` (reference sheets and GIFs).

## The shuffles

Reference sheets and GIFs: `resources/animation/m2tw_shuffle_refs/`, for the Mace skeleton (one-handed weapon + shield, the M2TW body closest to our knights and men-at-arms), the Spear skeleton and the Bowman skeleton, left and right, plus the Mace's at-ease `stand_A_step_left/right`. Each sheet: back view (the RTS camera), rear three-quarter view, a top-view strobe with the planted footprints and their time spans, foot lift and pelvis bob, and the sideways travel of each foot.

| clip | length | travel | speed |
|---|---|---|---|
| Mace shuffle left | 1.50 s | 1.20 m | 0.80 m/s |
| Mace shuffle right | 1.50 s | 1.27 m | 0.85 m/s |
| Mace shuffle forward | 1.50 s | 1.35 m | 0.90 m/s |
| Mace walk (one cycle) | 0.90 s | 1.62 m | 1.80 m/s |
| Mace step left/right (at ease) | 1.00 s | 0.64 m | 0.64 m/s |
| Spear shuffle left/right | 1.60 s | 1.20 / 1.24 m | 0.75 / 0.77 m/s |
| Bowman shuffle left/right | 1.50 s | 1.20 / 1.27 m | 0.80 / 0.85 m/s |

What the shuffle does: the ready stance is bladed, left (shield) foot forward and right foot back, torso turned so the shield shoulder leads. One shuffle is two foot moves and the soldier never turns:

- shuffle left: the back (right) foot lifts about 15 cm and crosses behind the front foot to the left, then the front (left) foot glides out about 1.2 m, lifting only 4 to 5 cm;
- shuffle right: the front (left) foot glides across in front first (4 to 5 cm lift), then the back (right) foot steps out high (15 cm).

The at-ease step is a plain side step: the leading foot steps out 0.6 m, the other closes, stance square.

M2TW's slowest locomotion is this 0.75 to 0.9 m/s shuffle. Nothing in the game moves at our 0.1 to 0.4 m/s creep.

## Attack and engagement: what is known, by source

The exact logic is compiled code in `medieval2.exe`; there is no source. What can be read:

### Data files (unpacked with `tools/unpacker` under Proton 8.0 wine, `--filter=*.txt` and `*.xml`)

- `descr_pathfinding.txt`: `formation_hold_distance 20.0` ("formations update 20m after the last point").
- `battle_config.xml`: `melee-hit-rate 1.75` (a global melee balancing factor); `unformed-charge finish-proportion-infantry 0.75`, cavalry 0.4 (the unformed charge task ends once that share of the unit has charged).
- `config_ai_battle.xml`, `melee-manager` (the AI's): `max-engage-dist` 40 m infantry and missile, 120 m cavalry in the open (100 / 200 in settlements); `attack-dist-multiplier 3.0`; retreat analyser distance 40 m; outflank priorities.
- `descr_skeleton.txt` (the header documents the flags):
  - attacks come in success / fail pairs (`eager_attack_centre_mid_c_slashrl_s0_success` / `_fail`), so the outcome is known when the swing starts and the matching animation plays;
  - each success clip carries `-id`, the weapon impact point relative to the root's start, and `-if`, the impact frame. For the one-handed skeleton: punch 1.12 m at 0.9 s, push 0.64 m at 1.15 s, high slash 1.48 m at 0.9 s, mid slash 1.62 m at 0.9 s, combo follow-up 2.01 m at 0.75 s, lunging slash 2.79 m at 1.3 s, charge attack 1.13 m at 0.75 s. The letter in the name (a / c / e) is a distance band, and the heights are hi / mid / overhead / lo;
  - the victim plays a defend clip with its own impact frame, some flagged `-evade` with a probability weight;
  - turns in place are their own clips with angle ranges (`ready_turn_cw_15` for 5 to 30°, `_45` for 29 to 68°, `_90` for 67 to 115°);
  - `locomotion_table soldier` is named but defined in the exe.

### The exe's class names (RTTI strings)

- Soldier controller stages: moving, moving to, moving through, shuffle, shuffling forwards / backwards / left / right, turning initial / turning / turning final, stopping, stopping before move, **stopping and waiting before move**, finished and idle, finished and moving. So a soldier reaching a nearby spot either shuffles without turning or turns, moves and turns back, and waiting before a move is a real state.
- Soldier actions: `ACT_ATTACK_MELEE`, `ACT_ATTACK_SPEAR`, `ACT_READY`, `ACT_MAN_MOVE_INDIVIDUAL`, `ACT_MAN_CHARGE`, `ACT_MAN_CHARGE_ATTACK`, `ACT_PURSUE`, `ACT_KNOCKDOWN`, `ACT_WITHDRAW`, among others.
- Unit tasks: `MELEE_ATTACK`, `MELEE_ATTACK_FORMED`, `MELEE_ATTACK_PHALANX`, `ATTACK_ENGAGE`, `INFIGHT`, `RESHUFFLE`, `CHARGE_FORMED`, `CHARGE_UNFORMED`, `PURSUE_FORMED`, `REFORM`, `READY`, `INTERCEPT_UNIT`, `SCHILTROM`, `WITHDRAW`, `FLEE`, among others.
- Engagement queries `IS_UNIT_ENGAGED`, `IS_UNIT_ENGAGED_WITH_UNIT`; melee sectors `BUC_MELEE_FRONT / FLANK / REAR`; `m_is_in_formation`, `m_melee_state`.

### Engine structs (M2TWEOP, devlog 0036)

Every soldier keeps his formation slot and his displacement from it all battle; one target per soldier and a count of attackers per victim; the melee action carries `crowded`, `isSideStepping`, `allowAttack`, `readyStance`, a paired `hintSoldier` and a `blockedCounter`; units keep per-enemy engagement records (`engagedSoldiers`, `spearPoints`, `engagedRatio`).

### Not known

Target choice among enemies in range, the distance at which a soldier leaves his slot to attack, when he shuffles rather than walks, how `crowded` is decided, how long he waits, and the hit roll behind the success / fail pick. These live in the exe. Paths to them: disassembly of `medieval2.exe` starting from the RTTI vtables above (`SOLDIER_CONTROLLER_NORMAL`, `ACT_ATTACK_MELEE`); live memory probes through EOP as in devlog 0057; or measuring filmed M2TW fights.
