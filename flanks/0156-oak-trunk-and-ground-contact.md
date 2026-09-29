# 0156: Oak trunk and ground contact
Written by GPT-6 Astra.

Gota finds the oak trunks ugly and clay-like, with poor ground contact. Shrub B and birch are good enough and should keep their current appearance. References: resources/vegetation/m2tw_trees_ref1.png and ref2.png, inspected at reduced size. Ref1 supplies the close bark/root direction: finer brown-grey fissures, a broad root transition, and darker weathering near the ground. Ref2 shows how the trunks read at map distance. M2TW also has grass and cast shadows; this graphics branch does not yet have those. They contribute to contact but do not excuse the oak's angular joints and pale stretched bark.

## First revised specimen

Revise the natural-colour mature oak first for review, before applying the treatment to the other five oak shape/palette combinations. The Sandbox placement at (-16,-3), yaw=0.45, scale=1 now refers to mature_oak_trunk.glb. Original mature_oak.glb and every other tree/shrub file are untouched; Grassland remains the original specimen row. Private build: assets_dev/vegetation/oak_trunk_v1. Public candidate: assets/vegetation/mature_oak_trunk.glb, under the existing GLB LFS pattern.

The woody structure interpolates the original trunk/main-limb paths with Catmull-Rom segments, builds tubes with transported ring frames, and uses a 0.055 m voxel union plus smoothing to merge branch collars. Fine terminal wood hidden within the foliage is omitted; thicker top forks remain. The ground-plane cut is capped, then the two wood meshes are reduced to 1,400 and 270 triangles. Buttress lobes widen the base with unequal reach and angle. A first comparison showed an overly regular star-shaped flare, so the root pattern was made less uniform before the final capture.

A new original bark atlas was generated with built-in imagegen; exact prompt and source hash are in the iteration. It uses finer, subdued grey-brown fissures rather than the old pale broad streaks. Source is preserved unchanged; a 1024-square opaque RGBA image is embedded. Bark UVs follow local branch arc length at 1.8 m per texture tile. Per-face angular coordinates are unwrapped across the cylinder seam. The exporter declares mirrored texture wrapping, and src/vegetation.rs now honours explicitly mirrored sampler axes. Existing atlas sampling stays clamped. A smooth, spatially varying olive-brown vertex tint darkens the base without changing the leaves.

## Preservation and budgets

Both foliage meshes retain every triangle attribute and source texture pixel: 1,560 near triangles and 420 middle triangles. Blender re-quantizes custom normals on reconstruction (initial maximum difference about 0.000123), so the export pipeline restores the original GLB normals by position/UV after exporting. The raw comparison then matches all foliage attributes exactly. Far colour and volume normals are rebaked from the revised trunk and original leaves.

L0: 2,960 triangles, height 13.719443 m. L1: 690 triangles, height 13.836053 m. Far: four triangles with the existing 16 m padded frame. Budgets remain 3,000/700/4. Root origin is zero, applied transforms, GLB Y-up. The first export had a 2–7 mm negative base from decimation; vertices below the ground plane are corrected before final export. Raw verification checks closed consistently wound bark, finite attributes, opaque vertex alpha, bark/foliage materials, bounds and foliage equality.

Only the allowed vegetation source file changes runtime behavior; this is layered over the preceding uncommitted Sandbox placement. No terrain height, water, sim, pathfinding, shared RNG or new FL switch changes. No sim hash check needed.

## Review and remaining work

Review page: tmp/shots/0028_oak-trunk/index.html. Matched close camera: x=-16,z=-3, distance=15,pitch=0.38,yaw=0.65. Root view uses distance=8,pitch=0.55. Whole tree, western copse, reverse side and camera motion accompany it. Stop for this trunk treatment's art review before propagating to the remaining oaks. Birch and shrub B remain unchanged.

The guard detected successive target-agent/opt-dev/flanks processes during the work. No other game was killed. Read-only checks and documentation continued while GPU work waited.

Final asset validation passes: raw inspector, closed bark topology and exact foliage comparison at both mesh levels. Opt-dev build and strict all-target clippy pass. The shipped candidate matches the verified private GLB byte-for-byte.

Final stills and 22-second motion recording are saved and linked. Inspected 24 chronological frames at 4 Hz around the near approach, rather than full-frame-rate playback. Logs show mature_oak_trunk L0 -> L1 -> card -> L1 -> L0; no vegetation load errors. No battle performance claim. Private iteration checkpoint: eaa16ff. Public runtime and candidate asset remain uncommitted for art review, alongside the preceding Sandbox changes.

