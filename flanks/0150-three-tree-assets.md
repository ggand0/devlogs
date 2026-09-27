# 0150: Build the three approved tree designs
Written by GPT-6 Astra.

2026-09-27. Graphics worktree, feat/vegetation.

## Scope and approvals

Gota approved mature oak A, smaller leaning oak B, and revised silver birch C. He explicitly asked to update the concept comparison page and start all three trees. The page now marks all three approved. This authorizes the three-tree asset stage together; shrubs, populated placement, wind and the cost gate remain later stages of this branch. Dense meadow cover remains a separate branch.

## Assets

New private iteration: assets_dev/vegetation/tree_set_v1/. build_tree.py contains the mesh/export/bake pipeline; structures.py contains the separate branch structures and species settings. The accepted oak_v2 and preserved oak_v1 remain unchanged, checked against SHA-256 hashes for all 44 files present at the start of this stage.

The mature oak has a thicker bole, heavy curved boughs and a broad irregular crown. The smaller oak has its own leaning trunk, seven primary boughs and a smaller asymmetric crown. The birch uses a slim upright trunk, ten ascending/outward primary branches and foliage distributed along their lengths. Its first internal render concentrated the foliage into isolated rounded groups; the retained version redistributes it into 33 groups along the branches and reduces the local normal weighting. The remaining coarse foliage grouping is a visual-review point, especially against the finer concept image. No claim of exact photographic reproduction.

| Tree | L0 / L1 / card triangles | L0 height | L0 width × depth | L1 height |
|---|---:|---:|---:|---:|
| Mature oak | 2912 / 636 / 4 | 13.719443 m | 13.492862 × 13.088769 m | 13.836053 m |
| Leaning oak | 2340 / 560 / 4 | 9.001 m | 7.085 × 6.791 m | 9.071 m |
| Silver birch | 1936 / 450 / 4 | 15.482 m | 7.344 × 7.042 m | 15.701 m |

Exact values are in each output folder's build_report.json and verified_glb.json. Caps are 3000/700/4 for the oaks and 2000/450/4 for the birch. The birch L1 reaches its 450-triangle cap. L0/L1 height discrepancies remain below 0.25 m, horizontal bound discrepancies below 0.45 m. Origin is the trunk base with minimum exported Y zero; nodes have identity transforms, export is +Y up. Each asset contains variant_0 with L0, L1 and card. Vertex attributes include VEC4 colour with alpha 1, UV0 for the atlas and UV1 bend/foliage weights. Unit part IDs and pivots do not apply.

The two new oaks copy the accepted oak atlas unchanged. The built-in image generator produced one original birch atlas with three triangular-leaf sprays and one pale marked bark quadrant. It returned real 1254 × 1254 RGBA on the first call. The exact prompt and texture provenance are stored in the iteration folder. No Reddit or Wikipedia photographic pixels enter the textures or meshes. Far colour and normal atlases were baked from each L0 into 1024 × 512 RGBA images. Plane frames are 16 m, 11 m and 18 m respectively, including transparent margins.

## Runtime

Only src/vegetation.rs changes in the public tree. The former single-oak resource now contains independent named assets. Each import enforces the supplied species caps before publishing handles. Each specimen records its asset index and selected level; the LOD system retrieves that specimen's mesh and material without affecting its neighbours. Failed imports skip only that asset and retain the specified review position for the others.

Grassland displays four trees at x=-495, z=-60 (accepted oak), -36 (A), -12 (B), +12 (C). They are review specimens within the scenery margin, not the final landscape population. Classic remains empty and River retains its procedural set. The existing projected-height thresholds, hysteresis, overhead fallback and mip filtering are unchanged. Oak texture content is shared between source atlases, but the specimen loader still allocates separate image/material handles per imported GLB. Consolidating those for population remains integration work.

Read the diff relative to the saved pre-stage vegetation.rs: only tree loading, render resources, specimen spawning and detail selection change. No terrain heights, water, blocking, sim RNG or new FL switches are introduced. No sim hash run is required for this stage. The public diff remains uncommitted on feat/vegetation for review; no final assets are promoted.

## Validation and review

All three models pass the raw glb_inspect.py check and verify_tree.py checks for attributes, indices, budgets, bounds, wind channels and embedded cutout materials. The first verifier run exposed a shadowed Python variable in the adapted validator; it was fixed and all three reports were rerun successfully. Build and strict all-target clippy pass. All four assets load successfully in game. Reports and logs are under tmp/runs/0016_vegetation-tree-set/.

The review is tmp/shots/0016_vegetation-tree-set/index.html. It includes an actual-scale lineup, individual 40/250/900 m captures, selectable L0/L1/far asset previews and a continuous camera recording. Asset previews are individually framed and cannot be used as a relative-size comparison. The current graphics branch has no cast tree shadows; in-game lighting is brighter than the asset previews. The screenshot fps overlays are incidental and provide no vegetation performance claim.

The process guard was run on the host before each Blender run, build, clippy and game launch. All launches used the wrapper. Capture scripts only read window pixels and stop the process they started. No synthetic desktop input, no other game termination, and no subagents.

