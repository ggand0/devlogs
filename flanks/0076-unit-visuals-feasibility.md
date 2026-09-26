# 0076: Unit visuals overhaul, feasibility and measured cost (2026-09-20)

Owner question: what does it cost to replace the code-built cuboid soldiers with authored models, and how do we reach the visual level of Ultimate Epic Battle Simulator (UEBS)? Investigation only. No renderer change yet.

## Where the renderer is today

- One instanced draw per unit kind (4 draws) plus 4 corpse buckets. Custom WGSL pipeline in the Transparent3d phase.
- Meshes are merged cuboids: knight 204 tris, man-at-arms 192, spearman 204, archer about 500.
- No textures. Per-vertex color, alpha = team-color blend amount.
- No skeleton. The UV channel carries a body part id and a pivot height. The vertex shader rotates rigid parts about their pivots.
- All animation is driven by sim signals per instance (yaw, walk amount, lunge progress, stance band, march phase, wall, stagger, death). It is pose parameters, not clip playback.
- Lighting is a hardcoded sun lambert in the vertex shader. Shadow maps are off globally. Units cast and receive no shadows.
- No LOD. Every soldier draws the full mesh at every distance.

## The probe: FL_MESH_TESS

Added `FL_MESH_TESS=n` in unit_meshes.rs. It splits each cuboid face into an n x n grid. The silhouette is identical and the triangle count rises by n squared. Default 1, so normal runs do not change. Build and clippy clean. NOT committed.

Run recipe (muted, scripted battle, fixed camera): `FL_TEST_FRONT=1 FL_UNITS=100000 FL_AI=0 FL_ENEMY_STATIC=1 FL_VOLUME=0 FL_CAM_LOCK=1 FL_CAM_DIST=<d> FL_MESH_TESS=<n>`, plus `FL_NO_CULL=1` for the draw-everything case. Read `main_transparent_pass_3d` from the log. Trust GPU pass times, not fps.

WARNING: TESS 5 and 7 at 200k push 1 to 2 billion triangles per frame. That starves the desktop compositor and the whole session lags. Stay at TESS 4 or lower on a machine someone is using.

## Results (RTX 3090, opt-dev, 197k soldiers alive)

Wide shot, DIST 900, culling off, all 197k drawn:

| TESS | tris per soldier | tris per frame | unit pass GPU |
|---|---|---|---|
| 1 | about 206 | 41M | 5.5 ms |
| 2 | about 830 | 162M | 13.2 ms |
| 3 | about 1,860 | 365M | 25.0 ms |
| 4 | about 3,300 | 650M | 42.9 ms |
| 5 | about 5,160 | 1.0B | 70.2 ms |
| 7 | about 10,100 | 2.0B | 137.5 ms |

Gameplay zooms, culling on:

| view | soldiers drawn | pass at TESS 1 | pass at TESS 5 |
|---|---|---|---|
| DIST 40 (close-up) | 5.4k | 0.3 to 0.6 ms | 3.4 to 4.2 ms |
| DIST 120 (typical play) | 27k | 1.4 ms | 9.3 to 9.9 ms |
| DIST 280 (default) | 81k | 3.2 ms | 27.7 to 28.4 ms |

## Reading

- Cost is linear in triangles: about 15M triangles per millisecond on the 3090, plus about 2.8 ms fixed. Close-ups run nearer 7M per ms because triangles cover real pixels there.
- Without LOD, authored models do not fit. A 2k-tri soldier costs about 27 ms per frame at 200k on a 3090. An iGPU is roughly 8 to 10 times slower.
- With LOD, authored models are close to free. The close-up view draws only 5k soldiers and the typical view 27k. Only those need detail.
- Sketch budget: LOD0 2 to 3k tris inside about 30 m, LOD1 600 to 800 to 100 m, LOD2 150 to 250 to 300 m, LOD3 40 to 60 beyond. Estimated unit pass under 2 ms in every view above.
- The wide shot gets CHEAPER than today: 197k x 50 tris = 10M against 41M now. That is the view that hurts iGPU laptops (devlog 0071), so LOD helps the itch audience too.
- LOD selection fits the existing CPU sync pass. It already visits every unit for culling. Buckets go from kind to kind x LOD, so 4 draws become 12 to 16.
- CPU side is unchanged by mesh density. At 200k it is already the larger cost (sync 5 to 6 ms per frame).