## Mature-oak treatment accepted

Gota accepts this version for the record, while noting that it may still benefit from refinement and that materials or missing shadows might account for some remaining oddness. Improvements retained: merged and smoothed woody junctions, uneven flared roots, finer grey-brown bark, consistent bark scale along branches, darker olive-brown weathering near the base, and matching far colour/normal bakes. Foliage stays byte-verified against the original attributes and pixels. The next requested specimen is the first upright oak, using the same trunk treatment in the Sandbox scene.

## Upright oak with the same treatment

The accepted mature-oak revision, mirrored bark sampler support and preceding Sandbox composition were committed publicly as 3b2ebc9, "Add a Sandbox copse and refine the mature oak trunk". Both new GLBs were confirmed under Git LFS. Private mature-oak acceptance table: c4b61ca. Gota asked to see the first upright oak with this trunk next.

Created assets_dev/vegetation/upright_oak_trunk_v1 using the original upright-oak branch structure and natural-colour foliage from oak_palettes_birch_v1/natural/oak. Root spread, root height, base-weathering height and voxel size scale by 0.67/0.85 (about 0.788), the original trunk-radius ratio, so it does not inherit the mature tree's oversized foot. Bark image, material colour and 1.8 m vertical tile scale are shared with the accepted mature oak. Far colour and volume normals are freshly baked. No new image generation.

Near wood has 1,350 triangles and foliage 1,320, totaling 2,670 versus the original 2,672. Middle wood has 270 plus 420 foliage, totaling 690; far remains four. L0 height 13.164443 m, L1 height 13.276800 m, root origin zero, identity transforms, GLB Y-up. An initial 0.263 mm positive root-plane offset from decimation was normalized before final export. Raw inspector and topology/material/bounds verifier pass; every foliage triangle attribute and texture pixel at L0/L1 matches the original exactly.

Sandbox's natural upright oak at (-26,1), yaw 2.10, scale 0.90 uses the new oak_trunk.glb. Existing oak.glb and the Grassland specimen row remain unchanged, as do the lighter upright oak, both leaning oaks, birches and shrubs. This turn's source diff adds only the asset registration and changes that one Sandbox placement name. Build and strict all-target clippy pass. No sim hash is needed for this asset/rendering-only change.

Review page: tmp/shots/0029_upright-oak-trunk/index.html. Matching before/after camera: x=-26,z=1, distance=14,pitch=0.38,yaw=-0.45. The initial yaw=0.65 close view was obstructed by the unchanged leaning oak. The final comparison uses an open-side angle; its before image temporarily loads the original upright GLB into the candidate slot, then restores the verified revision. Root, whole-tree, western-copse, whole-scene and reverse views accompany the comparison. Scene lighting is unchanged. Bark already uses a textured StandardMaterial (roughness 0.92, reflectance 0.15); near bark has no surface normal map. DirectionalLight.shadow_maps_enabled is false in this branch. Missing cast shadows and the dark shaded side remain plausible contributors to the artificial appearance and should be evaluated separately after the asset comparison.

Final upright-oak review: seven stills and a 22-second continuous zoom/small orbit are saved; all local review-page links resolve. Inspected 24 chronological samples at 4 Hz around the near approach, not full-frame-rate playback. Logs show all three oak detail levels without vegetation load errors. Final public candidate matches the verified private GLB byte-for-byte after the before-image swap was restored. Private iteration commit ad6c639. The upright-oak source/asset/README changes remain uncommitted publicly for this new art review; the accepted mature-oak version is already committed as 3b2ebc9.

## Lighter bark for this grassland

Gota likes the revised fine bark texture but finds its colour too dark for this map, citing debug7.png and m2tw_trees_ref2.png. Preserve the dark palette for other maps. The approved next test is a lighter warm stone-grey/tan bark variant on the upright oak alone, with the same geometry and lighting.

Built-in imagegen made a colour edit of the original generated project bark, retaining the fine plate/fissure appearance while lifting the midtones and softening contrast. It is a generated edit, so its fine pixel pattern is not asserted identical. Exact prompt, source hash and source PNG are saved in assets_dev/vegetation/upright_oak_bark_light_v1. No M2TW screenshot pixels enter the asset. Source stays intact and is packed into a 1024-square opaque RGBA bark image for the GLB.

