# 0155: Sandbox vegetation composition
Written by GPT-6 Astra.

Gota requested item 2: a small organic arrangement of the accepted oaks, warm birch and B shrubs, in the new Sandbox map rather than Grassland. Read Claude's sandbox-map-for-astra-2026-09-27 note. Base commit f0ebf35 supplies the separate map kind on this graphics branch.

## Placement and isolation

Split the Grassland/Sandbox vegetation match. Grassland keeps its western specimen row and every existing asset. Sandbox has six oaks (all three shapes, natural and lighter palettes), two warm birches and twenty low shrubs. Roots span x=-30..30, z=-18..18 m near the centre. The western copse is tighter; the eastern group is more broken, with a broad opening between them. Small shrub groups follow the outer edges and a pair reaches into the opening. Tree scale is 0.83–1.08; shrub scale is 0.72–1.24. Yaw varies per instance. The first capture looked too evenly spaced, so the western copse and several shrub groups were tightened before the review captures.

A shared spawn helper samples the existing terrain height, applies yaw/scale and sinks roots by scaled height clamped to 1–8 cm. LOD selection uses scaled projected extent and a transformed centre, so smaller and larger copies transition at the intended screen size. Identity scale/yaw preserves Grassland behavior. The Sandbox shrub has no specimen-row coordinate. Assets are still loaded at startup, so this does add a loaded GLB even on other maps.

Only src/vegetation.rs changes runtime behavior. Terrain heights, water, simulation, shared random sources and pathfinding are untouched; no new FL switch. Build and strict all-target clippy pass. No simulation hash run is needed for this rendering-only diff.

## Shrub distance levels

The accepted v4 still had the v1 middle mesh and far cards. Created a separate Sandbox asset so the fuller canopy carries to distance without altering Grassland's approved asset. Private iteration: assets_dev/vegetation/shrub_b_lod_v1. Public candidate: assets/vegetation/shrub_b_sandbox.glb, covered by the existing LFS rule.

L0 is appended untouched from the v4 scene: 1,854 triangles, 2.5 by 1.85 by 0.8 m. The verifier compares every triangle's vertex attributes and both texture images against v4, finding zero removed or added triangles and identical pixels. L1 copies all 222 bark triangles and selects 176 of the 704 deterministic sprays, keeping outside corners for folded sprays. Each is a two-triangle quad enlarged by 1.85 around its base, constrained to the original envelope. Total L1: 574 triangles, 2.4622 by 1.85 by 0.8000 m. This deliberately exceeds the old 120-triangle middle mesh to retain foliage coverage. Far remains four triangles, with 512 by 256 colour and volume-normal atlases freshly baked from v4. The transparent-padded card frame is 3 m; visible shrub height remains 0.8 m.

Raw glb_inspect, verify_shrub and verify_near pass. Output and shipped candidate match. This is a small appearance experiment, not evidence of cost at forest density; triangle counts alone do not capture alpha overdraw.

## Review

Page: tmp/shots/0027_sandbox-composition/index.html. Eight stills cover overall composition, both groups, shrub close detail, overhead, reverse lighting, a wider camera and the original Grassland view. The overview uses x=0,z=0, distance=90,pitch=0.62,yaw=0.65. Select Sandbox and Scene from the menu, then zoom into the centre, or use the page's launch command.

Grassland comparison against tmp/shots/0026_shrub-b-fuller/fuller-close_4s.png: all 1,120,000 world pixels in rows 140–839 match exactly; HUD excluded. Both use x=-495,z=0,distance=8,pitch=0.5,yaw=0.7. Existing GLBs are unchanged.

The 22-second composition clip uses the existing continuous zoom/small orbit from 42 to 900 m. Inspected 44 chronological samples at 2 Hz, not full-frame-rate playback. Close shrub transitions have a separate 22-second recording focused at x=9,z=14 with an 8 m close distance. Inspected 16 chronological samples at 4 Hz around the close approach; logs confirm all three shrub levels. The shrubs keep a low branching silhouette across the sampled transitions. Only existing gamepad-mapping warnings appear in capture logs. No populated-battle performance claim. This branch still has no tree cast shadows or wind.

Stop here for placement review before expansion or upright shrub A. Runtime changes remain a review candidate; private asset iteration is checkpointed separately.

Private asset iteration commit: bd99571. Public vegetation source, README and Sandbox GLB remain uncommitted for placement review. Review-page local links resolve.

The Sandbox composition and separate shrub detail asset were committed with the accepted mature-oak trunk in public commit 3b2ebc9. Subsequent upright-oak review is recorded in devlog 0156.

## Item 2 complete

Gota accepts the small composition after the oak trunk refinements: the trees now blend with the ground well enough. Item 2 is effectively complete: six irregularly spaced oaks, two warm birches and scattered B shrubs, with clear gaps and varied orientation/scale. Latest review: tmp/shots/0031_oak-trunks/index.html. Composition/runtime work is committed as 3b2ebc9; the six matching oak trunks are committed as 950234f. The public worktree is clean.

Deferred art note: the darker leaf palette now contrasts slightly awkwardly with the paler trunk. Keep the current appearance; revisit this combination during a later colour pass. This is not a blocker for item 2 and no recolouring is requested now. Devlog 0156 records the trunk technique and checks.

Next staged action is upright Shrub A from the accepted concept in assets_dev/vegetation/shrub_concepts_v1/upright.png: a roughly 1.7 m tall, 2.2 m wide multi-stem shrub with foliage close to the ground. Use the accepted B shrub's fine foliage treatment as the quality reference. Build and review one specimen beside B and the trees in Sandbox before expanding placement. Original concept triangle caps are provisional; B's accepted detailed asset already exceeds its early target. Dense ground vegetation remains a separate branch.