## What UEBS does (web research, developer statements)

- UEBS 2 runs rendering, culling, animation, transforms and AI on the GPU. It uses "full GPU bone animations" with hundreds of clips per character.
- Community mod guides report aggressive distance LOD and skinned meshes throughout. Triangle counts, LOD counts and texture sizes are not published.
- The game is built around 1M units on a 2080 Ti. Its scale comes from a GPU sim. FLANKS sims real melee on the CPU, so 200k with better visuals is the honest target, not millions.

## Gap to UEBS, ranked by visual payoff

1. Authored, textured models with an LOD chain. Atlas per kind with a team-mask channel, which replaces the vertex-color alpha trick.
2. Shadows and grounding. Units need a shadow pass of their own (they draw through a custom pipeline), lowest LOD, near cascade only. A blob-shadow quad per soldier is the cheap first step.
3. Real skeletal motion. The rigid part rig is the one-bone-per-vertex case of GPU skinning. Baking bone matrices per frame into a texture is about 1 MB per rig and serves every LOD. The sim signals map cleanly: lunge progress becomes attack clip time, the shared march phase becomes walk clip time, wall and stance become blend weights.
4. Per-soldier variation from the existing seed: tint, atlas row (heads, shield designs), scale.
5. Atmosphere: fog, tonemapping, per-pixel lighting on near LODs.

Vertex animation textures were considered and rejected as the main route. Memory grows with verts x frames x LODs (about 14 MB per kind per LOD for 20 s). Layering a swing over a walk needs two full samples. Bone textures avoid both. Bevy meshlets do not apply (no custom vertex shader). No Bevy crate does instanced bone-texture skinning, so it is a port into the existing shader.

## Proposed order (nothing built yet, owner decides)

1. Asset path: load glTF meshes into the existing buckets, joint index per vertex replaces the UV part id. The shader still computes the part rotations procedurally. Validate with a CC0 pack (Quaternius or KayKit knight) before any generated asset exists.
2. LOD buckets in the sync pass.
3. Texture atlas, team mask, per-pixel light on LOD0 and LOD1.
4. Shadows.
5. Bone-texture skinning with real clips, if the rigid rig still reads as toy soldiers up close.

Steps 1 and 2 are engine work and do not wait for art. The Astra experiment can run in parallel against a written asset spec.

## Asset generation route (GPT-6 Astra through Codex and Blender MCP)

- Feasible for this art. Astra writes Blender Python, it does not emit geometry. Reports agree it is strong at hard-surface and props and weak at organic topology, skin weights and animation polish.
- Armored low-poly soldiers are mostly hard-surface. A rigid bind of one bone per part removes weight painting, which is the step agents fail at.
- Take motion from CC0 clip libraries (Quaternius Universal Animation Library, KayKit) by modeling onto their skeleton. Do not ask the agent to animate.
- Local state: Blender snap is 2.93.18, too old. Snap `latest/stable` is 5.2.2, classic confinement, so the MCP socket is not sandboxed. Codex is not installed. uvx and Node 20 are present.
- Licensing for a public repo: CC0 packs are fine to commit. Mixamo files may ship in a game but may not be redistributed raw, so never commit them.

## Addendum: what stops 1M soldiers (2026-09-20, analysis, estimates marked)

Owner question: which bottlenecks keep the game from 1M units, and is Bevy the limit?

Cost structure at 197k on the dev box (3090 + 3900X), from today's logs and devlogs 0016 / 0047 / 0050:

- Per frame, CPU: `sync_instance_data` 5.4 to 5.9 ms (already parallel). Then four passes over 12.6 MB of instance data: chunk concat, extract memcpy (serial, at the main/render sync point), `write_buffer` staging copy, GPU copy.
- Per frame, GPU: unit pass 5.5 ms in the draw-everything view.
- Per tick, worker thread, 33 ms budget: grid 3.2 + step 6.3 + field 0.95 + audit 1.5, about 12 ms. Dense melee ticks cost more. Over budget means the battle runs in slow motion, frames do not hitch (0050).
- Sim state is 91 B per soldier. Memory is not a constraint at 1M.

