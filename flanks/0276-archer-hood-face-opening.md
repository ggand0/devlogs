# Archer hood face opening

Written by GPT-6.

2026-10-18. New review iteration `assets_dev/archer/hood_v4/`, correcting the unnatural face border marked in `refs/archer/archer_review5.png`. Hood_v3's overall size was judged suitable. Its flat brow, abrupt upper corners, nearly vertical cheek edges and wide pale lining made the opening look like a rigid frame. V3 stays on disk unchanged.

## Geometry and material

The opening now arches over the forehead, turns through the temples and narrows toward the lower jaw. Compared with v3, the upper corner moves from X ±82 mm, Y 1.716 m to X ±61 mm, Y 1.700 m. Temple edges sit at X ±68 mm, cheek edges at ±60 mm and the lower ends at ±42 mm. Side opening points move back toward the face, while the central brow remains at Z 118.75 mm. These are changes to the opening boundary, not a global hood scale.

The boundary lining turns inward 6 mm and back 3 mm to form a connected cloth hem. The inner hood samples the exterior wool rather than the pale lining strip. This removes the rigid-looking contrasting frame while retaining thickness and backface culling. Existing topology is reused: hood 172 triangles, scarf 192. No triangles were added.

The hood remains 237 mm wide and 272 mm deep, with its crown at 1.800 m. Exported scarf vertex attributes match v3 exactly. Hood extents match within floating-point tolerance; the lower lining bound shifts about 2 micrometres when normals are recomputed. The 2,940 retained arms_v7 source vertices, their attributes, all rig nodes and pivots, and the distance meshes remain unchanged.

## Validation

Export SHA-256: `8ff17dd69e0b69d68da5777e9bdfd615de502796ba7fb22f21aa892815c54be3`.

- L0/L1/L2/L3: 2,996 / 680 / 250 / 56 triangles; height 1.800 m; original 22 joints.
- Hood part 18 and scarf part 19: zero open, nonmanifold, wrong-winding or degenerate edges/triangles in the new shells. Positive signed volumes: hood 0.001892 m³, scarf 0.001167 m³.
- Raw GLB attributes and surface reports saved beside the model. UV2 part data and VEC4 vertex colour remain present.
- 256 sampled poses: neutral, ±55-degree head turns, stand, ready, draw, raised aim and both releases, using the game skinning equations. No detected new garment self-crossings or tested exterior crossings against the face, body, arms or bow/string.
- The preceding scarf's concealed lining still crosses the torso, up to 24 detected edge crossings. Minimum new-triangle area ratio remains 0.0775. The intentional hood/scarf collar overlap is not claimed to be intersection-free. Sampled edge/triangle checks cannot prove continuous clearance or containment.

## Review

Matching 640-pixel face views cover front, both three-quarter angles and both sides; upper-body views cover front, rear and both sides. Textured head turns compare v3 and v4 side by side, 96 frames at 24 fps. The textured raised-shot orbit is 432 frames at 24 fps. Close views and sampled motion frames are inspected from the exported model; these are not in-game captures or a claim of real-time playback.

This remains an L0 face-opening checkpoint. Distance meshes, the retained 22-joint rig's older-spec exception and the other limitations in devlog 0275 remain. No public game asset or code changed. The new iteration stays uncommitted pending review.

Final review checks: both videos fully decode without errors and identify the final export hash. All 13 local links in the review page resolve. Inspected all five close angles, rear silhouette, head-turn contact frames, the full raised-shot orbit contact sheet and a full-size raised-aim frame. Hood_v3's saved GLB hash still matches its preceding report.


## Full-body review follow-up

The face opening looks provisionally suitable, but the full figure still needs review. Added full-body textured orbits to hood_v4: neutral (8 seconds), level draw/release (12 seconds), and raised aim/release (18 seconds), all at 24 fps. The raised-shot renderer now has a separate `up_full` mode, preserving the earlier close view. Each full-body orbit fits one fixed orthographic scale across all its frames, including feet, bow tips and the visible nocked arrow, with 6% minimum border clearance. The model and animations are unchanged. Shot sequences restart after release; there is no reload animation in these previews.

Full-body preview checks complete: 192 / 288 / 432 frames, each fully decoded without errors. Neutral resolution is 960×1080; both shooting videos are 1080×1080. Fixed orthographic scales are 2.456 / 3.237 / 3.312 m. All three manifests identify the unchanged v4 export. Inspected the three orbit contact sheets and full-size front, side, rear and raised-aim frames for full-body framing. All 19 review-page links resolve. This is rendered preview evidence, not in-game playback.


## Approval and source preservation

Gota approved the current model provisionally after the full-body neutral and shooting orbits on 2026-10-18. This closes the arms and hood shape review for now. The extra hood work covers the reduced width/depth and restored shoulder coverage from v3 (devlog 0275), followed by v4's arched brow, closer temple/cheek/jaw boundary, 6 mm inward and 3 mm backward hem turn, and wool-coloured interior replacing the wide pale border. The v4 opening changes preserve the chosen hood size and every v3 scarf vertex attribute.

Store hood_v4 with its build scripts, lining fit, reports, review images and videos in the private assets repository. Preserve only the three hood_v3 build inputs needed to reproduce the comparison GLB and retained-vertex map; the other v3 review work remains on disk. Build outputs and frame sequences stay untracked. Promote the garment build to `tools/blender/archer/hood/` on `feat/archer-hood-source`, with explicit source, wool and working-directory inputs. The public game GLB remains unchanged because the full model brief and distance-level matching are not complete.

Next work, one checkpoint at a time: move the quiver to the back over the right shoulder and add its chest/back harness; add the short sword or knife, left-hip scabbard and bow-stow transform on the current rig; match L1–L3 to the completed model and check silhouette/team colour at game sizes; then author the back-quiver reload, sword draw, cut, parry and charge. The original handoff specifies no shield block for this archer. Public integration follows the completed model checks. No further model or animation changes are included in this approval commit.


Committed: private assets `main` commit `f95967a`, **Add the archer hood v4** (71 files, 43 image/video files stored as LFS pointers). Public graphics branch `feat/archer-hood-source` commit `5123c44`, **Add the archer hood and scarf build** (five source/data/documentation files). Neither commit was pushed. The preceding arms_v7 source remains in private commit `1f2b78c`.

The promoted scripts were run in the separate `tmp/runs/archer-hood-source-check/` directory. Their output SHA-256 exactly matches the approved v4 GLB, `8ff17dd69e0b69d68da5777e9bdfd615de502796ba7fb22f21aa892815c54be3`; raw GLB and surface inspectors completed on that output. Both staged diffs passed whitespace checks. Commit messages were read in full, with a backstop search for attribution, people/model names and prohibited wording. Shared skeleton edits and other iterations were excluded. No Rust, shader, simulation or installed model changes were made.
