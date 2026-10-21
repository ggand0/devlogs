# Archer back quiver and sword draw

Written by GPT-6.

2026-10-21. Gota asked to work on the back quiver and sword draw together while away. These can proceed on the same approved hood_v4 body. The draw also needs a separate sword, left hip scabbard and bow stow, so those prerequisites are included. New review folders are `assets_dev/archer/quiver_v1/` and `assets_dev/archer/sword_draw_v1/`. The previous approved body, hood and arms are retained.

## Equipment

The hip quiver and belt knife account for 374 removed triangles. Their replacement also uses 374 triangles: closed quiver 76, harness 96, three visible arrow ends 72, sword hilt 52, blade 14, scabbard 32, scabbard hanger 12 and shoulder retaining loop 20. Total detail counts are 2,996 / 680 / 250 / 56, at 1.800 m. The 22 joint names, hierarchy, inverse bind data and existing pivot nodes remain exact. Two non-joint empties describe the sword grip and stowed bow.

The quiver runs from the left rear waist toward the right shoulder. Its shell includes an outer case, rolled lip, lining and bottom. A 38 mm shoulder strap crosses the chest and passes under the shoulder cloth. Arrow shafts and feathers are closed solids. The short sword has a 345 mm visible blade; its left hip scabbard angles down and back so extraction clears the chest cloth. The bow rests diagonally across the back with a leather retaining loop near the left shoulder.

Preservation checks compare every retained vertex attribute to hood_v4 and every L1-L3 attribute exactly. Existing used atlas pixels are unchanged; the neutral steel sample occupies an unused atlas patch. The textured model uses the atlas alpha as its team mask. Exported UV2 part data and VEC4 colour are retained, with one opaque atlas material. New equipment uses parts 0 and 19; the separate sword uses proposed part 20, rigidly skinned to hand_r.

## Problems found and fixes

- The reused hip-quiver shell and rim had open edges. A new closed cup replaces both. Three overlapping feather roots also created nonmanifold welded edges; their roots are separated around the shaft.
- The first back placement touched the shoulder cloth. Moving the case rearward clears the cloth and existing shooting clips. The upper harness was lowered under the scarf to remove isolated dark fragments on its front edge.
- The side-view review showed too much space beneath the quiver. Tilting its lower end about 89 mm toward the waist closes that gap while preserving the mouth position and shoulder clearance. The final draw and shooting checks remain clear.
- Averaging UVs across separate source islands sampled a dark unused atlas area. Material samples now use a face-centre UV within the intended island. The blade has its own neutral steel patch.
- The first sword extraction crossed the front cloth. Angling the scabbard forward at its mouth and back at its tip gives the blade a clear upward/forward draw path. The free hand stays out of that path.
- A single diagonal elbow pole could align with the reach direction and cause abrupt arm roll. Arm orientation now uses the original elbow bend plane, with fitted elbow angles that preserve the authored palm target and wrist orientation. Fitting rejects contacts and excessive per-frame turns, including quarter-frame interpolation checks.
- Turning the upright bow behind the body swept its shaft or string through the forearm. The bow is lifted into space beside the body, rolled end over end, then carried to the mount. The hand returns on the body side of the string. The wrist holds its stow orientation during initial withdrawal before turning forward.
- A 0.7-second stow left too little time for the carry and release. Stow now lasts 1.2 seconds. The sword still extracts in 0.45 seconds; the complete transition lasts 2.35 seconds.
- A mount beside the torso looked unsupported in the rear view. The bow now leans inward across the back and has a visible leather retaining loop. One scabbard hanger was simplified to keep the total below 3,000 triangles.
- The first guard kept straight standing legs and spread the elbows too far. The final guard lowers the elbows, turns the chest 12 degrees, drops the pelvis 35 mm and steps the left foot forward 140 mm. The right sole stays planted.

## Motion and attachment contract

`archer.draw_sword.json` contains 48 samples at 20 Hz, is non-looping and has zero external root travel. The first sample exactly copies `release_aim_v2/archer.stand.json`. The last sample exactly matches the new `archer.melee_ready.json`; it does not replace the existing bow-ready pose.

| Event | Seconds |
|---|---:|
| Bow stowed | 1.20 |
| Sword gripped | 1.40 |
| Sword clear | 1.85 |
| Melee guard | 2.35 |