The palette builder appends the existing upright near/middle meshes instead of regenerating them. Raw comparison verifies every mesh attribute unchanged at L0/L1/card, including wood normals/UVs/colours and the root-weathering tint. Foliage texture pixels also match exactly. Far colour and volume normals are rebaked. Counts remain 2670/690/4; near height 13.164443 m. Raw GLB inspector, topology/material/bounds verification, foliage comparison and full mesh-attribute comparison all pass. Build and strict all-target clippy pass.

Candidate assets/vegetation/oak_trunk_light.glb is a separate file. The previous dark oak_trunk.glb remains byte-identical (SHA-256 d180d036bceac237275cb08d1ec05690187f6471b54da2c9b226bf9a428c1f35). Only the upright oak at (-26,1) in Sandbox uses the light palette. Mature oak, remaining oak variants, birches, shrubs, terrain and lighting are unchanged. The source edit only registers the extra GLB and changes that Sandbox placement's asset name. No sim inputs, random sources or switches changed; no hash run.

Review: tmp/shots/0030_upright-oak-light-bark/index.html. Dark/light close and whole-scene captures use matching cameras; the dark captures come from the preceding upright-oak review. Close camera x=-26,z=1,distance=14,pitch=0.38,yaw=-0.45. Scene camera x=0,z=0,distance=90,pitch=0.62,yaw=0.65. Root, whole-tree, reverse-side and motion views are included. The lighter lit bark sits closer to the old pale trunks; the almost-black shaded face remains, illustrating the separate indirect-lighting issue. Stop for palette review before propagation.

Final light-bark review includes a 22-second zoom/small orbit. Inspected 24 chronological samples at 4 Hz around the near approach, not full-frame-rate playback. Logs show all three detail levels without vegetation load errors. Private iteration checkpoint: 7ae5a03, "Add a lighter bark palette for the upright oak". The dark and light upright candidates, registrations and README remain uncommitted publicly pending this art review.

## Accepted light bark across all six oaks

Gota accepts the lighter warm-grey trunk for this map and asks to apply it to the remaining oaks, update the small oak's texture/treatment to match, commit and document progress. All three shapes in both natural/lighter foliage colours now use the same accepted bark pixels. No new image generation or colour tuning. The dark mature and upright trunk files remain available for other maps; the standalone accepted light upright is also retained. The six preceding canonical files are backed up in work/backups/0031-oak-trunks-previous-assets as well as public history.

Private recipe: assets_dev/vegetation/oak_trunks_light_v1. Upright and mature builds append the already revised near/middle meshes and transfer each existing foliage palette. Their wood attributes are unchanged. The small leaning oak adopts the same Catmull-Rom paths, transported tube frames, voxel-joined collars, smoothing, capped ground-plane cut and asymmetric root flare. Root dimensions, voxel size and base-weathering height scale by 0.42/0.85 relative to the mature oak's base radius. Its lean and foliage silhouette stay unchanged. Bark uses the same 1.8 m arc-length tiling and mirrored sampler. Small wood budgets are 1200 near and 300 middle triangles; every variant gets a fresh far-colour and volume-normal bake.

| Shape | Near / middle / far triangles | Near height |
|---|---|---|
| Upright | 2670 / 690 / 4 | 13.164443 m |
| Mature | 2960 / 690 / 4 | 13.719443 m |
| Small leaning | 2344 / 664 / 4 | 9.001080 m |

Both foliage levels of all six outputs match every original triangle attribute and texture pixel, including each palette's vertex colours. Raw checks also prove shared accepted bark pixels, identical wood within each palette pair, unchanged accepted upright/mature wood attributes, finite channels, opaque bark, mirrored wrapping, zero root-plane offset and closed consistently wound wood. All six pass the raw GLB inspector. Build and strict all-target clippy pass.

The six canonical assets are replaced, so both Sandbox and the Grassland specimen row use the matching trunks. Sandbox positions, scales and orientations stay fixed; its mature tree uses the canonical mature_oak name. The runtime no longer loads unplaced archival trunk variants. Birch, shrubs, terrain, lighting and sim are unchanged. Source diff is vegetation asset names/fallback paths only, so no sim hash run is required.