GPU per-soldier floor (hypothesis, two data points): today's linear fit leaves 2.8 ms at 197k that does not scale with triangles. The cube era (12 tris) cost 1.3 to 1.5 ms at 130k drawn. Both give about 10 to 14 ns per instance on the 3090. Instanced draws of tiny meshes carry per-instance overhead. LOD does not remove it. Not isolated yet.

Bevy verdict: not the blocker. Soldiers are rows in SoA arrays drawn by a custom pipeline, not entities, so ECS, transforms, visibility and batching do not scale with soldier count. The render app exposes wgpu compute, storage buffers and indirect draws, which is everything GPU-driven rendering needs. Real lower-level limits: no portable mesh shaders or Nanite-style software raster in wgpu (matters only for multi-million sub-pixel triangles), `write_buffer` adds a staging copy, extract is a sync point, and custom render code breaks on each Bevy release. The one place Bevy's standard path shows in a profile is the far-zoom render-world burst (14 ms against 2.5 ms, devlog 0047), which scales with regiments (banner entities, vegetation, terrain chunks), not soldiers.

Perf items, rough gains on the dev box:

1. Distance LOD. Wide-view unit pass 5.5 to about 3.5 ms. Little fps change on the 3090 (CPU-limited), about 2x wide-shot headroom on iGPUs. Prerequisite for authored models. Small.
2. GPU-driven instance build: upload compact tick state at 30 Hz, compute shader interpolates, smooths anim signals, culls, picks LOD, writes indirect draw args. Removes the sync pass and the per-frame 64 B per soldier upload, about 7 to 9 ms CPU per frame at 200k. Render-side ceiling about 250k to about 1M (with 1 and 3). Large.
3. Far soldiers as impostor quads in one non-instanced draw (vertex pulling). Attacks the per-soldier floor. Far soldiers about 5x cheaper. Needed above about 500k in wide shots. Medium.
4. Slim instance data 64 to 32 B plus zero-copy extract. Minus 1 to 2 ms per frame at 200k. Stopgap, superseded by 2. Small.
5. Banners and other per-regiment meshes into the instanced path. Removes the far-zoom 25 to 29 ms hitch class. No unit-count gain. Needs a fresh trace first. Small to medium.
6. Sim: SIMD separation kernel (deferred in 0016) plus melee candidate reduction (0048). Tick about 1.5 to 2x faster. Sim ceiling about 400k to about 600k. Medium.
7. Sim: skip pairwise work for formed regiments far from any enemy. 2 to 5x in typical battles, nothing in a full-field melee. Touches the simulate-never-fake rule. Owner design call.
8. Sim on the GPU, the UEBS route. 10x or more. Costs FL_HASH bit-determinism and makes audio, morale and UI readback asynchronous. An architecture swap, not an optimization.

Ladder (estimates): today about 200k to 250k at 60 fps. With 1 to 3 the render side reaches about 1M and the CPU sim becomes the wall near 400k to 500k. With 6, about 600k. 1M needs 7, a newer many-core CPU, or 8.

## CORRECTION (same day, after reading devlogs 0020, 0049, 0050 and the source)

Three statements in the addendum above are wrong. The full plan with the fixes is docs/plans/010-scale-to-1m.md.

- The sim does NOT run on a worker thread on `main`. The pipelined tick (d64e6d4), the catch-up clamp, the hitch attribution and `FL_HASH` exist only on the shelved `perf/spikes` branch. `step_sim` runs inline in `FixedUpdate`, inside the frame.
- So an over-budget tick DOES stretch frames on `main`. It does not turn into slow motion.
- Item 6 as written (SIMD separation kernel plus melee candidate reduction) was already tried here and failed. SIMD measured slower (devlog 0020, note in `spatial.rs`). The candidate prune measured zero and was reverted (devlog 0049). Per-soldier fixed work dominates the tick. The rewritten item 6 targets that work.
- New facts: the map is 1024 x 768 m and cannot hold more than about 360k soldiers, so 1M needs a bigger battlefield. The per-instance GPU floor was confirmed in devlog 0077.
