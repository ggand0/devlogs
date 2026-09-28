# 0157: Upright shrub A
Written by GPT-6 Astra.

After accepting item 2's Sandbox composition, Gota requests upright Shrub A. The accepted reference is assets_dev/vegetation/shrub_concepts_v1/upright.png: an uneven dense multi-stem crown, short visible basal stems and foliage near the ground, roughly 1.7 m tall and 2.2 m wide. Build one specimen for review before wider placement. The earlier 600/160/4 concept caps were provisional and predate B's accepted fine-foliage treatment.

## Geometry and materials

New private iteration assets_dev/vegetation/shrub_a_v1. Seven distinct authored stems spread from the base, with different heights and reach; fourteen forks arise at different points along them. Tapered closed tubes use five sides on main stems and three on forks. Alternating, deterministically oriented shoots follow each branch, rather than placing isolated round tree crowns. Shorten entire low sprays where needed to clear the soil. Normalize the near envelope to 2.2 by 1.85 by 1.7 m and transform normals with the inverse scale.

The 840 shoots include 210 shallow folded sprays and 630 flat sprays, totaling 2100 foliage triangles. Wood adds 448, for near 2548. The middle mesh retains the wood and selects 210 distributed sprays, expands them by up to 1.8 around their bases, and fits them to the near envelope: 868 triangles. Far detail has four triangles, with fresh 512 by 256 colour and volume-normal atlases from A's own canopy. Both mesh levels measure 2.2 by 1.85 by 1.7 m. Far geometry has a padded 2.7 m frame. Root origin zero, identity transforms, GLB Y-up. Working caps 2800/900/4.

Reuse the accepted B small-leaf and bark textures exactly; no image generation or new texture colour edit. Near leaf image is 1024-square RGBA; bark is 256-square with opaque alpha. Leaf multiplier (0.66,0.73,0.72) matches B, with 0.88–1.05 per-spray variation. Raw decoded texture pixels match B exactly; texture hashes are recorded in the iteration. UV0 samples those textures; UV1 stores bend/foliage markers; vertex alpha remains opaque. Wood boundaries are closed and consistently wound; overlapping fork roots are concealed within their parent stems.

## Integration and checks

New candidate assets/vegetation/shrub_a.glb. One Sandbox specimen at (14,14), yaw0.30, scale1, beside B shrubs and the small oak/warm birch group. Existing plants remain fixed; Grassland's row is unchanged. Source changes only add the asset registration with 2800/900/4 caps and this one tuple in src/vegetation.rs. No sim inputs, terrain heights, water, shared RNG or new FL switch; no sim hash required.

The initial candidate passes raw GLB inspection and geometry/material/topology/LOD-bound checks. Build and strict all-target clippy pass. Blender near/middle previews are inspected; in-game capture was blocked by a running flanks-new benchmark. Gota confirmed the GPU is still in use; no other process was killed. Documentation and packaging continue while captures wait.

Repository build scripts are being packaged under tools/blender/shrub_a, reading embedded texture bytes from assets/vegetation/shrub_b_sandbox.glb to remove private-scene dependencies. Their output still needs comparison against the initial candidate when GPU work is available. Review captures are planned under tmp/shots/0032_shrub-a, with close/detail, gameplay distance, context, reverse, overhead and camera-motion views. No public acceptance commit before the specimen review.

Private iteration checkpoint: 715c2cc. Public candidate/runtime/README and standalone build scripts remain uncommitted for review. The repository raw verifier passes on the initial candidate; it uses the corrected public accessor reader without the older private helper's index-normalization workaround. The standalone recipe itself still awaits a Blender run when the GPU is released.

## In-game review completed, 2026-09-28

Gota released the GPU. The guard reported no game, and the standalone repository recipe built successfully from assets/vegetation/shrub_b_sandbox.glb without any private Blender scene. Raw comparison with the private initial candidate matches every triangle attribute and decoded colour/normal texture pixel at L0, L1 and card. Repository raw verification passes. Comparison script is saved as work/scripts/compare-shrub-a-rebuild.py. No geometry or rendering-source changes were needed, so the existing passing build/clippy checks still apply.

Review page: tmp/shots/0032_shrub-a/index.html. Six game stills cover close, branch/leaf detail, gameplay distance, context with B and the oak/birch group, reverse side and overhead. The accepted concept is included. Close camera focuses (14,14), distance5, pitch0.34, yaw-0.45. Detail uses distance3.6 and crops the upper crown to show branch/leaf detail. Gameplay distance15; context30. The upright form reads clearly, but the in-game canopy is more open than the concept. Leave this first specimen unchanged for Gota's review rather than starting another density revision before feedback.

A 22-second continuous zoom/small orbit is recorded. Inspected 24 chronological samples at 4 Hz around the approach, not full-frame-rate playback. Logs confirm near -> middle -> far -> middle -> near, then middle as the camera withdraws, with no ERROR lines. No battle or dense-placement performance claim. No capture game remains. Existing plant geometry, palettes, placement and lighting are unchanged.

Review links resolve and the shipped candidate remains byte-identical to the validated private asset. Private review checkpoint 4170715 supplements build checkpoint 715c2cc. Public candidate, vegetation registration, README and standalone tool scripts remain uncommitted pending the first specimen review. Stop before expansion or another shrub variant.

## First specimen accepted; density comparison

Gota likes Shrub A, asks whether it is as dense as B when seen at comparable size, and requests committing this version regardless. The circled A/B comparison is resources/vegetation/debug8.png; debug7.png is the older oak-trunk scene. Inspected both after resizing.

Added a practical in-game comparison at tmp/shots/0033_shrub-density/index.html. A: focus(14,14), distance6, scale1. B: focus(9,14), distance7.64, placed scale1.12. Both cameras use pitch0.50 and yaw-0.45. The distance ratio follows the widths (2.2 versus 2.5*1.12) to bring them to comparable screen widths. Existing yaw, height, terrain position and perspective remain distinct; this is a visual comparison rather than a precise foliage-coverage measurement. Both specimens are near meshes.

A still reads more open, especially around the taller stems and upper crown. It has 840 sprays versus B's 704, but occupies a taller crown (1.7 m versus 0.8 m), with different spray lengths/orientations and distribution. Spray count alone is not density. Increasing apparent size exposes the existing gaps; it does not fill them. Preserve the accepted fine-leaf style and this current version. A possible later density pass should add branch-attached fine sprays in the upper/middle gaps while keeping the irregular outline and visible basal stems; no such edit is requested or made now.

Current acceptance includes the candidate, one Sandbox specimen and standalone reproducible scripts. All prior geometry/material/texture, build and strict-clippy checks remain applicable; only a helper docstring changed during this commit step. No new GPU build was needed. Existing assets, simulation, placement and lighting remain unchanged.

Accepted public commit: 4c90248, "Add an upright shrub to the Sandbox vegetation". Private acceptance record: f285fca. The new GLB is confirmed in Git LFS. Public working tree is clean after the commit; devlogs remain uncommitted and backed up.