Review: tmp/shots/0031_oak-trunks/index.html. Matched whole-scene before/after plus natural/lighter small oak, small roots, natural/lighter mature and lighter upright close views. Scene camera remains x=0,z=0,distance=90,pitch=0.62,yaw=0.65. The natural small oak is at (23,12); close distance10 and root distance6. Existing deep shade on the reverse side remains a lighting concern, not another bark-colour change.

Final captures include a 22-second camera sweep through distant and close views. Inspected 24 chronological samples at 4 Hz during the approach; this is not full-frame-rate playback or a performance benchmark. No vegetation load errors appear in the capture logs. All shipped files match their verified private outputs. Dark upright bark hash remains unchanged. Every local review-page link resolves. Public commit 950234f ("Unify oak bark and refine the small oak trunk"); private build/validation checkpoint 4d33771. All eight added/updated GLBs are confirmed through Git LFS. Devlog remains uncommitted as required.

## Oak set and composition accepted

Gota confirms the trees blend with the ground well enough and considers composition item 2 essentially done. Preserve the six current oak variants. Deferred tuning: darker foliage now slightly mismatches the lighter trunk, but keep it as-is for a future colour pass. No asset changes or new renders are needed for this acceptance. Public asset/runtime commit remains 950234f; private build checkpoint remains 4d33771. The next staged asset is upright Shrub A, reviewed in Sandbox before wider placement.


## Front-lit trunk assessment, 2026-09-28

Pale foliage is accepted and committed as 694eade; see devlog 0152. Gota reports clay-like trunks at moderate distance in resources/vegetation/debug10.png, acceptable distant appearance in debug11.png, and stronger form from the side in debug12.png. Inspected all three at reduced resolution. Broad pale trunk faces and smooth thick forks dominate the front view, while fine bark markings are difficult to resolve. Side lighting supplies the broad light/dark separation missing from the front. This is a visual diagnosis, not yet an isolated rendering experiment; the screenshots alone do not establish which oak detail level is active.

Local evidence: src/main.rs setup_world sets DirectionalLight.shadow_maps_enabled to false. The dark side in debug12 is therefore surface lighting, not a cast sun shadow. The mature_oak_pale GLB has a base-colour texture on oak_bark_tiled, with neither a normal texture nor an occlusion texture. The far-card material has a volume normal map, which does not provide bark relief on near/middle geometry. Runtime StandardMaterial uses roughness 0.92 and reflectance 0.15. The tree loader reads normal textures and generates tangents, but does not load occlusion textures. It transfers mirrored wrapping for base colour only; a future tiled bark normal map must also preserve its sampler wrapping.

Proposed next review, not implemented: keep the accepted colours fixed and use one mature oak at matched front/side cameras and moderate/close distances. First compare the existing asset with real canopy/branch shadows once the shadow work is available, checking both oak mesh detail levels. Shadow integration needs coordination because src/main.rs is shared and outside this task's edit allowance. Then assess a bark-relief trial with restrained medium-scale grooves that remain visible at the reported distance; fine grain alone is unlikely to address the broad smooth surfaces. If forks remain flat, test subtle cavity/ambient occlusion around junctions separately, with the appropriate runtime material support. Avoid baking a permanent directional dark side into the bark. Shape changes are a later option if the smooth thick forks remain the dominant issue after those comparisons.

No trunk, lighting, simulation or other asset changes were made for this discussion. Ground blending and leaf colours remain accepted. Missing cast shade and insufficient medium-scale surface relief are plausible contributors; neither is claimed as a complete fix before comparison.


## Trunks accepted with sun shadows, 2026-09-29

The earlier assessment described the pre-merge branch with cast sun shadows disabled. That is superseded by main merge b168a8d and cascade commit e593c33: sun shadows are enabled by default with three cascades reaching 280 m. Read the shadow integration note and tree-shadow handoff dated 2026-09-28/29. Inspected Gota's debug13.png and debug15.png at reduced resolution. Canopy shade breaks up the broad lit wood surfaces and ground shadows anchor the composition.

Gota explicitly approves everything in the Sandbox small composition with these shadows. Close this trunk review at the existing geometry, bark and foliage colours; do not proceed automatically to the previously proposed normal-map/cavity/geometry experiments or require another on/off comparison for acceptance. These remain optional if a specific issue returns. Next recommended stage is Grassland planting according to the original deployment-aware plan; see the new acceptance entry in devlog 0155. No code, asset or lighting changes and no additional renders were needed to record this approval.