The review evaluates the game's CPU skinning equations. Sword part 20 follows the pelvis/scabbard before 1.40 s and hand_r afterward. Bow parts 8, 13, 14, 15 and 17 follow the left arm before 1.20 s and the chest mount afterward. Arrow part 16 is hidden. These are explicit attachment rules in `equipment_pose.py`, not new skeleton joints and not behaviour implemented by the JSON alone.

The sword grip is (-0.365, 0.885, 0.190) m. The scabbard mouth is (0.185, 1.025, 0.135) m. The final bow mount is (0.400, 1.100, -0.400) m, with rotation recorded in `settings.json` and `pivot_bow_stow`.

## Validation

- Model SHA-256: `10ef07c23073c73cd403e0ff3df0ee51b752a107a78c69820d05b67644fd2a28`.
- Clip SHA-256: `1392bd6fbed06249e56d53d9268740870b37226e4b11c814ca7a083d6a54eb2a`. Loading `carry_elbows.json` reproduces the exact clip bytes.
- Quiver, harness, arrow shafts, feathers, blade, scabbard and retaining loop have zero open, nonmanifold, wrong-winding or degenerate edges/triangles in the new surfaces.
- At 189 interpolated draw poses, no tested quiver/head/arm/torso, blade/arm/head/torso or bow/head/torso/arm/quiver crossings were found. Intentional hilt/hand, sheathed blade/scabbard, covered strap/body and retaining-loop contacts are outside those pair tests.
- Quiver checks also cover 173 poses across the existing ready, draw, raised aim, level release and raised release clips, with no tested quiver/head/arm/bow crossings.
- Attachment changes displace vertices by less than 0.00012 mm at the same event pose.
- The shared clip validator passes: maximum joint turn is 1,402.97 degrees/s at hand_l, below its 1,500-degree/s threshold. This is a format/motion check, not an anatomical quality score.
- Dense support checking finds zero airborne samples, maximum planted-right-sole drift 0.245 mm and minimum ground height -0.236 mm. The left sole remains exact until the step begins at 1.85 s.
- The full visible sweep reaches 1.437 m width, 1.013 m forward, 0.708 m backward and 2.057 m height. These extents require formation and neighbour checks during integration; the draw has not been tested in a battle.

The checks sample edge/triangle crossings and cannot prove continuous clearance or containment. Only L0 has the new equipment. L1-L3 still contain the preceding equipment and need matching before shipment.

## References and scope

The peasant archer standing sheet supplies the back-quiver/harness layout. The original levy archer image supplies clothing and belt-sidearm context. The hand-gunner standing-to-ready sheet and ordered GIF frames expose whole-body settling, but do not show sword extraction. No corresponding bow-stow or sword-extraction reference was available in the local index. The reach, extraction and timing are original authoring; no reference pose score or extraction match is claimed. Raw reference imagery and transferred curves are not included in either iteration.

Both folders are review iterations. No public installed model, game source, LOD matching, reload or other melee clip changes are included. The game loader currently handles parts through 19; part 20 and the attachment events need coordinated engine work after review. No new iteration is committed before review. The prior hood commits remain the approved foundation.

## Review media

The review pages provide a textured equipment orbit, a 24-second animated orbit in both camera directions, fixed oblique and gameplay motion, and front/rear/left/right/top/event views. All 12 local image, video and page references resolve.

Final media verification: all four 960×960, 24 fps MP4s fully decode with ffmpeg. Equipment orbit: 288 frames / 12 s. Animated clockwise/counterclockwise orbit: 576 frames / 24 s. Fixed oblique and gameplay videos: 144 frames / 6 s each. Render records match the final model and clip hashes.

All four videos were played to completion in a separate headless Chrome instance. Playback reported exactly 288 / 576 / 144 / 144 decoded frames for the corresponding files, with no media errors. Inspected the browser's ordered captures, the full cardinal/event sheet and full-size rear/side views. The frame sequences show the complete feet, bow and sword throughout; the camera reversal does not reverse the animation. The repeating preview resets after the guard pause, while the delivered clip remains non-looping. This is an asset preview, not an in-game playback check.

Review pages: `assets_dev/archer/quiver_v1/review.html` and `assets_dev/archer/sword_draw_v1/review.html`. Integration note: `work/notes/101-archer-quiver-and-sword-draw-2026-10-21.md`. Both iterations remain uncommitted for visual review. No public source or installed asset changed.
