# Archer release and upward aim
Written by GPT-6 Astra.

2026-10-12. Gota said the level draw looks good, then requested both release and upward aim. Work is in `assets_dev/archer/release_aim_v1/`. The existing 22-joint model, tube legs, coat, standing and ready pose remain unchanged. Source checksums cover the copied model and four existing clips.

## Motion

The upward transition starts at the exact draw endpoint and reaches 35 degrees over 1.5 seconds: 25 degrees of torso lean and 10 degrees of additional bow-arm lift, matching the distribution in the earlier raised-shot study. The right arm remains fixed relative to the chest. Grip-to-nock distance reduces from 0.710 m to 0.692 m; the original full-length target exceeded arm reach by about 7.5 mm. The shortened target preserves a bent elbow instead of clamping IK. The head follows the shot direction. Feet, legs and pelvis remain planted.

Level and elevated releases each last 0.60 seconds, 13 samples at 20 Hz. Their first samples exactly equal their aim endpoints. The arrow disappears at `events.release = 0.0`, leaving projectile flight to the game. The right hand follows back beside the jaw over 0.22 seconds; a two-degree body settle follows. Bow limbs relax while each string half keeps its 0.87 m length and aims at their common geometric meeting point. The bow stays raised at the end for a later reload.

An initial 0.10-second string return produced an approximately 11 mm separation between string halves during runtime interpolation despite exact saved keys. Extending return to 0.15 seconds supplies three intervals at 20 Hz and reduces the measured separation below 5 mm. Final videos were regenerated after this change.

## References and review

Inspected the M2TW peasant archer release sheet and all 29 ordered GIF frames before authoring. Compared ours at release 0.00/0.05/0.20/0.60 seconds with reference frames 0/4/8/12 by event. M2TW swings the draw hand farther out and lowers the bow for recovery. Our follow-through remains beside the jaw and keeps the bow raised for the separate reload. The local reference page explains every over-threshold measurement. Upward aiming is compared against the level hold at three phases, with its intentional elevation differences explicitly described; there is no matching elevated M2TW clip being claimed.

The page contains both shots from three fixed views and their own 24-second two-direction orbit, seven-view sheets at all four detail levels, and consecutive-frame sheets. The previews prepend the unchanged level draw, then either release or raise and release. A restart is not a reload. Reference images and comparison measurements stay in ignored `local_reference/`.

## Checks

All three motion clips pass the shared validator. Dense CPU-skinning checks cover 151 upward-aim times and 61 times per release at all four levels. Maximum sole drift is about 3e-16 m; ground minima are within 0.00004 mm of zero. L0 width peaks at 0.546892 m; all-level maximum is 0.597990 m. The preceding draw retains its separate larger bounds.

Maximum interpolated string contact error while aiming upward is 0.0055 mm. String-half separation during release peaks at 4.47 mm level and 4.26 mm elevated. Exact saved keys retain the common string meeting point.

Sampled L0–L2 bow/body, bow/bow-arm, draw-forearm/body, bow-forearm/body and visible-arrow/body surface checks are clear. L0 string/draw-hand contact occurs during held aim and release time zero, then clears. Lower-body geometry and motion are inherited; earlier coat/tube attachment limitations remain as documented in the ready iteration.

Five phases per clip were checked against ready, standing and phase-offset neighbors at 0.9/1.0/1.05/1.4 m in four directions. Lateral neighbors clear throughout. Some front/back arrangements intersect at the tighter three spacings; sampled new-clip arrangements clear at 1.4 m. The initial draw retains its reported 1.4 m rank crossing and brief string/sleeve contact at 0.15–0.20 seconds.

Quiver relocation and retrieval/reload remain deferred. No source, shader or engine-integration edits were made. Visual review of these new clips is pending.

## Final artifact review

Inspected the seven-view elevated poses, matched M2TW release rows, consecutive level-shot frames and final elevated-release frames. All eight MP4s decode fully. Firefox played each through a loop, including both complete 24-second orbits; browser captures were inspected. The minimum conservative orbit framing margin is 0.175 m. Links and reference images resolve. Fixed videos have 70 level-shot frames and 106 upward-shot frames at 24 fps.

Private assets commit: `180a4d0d96e23b5e36a29f339ae663a8e531c080`. Visual review remains pending.
