# Wall shape review

Written by GPT-6.

2026-10-23. Saved the agreed three-review plan as `docs/plans/028-wall-models.md` and built review 1 only in `assets_dev/wall/shape_v1/`. Handoff 145 controls the dimensions; the reference sheets and the local M2TW screenshots informed the proposal. No texture work, tower, stair or engine changes.

## Shape

One continuous closed wall mesh, length 2 m along exported X, thickness 10 m along Z, walkway at Y = 8 m. The outer face is +Z. The origin is the footprint's ground centre. The two 1 m edge strips leave 8 m clear. Outer parapet and inner kerb are 0.5 m high. One 1 m wide merlon centred at X = 0 ends in a 0.35 m pyramid, reaching 9.5 m. Repetition gives 1 m gaps and 2 m pitch. The landing switch removes only the inner kerb.

## Evidence

`validation.json` reads the exported GLB geometry: wall 102 triangles, landing 86; both bounds X [-1, 1], Y [0, 9.5], Z [-5, 5]. Both have matching end profiles, positive signed volume, zero open edges, zero nonmanifold edges, zero winding inconsistencies and zero degenerate triangles. One opaque material, culling enabled, normals and UV0 present. The handoff exempts these models from soldier part IDs and team colour channels. `glb_inspection.txt` records the required raw attribute inspection.

The normal walkway surface is 16 square metres per 2 m unit. The landing has 18 square metres because the removed kerb exposes another metre at the inner edge. That is model surface, not a change to simulation walkability.

Seven plain-material EEVEE renders show an individual unit from outside and inside, an orthographic end profile, three units from above and the front, and a 24 m run from both sides. Orange figures are 1.8 m tall. `review_sheet.jpg` combines six views; `review.html` links the full images and measurements. The two single-unit cameras were widened before handover to keep the ground figure visible. `wall_review.blend` preserves the assembled scene. The build ran in Blender 5.2.3 after checking that no game was running.

## Review

Review the merlon width, pyramid shape, gaps, parapet and walkway proportions. Stop here until Gota reviews the shape. Stonework is review 2; final textured repetition is review 3. No commit before this iteration is approved. The existing assets_dev README changes and all unrelated work were preserved; only a wall row was appended.
