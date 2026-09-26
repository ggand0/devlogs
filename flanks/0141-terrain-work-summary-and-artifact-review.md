# Terrain work summary and artifact review
Written by GPT-6 Astra.

Date: 2026-09-26. Worktree: `/home/gota/ggando/gamedev/flanks-gfx`. Branch: `feat/terrain-look`.

Verified that `./devlogs` is a symlink resolving to `/home/gota/ggando/gamedev/flanks/devlogs`. This entry summarizes the terrain work through the review of debug0.png to debug5.png. No source changes, builds or game runs were made for this documentation/review step.

## Direction and scope

The reference is `resources/map/grassland_map_ref.png`: an M2TW-like grassland battlefield, with realistic units and eventually enough room for 200,000 soldiers. Initial proposal: `assets_dev/terrain/proposal_v1/proposal.md`. Exploration is recorded in devlog 0137.

The agreed sequence is surface repetition, distance-appropriate surface detail, then a deliberately shaped and larger battlefield. The first two have working implementations. The third has not started. Current terrain remains 1,024 by 768 m with 2 m cells and 393,216 triangles. Heights, movement, blocking and wading rules are unchanged. The river map is an experiment made with Kimi K3 and must be redesigned if taken forward; it is not the accepted base design.

Work is confined to the graphics worktree. Claude's main-worktree performance work and shared source files were not edited. The larger map will need deployment and simulation/performance coordination rather than only multiplying terrain dimensions.

## Saved implementation

### Surface v1: 42e30de, Add blended pasture terrain materials

Replaced absolute-height color bands with pasture, soil and stony-ground materials. Downloaded Poly Haven CC0 Grass Ground, Grass Path 2 and Brown Mud Dry directly. Asset URLs, checksums and license links are recorded under `assets/terrain/`; an offline preparation script packs color and normal/roughness into six mipmapped KTX2 textures. Normal maps are already present and sampled by the terrain shader.

Converted terrain chunks from triangle soup to indexed meshes with shared smooth normals. Vertex count fell from 1,179,648 to 209,088, with triangle count unchanged. Mesh attribute/index storage fell from roughly 45 MiB to 12.48 MiB per copy. Kept crater deformation, added exposed-soil rendering and included neighboring normal stencils in remesh invalidation.

Review found a clear improvement but visible repeated texture cells. Fixed-view diagnostics later separated fine texture rows from broad boxy coverage patches. Lighting-only diagnostics at those views did not show either pattern.

### Surface v2: c9303f3, Reduce terrain repetition across viewing distances

Added a single 513 by 385 coverage image over the whole battlefield, with irregular warped regions and local terrain relief. It supplies dryness, exposed-ground coverage and brightness variation, and uses 790,020 bytes of GPU storage. Hollows tend to be greener than nearby shoulders.

Filtered the pasture source into neutral fine detail, using periodic log-luminance filtering to remove broad source clumps and hue. Grass samples rotate by arbitrary angles, with variance correction to reduce contrast loss in blends. This is not the full histogram-preserving stochastic-texturing algorithm. Separate samples cover 6 m for close detail and 34 m for broader turf variation. The latter was added because the first v2 candidate looked too smooth at gameplay distance.

Close color detail fades between 65 and 300 m from the camera; normal detail fades between 50 and 220 m. Projected texture footprint also controls the fades, including the broad turf layer. The broad turf layer changes color without a corresponding normal layer at that scale. Negative values of the existing `FL_CAM_SWEEP` variable add continuous zoom/orbit capture; positive values retain the previous jump behavior.

Both commits are local on `feat/terrain-look`. No push or PR has been made. Current code was clean at the start of this review.

## Verification and artifacts

Build and strict clippy passed with the opt-dev profile and eight jobs. Both existing geometry/crater tests passed. The changed texture passed Khronos KTX2 validation. Terrain dimensions, simulation methods and generation were compared against the prior commit and remain unchanged, so no simulation hash run was required under the rendering-only rule in AGENTS.md.

