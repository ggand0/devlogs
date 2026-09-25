# Knight side shuffle, first motion review

Written by GPT-6 Astra.

2026-09-25. Built a knight-only side-shuffle candidate from `tmp/handoffs/HANDOFF-side-shuffle-2026-09-25.md`. The stage ends at motion review; Gota has not accepted this candidate yet. No `src/` or renderer edits, asset replacement, attack-table changes, commits or pushes.

The existing knight has square symmetric legs and hip pivots at (+/-0.115, 0.910, 0) in glTF metres. The M2TW mace reference instead crosses feet from a bladed fighting stance. The new motion takes the leading foot outward, plants it, then brings the trailing foot across to recover the original stance. The direction is mirrored in the leg/pelvis curves while the equipment stays on its original side. Facing never changes.

The square stance makes the reference's 1.2 m single step-and-close too wide. This candidate travels 0.60 m in 0.75 s, keeping the reference's nominal 0.8 asset m/s. Two cycles cover 1.20 m in 1.50 s with four foot moves. Leading/trailing foot lifts are 50/75 mm; maximum pelvis dip is 73 mm. Small lateral pelvis shift and under one degree of torso counter-roll keep the upper body restrained. The first review reads as a brisk upright sidestep, less crouched than M2TW's ready shuffle.

The solver derives knee/ankle pivots from the shader's existing 50%/90% hip-height convention, then solves hip roll, hip pitch and knee flex against world-planted ankle targets. Ankle pitch and roll keep soles horizontal. A 129-sample table for each direction stores these joint angles, pelvis offset and torso roll. Both endpoints are exactly the shipped standing pose. The preview deforms the shipped mesh with the serialized table and the shader's height-based bend bands, rather than using a separate idealized rig.

Numerical checks cover 1,001 interpolated poses per direction. Maximum planted-sole world error is 0.0310 mm, ankle-target error 0.1523 mm, standing endpoint error zero and mirror error zero. Maximum hip roll is 25.12 degrees; knee flex 57.25 degrees. All ankle targets are reachable. Ground checks across L0-L2 find only about 0.030 mm numerical penetration. Blender playback agrees with the table-derived vertices within 0.000217 mm across 48 frames per direction. These checks do not establish complete anatomical or self-collision correctness.

The unchanged source GLB passes byte inspection for VEC4 vertex colors, part UVs and pivots. L0/L1/L2/L3 remain 2768/678/250/52 triangles and 1.800 m high. L0 has 440 vertices per leg; all vertices have their original part IDs. No GLB export was needed.

Rendered two complete cycles at 24 fps from back and rear-oblique fixed cameras, including root translation and standing holds. Inspected both directions' full first-cycle frame sequences in order plus back-view poses. Delivered continuous MP4/GIF playback and reference-style sheets with top-view planted footprints, foot lifts, pelvis dip and world sideways travel. No in-game review was performed. Snap Blender required running outside the sandbox; ffmpeg also required escalation after the syscall filter stopped encoding. Both local rendering/encoding runs completed.

Files are in `assets_dev/knight/shuffle_v1/`. Entry point for review: `knight_shuffle_review.mp4`; pose table: `knight.shuffle.json`; playable scenes: `knight_shuffle_left.blend` and `knight_shuffle_right.blend`. Reproducible scripts are promoted to `tools/blender/knight/{shuffle,build_shuffle,review_shuffle,compose_shuffle}.py`. Python compilation, promoted-script table regeneration, numerical validation and `git diff --check` pass.

Integration details and checksums are in `tmp/notes/knight-side-shuffle-for-claude-2026-09-25.md`. Two details matter: cycle distance must be scaled with the model (0.366667 world units for a 1.8 m mesh shown at 1.1 units tall), and an arbitrary mid-cycle stop needs step completion or a settling transition. Fading the full motion table to standing while the sim continues to move cannot preserve foot contact. L3 should keep its all-body silhouette static relative to the moving root, avoiding pelvis dip on unsplit feet.

User-visible result is a review candidate only. Other kinds, runtime wiring, arbitrary-distance transitions and art acceptance remain separate stages.