Motion inspection: 88 sequential samples at 4 Hz across the full 22-second camera clip. The tool interface does not play video directly; the full recording is embedded in the review page. The samples show the trees through the zoom; at closest range the outer specimens can leave the frame. Logs confirm all four independently cross L0→L1→card→L1→L0→L1, with the 16/20 px and 108/132 px thresholds. This is sampled inspection, not a claim of inspecting every frame or seamless transitions.

Added complete-crown 40 m views at pitch 0.75 radians after the initial 0.5-radian close views placed the taller crowns against the upper frame edge. The review uses the complete views; originals remain on disk. Mid/far views use pitch 0.5. The source art files remain unchanged and all nine tree/distance images exist.

Private iteration commit: `ec8dc1a`, Add two oak variants and a silver birch. Public vegetation changes remain uncommitted for review.

## Checkpoint: four trees and accepted mature-oak colour (2026-09-27)

Gota accepted the mature oak's natural-colour revision (tree_set_v3) and asked to commit and document the four-tree progress. Public branch commit: `e0cdd0a`, Add four grassland tree variants. It includes src/vegetation.rs, the four GLBs in assets/vegetation/, their README and the Git LFS pattern. All four staged binaries were verified as LFS pointers and their on-disk SHA-256 hashes match the reviewed source GLBs exactly. No push or merge.

| Runtime asset | Saved source iteration | L0 / L1 / card triangles | Current status |
|---|---|---:|---|
| oak.glb | oak_v2/oak.glb | 2672 / 636 / 4 | First upright oak accepted; apply the mature oak palette in a later pass |
| mature_oak.glb | tree_set_v3/mature_oak/tree.glb | 2912 / 636 / 4 | Shape and natural green colour accepted; opaque bark correction included |
| leaning_oak.glb | tree_set_v1/leaning_oak/tree.glb | 2340 / 560 / 4 | Shape accepted; a subtle fresher green variation is the proposed next direction |
| silver_birch.glb | tree_set_v1/silver_birch/tree.glb | 1936 / 450 / 4 | Shape accepted; cooler subdued grey-green colour target recorded |

The colouring sequence is preserved: initial natural/pale foliage in tree_set_v1; stronger green generated textures and dark tint in tree_set_v2, rejected as too video-gamey; original natural oak texture restored with linear leaf tint (0.55, 0.64, 0.68) in tree_set_v3, accepted. See devlogs 0151 and 0152 and their comparison pages. The accepted source is unchanged project artwork, with material colour applied consistently to meshes and far-colour baking. The reference photos were not sampled into textures.

Of the two oak photographs, oak_ref0.jpg was the stronger qualitative palette guide: restrained warm greens with darker canopy areas. oak_ref1.jpeg supported the light/shade variation. Neither was numerically colour-matched. Gota suggested sharing the accepted colour with the first oak, then giving the small oak a slight variation; record this as the next pass, not a completed change. A subtly fresher, slightly warmer green is suitable for the small oak while keeping saturation restrained. For birch, the target is the cooler grey-green of the orange-circled tree in silverbirch_ref0.png; reddit_silver_birch.jpg remains a secondary reference.

The trunk fix currently ships only on mature_oak: separate opaque bark texture/material, continuous mirrored UV wrapping and opaque mip alpha. The other three saved assets still use combined cutout atlases and require that fix alongside their material revisions. This commit captures the branch checkpoint, not completed population or the forest cost gate. Grove placement, shrubs, wind, material sharing and performance work remain; dense meadow cover stays in a separate branch.

The source diff was reread: rendering only, no terrain height/water/sim RNG/new FL changes. Existing opt-dev build, strict all-target clippy and three mip tests pass; the asset copies are byte-identical, so no additional GPU run was needed for this commit. Builder scripts and reports remain in the private assets repository: ec8dc1a (three new trees), 905348f (opaque bark/material pipeline), 79827db (accepted mature-oak colour). Reproducible public builder promotion remains for the asset pipeline completion step.

Loader detail for subsequent work: assets/vegetation/<name>.glb now takes precedence over the private fallback. Merely rebuilding a private iteration or changing the fallback will not change the in-game specimen while the public asset exists. Deliberately update the corresponding public GLB when reviewing the next version; preserve the prior private iterations.

## Oak palette extension and birch candidate (2026-09-27)

The lighter mature-oak palette is accepted. All three oak shapes now have natural and lighter variants, six active oak assets total; earlier pale versions are retained separately. Birch now has a cooler grey-green colour candidate and the opaque-bark seam correction. Geometry and budgets remain unchanged. Private iteration c5e28ce and the detailed colour/validation record are in devlog 0152. Review: tmp/shots/0020_oak-palettes-birch/index.html. Public candidate assets and specimen entries remain uncommitted for this review.

## Accepted palette checkpoint and warm birch (2026-09-27)

Public commit 9dacc12 saves six oak natural/lighter variants, the accepted cool birch and pale archives through LFS. Gota retains the cool birch for other maps and requested a warmer version for this map. Private iteration f44c201 builds that separate candidate, with unchanged geometry, bark and triangle budgets. Updated review uses the lit side shown in debug5/debug6, with matched cool/warm views: tmp/shots/0021_birch-warm/index.html. Details and validation are appended to devlog 0152. Warm candidate remains in review.
