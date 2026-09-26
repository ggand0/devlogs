# Archer draw and release prototype

Written by GPT-6 Astra.

2026-09-23. Read the archer draw/cheer handoff, the full unit asset contract, the Blender setup notes, the spearman v3 handoff and its engine integration log. Kept this stage to the archer's stationary draw and release, following the rule to review one stage before continuing. No subagents, desktop input, live Blender edits, `src/` edits, installed asset replacement or commits.

## Result

Review: `assets_dev/archer/articulated_v3/review.html`, with a two-view MP4 at 30 fps, speed controls and frame stepping. `archer_draw_review.blend` contains the same 79-frame sequence. `archer.glb` and `archer.draw.json` are the L0 mesh and motion tables. Reproducible sources and project texture inputs are in `tools/blender/archer/`; their default output is `assets_dev/archer/rebuild_v3/`.

Both arms have separate upper arm, forearm and hand meshes. Shoulder covers stay with the body, elbow covers with the upper arms, and wrist covers with the hands. The bow has two hinged limbs and two rigid string segments whose lengths remain fixed. A held arrow follows the nock and disappears at the loose event. The draw lasts 1.5 s and settles at the jaw before loose; follow-through lasts 0.6 s. The draw elbow stays behind the shoulder. The full-draw arrow points along model +Z to within 0.002 degrees.

The two arm chains need three-dimensional joint rotations. Linear Euler interpolation caused an intermediate joint excursion, so the tables use six local XYZW quaternions with normalized interpolation, followed by bow yaw, limb bend and discrete arrow visibility. There are 65 samples per phase, 27 values per sample. This is a different format from the spearman's four-channel pitch table and must be integrated explicitly.

## Measurements

- L0: 2,955 triangles, 1.800 m character height. L1–L3 are not authored in this stage.
- One opaque material, embedded 2048² atlas, VEC4 color, both UV channels and all 14 non-body pivot nodes verified from the GLB bytes.
- All triangles have one rigid part. All new arm, bow, string and held-arrow parts have zero open, nonmanifold or same-winding edges after welding.
- 2,003 poses: maximum rigid edge-length change 7.76e-16 m; joint disagreement 5.31e-16 m; bow-grip disagreement 3.19e-16 m; string attachment/length error 4.62e-16 m; draw-hand/string error 0.1134 mm.
- Minimum measured joint-cover insets: draw shoulder 15.669 mm, elbow 12.863 mm, wrist 8.504 mm; bow shoulder 15.348 mm, elbow 12.914 mm, wrist 8.377 mm.
- Draw-to-follow rotations match exactly at the boundary. Arrow visibility deliberately changes at loose.
- Reloaded scene, all 79 frames: maximum position difference 0.003256 mm. The check permits 0.01 mm for Blender's float32 shape-key time evaluation.
- The promoted rebuild uses the reviewed texture bake and reproduces POSITION, NORMAL, TEXCOORD_0, TEXCOORD_1 and COLOR_0 exactly.

Rendered the entire time sequence from oblique and side cameras, inspected ordered sequences and a rear view with back-face culling. The available inspection tools display still images rather than video playback; Gota should judge timing and motion using the MP4 or Blender scene. Do not describe this as an in-game playback test.

## Limits and next stage

The installed archer remains v2. The renderer cannot read this new part/table contract yet. Reload, melee and cheer are not authored, and no engine or 200k-soldier performance claim is made. The prototype ends in follow-through; it is not a seamless firing loop. The actual quiver is at the right hip, despite the incoming handoff's reference to reaching over the shoulder.

The unchanged body/leg components retain surface exceptions from v2: body has 333 open edges, 4 nonmanifold edges and 3 winding disagreements; each leg has 40 open edges. Some boundaries are hidden garment/component seams, but the full exception audit has not been completed. This is not final asset acceptance. Keep the GLB out of `assets/units/` until the requested stages and acceptance checks are complete.

A return handoff records the part IDs, transforms, integration requirements and remaining work: `work/handoffs/HANDOFF-archer-draw-prototype-for-claude-2026-09-23.md`.

## Review status correction

I incorrectly recorded Gota's "Ok good" as approval in the return message. Gota explicitly said it was not approval and reported impossible right-arm rotations, the hand entering the body during the lift and a motion that does not read as drawing. Corrected the message and handoff to mark the prototype unapproved and unsuitable for integration. The earlier geometry/contact checks did not test anatomical motion.

References: `resources/archer/archer_anim_feedback0.png` and `resources/archer/braveheart_archer_ref.png`. The next requested pass holds a believable drawn-bow pose horizontally and raises it to about 45 degrees, without a separate draw or release.