Captured close, gameplay and wide views, a 12-second continuous zoom/orbit, crater changes, and 200,000 soldiers. Consecutive sampled motion frames were inspected; this was not exhaustive frame-by-frame playback. Runtime logs had no shader errors or panics. Ten v2 crater rebuild samples took 0.08 to 0.27 ms for one to four chunks, not a controlled crater benchmark.

Six bounded GPU checks compared plain PBR against v2 with 200,000 soldiers. At the gameplay camera and recorded 1,935 MHz, the opaque pass measured 0.10 ms plain versus 0.34 ms v2, about +0.24 ms (two matched-clock samples each). Close and wide clocks varied; there is no all-view sustained full-clock performance claim. Full limitations and numbers are in the v2 handoff.

- v1 review: `tmp/shots/terrain-surface-v1/index.html`.
- v2 before/after views and clips: `tmp/shots/terrain-surface-v2/index.html`.
- v2 logs, measurement CSV/JSON and helpers: `tmp/runs/terrain-surface-v2/`.
- Handoffs: `work/handoffs/HANDOFF-terrain-surface-2026-09-26.md` and `work/handoffs/HANDOFF-terrain-surface-v2-2026-09-26.md`.
- Implementation devlogs: 0139 and 0140.

## Current visual feedback

Gota considers v2 better than main and v1, and good enough as a baseline. The remaining discomfort is the mottled surface pattern, described as felt stretched over very smooth hills. Gota specifically suspects the texture patterns rather than lighting. This feedback should guide the next comparison; do not assume that adding lighting or stronger normals solves it.

Reviewed resized copies of `resources/map/debug0.png` through `debug5.png`. Debug2/3 make the dark mottling particularly clear. Debug5 annotates several curved, roughly parallel bands across slopes visible in debug4. These interior bands are the concern, separate from the obvious outside edge of the finite terrain sheet and the deployment boundary.

The mottling and distant bands may have different causes. A plausible contributor to mottling is the enlarged 34 m turf sample and its contrast amplification. The shader's footprint-dependent fade varies with distance and slope, so it is also a candidate for changes in apparent texture strength across hills. Other candidates are the rotated-sample blend, mip filtering and coverage variation. The existing heightfield and its shading can also produce creases; they are not ruled out by these screenshots. No specific cause of the annotated bands has been confirmed by an isolated render of that camera view.

The stochastic-texturing research specifically discusses color deviation at coarser mip levels and a prefiltering solution: https://eheitzresearch.wordpress.com/738-2/. That is relevant background, not proof that the same failure occurs here; our simpler variance-corrected shader differs from that implementation.

## Proposed next material investigation

1. Reproduce the annotated distant view and temporarily remove only the broad turf layer. Keep the camera, terrain and lighting fixed. This distinguishes the new enlarged texture pattern from other surface/shape contributions.
2. Inspect coverage color alone, neutral shaded geometry, and the detail layers separately at that same view. Freeze the turf fade and inspect selected mip levels to distinguish actual ground patterns from filtering transitions. Camera motion should reveal whether a band stays fixed on the terrain or tracks the view.
3. If the turf layer causes the mottling, replace the enlarged grass marks with quieter, purpose-made intermediate-scale variation and greater areas of calm ground. Avoid simply increasing blur or scattering more noise everywhere. Retain fine grass detail close to the camera.
4. If the bands come from filtering/blending, correct the responsible transition while preserving average color and controlling contrast across distance. Do not hide them with additional effects. If they survive the constant-color geometry view, investigate the heightfield/normal continuity separately before a simulation-affecting shape edit.
5. Compare the same close and annotated distant views, then continuous camera motion, before accepting another surface revision. Broader battlefield reshaping remains a subsequent stage.

No v3 implementation is included in this entry. The next step is a focused material isolation pass rather than a lighting or stronger-normal-map pass.
