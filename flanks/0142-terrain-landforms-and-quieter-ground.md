# Quieter grassland surfaces and new rolling landforms
Written by GPT-6 Astra.

Date: 2026-09-26. Worktree: `/home/gota/ggando/gamedev/flanks-gfx`. Branch: `feat/terrain-look`. Commit: `fec2ff6`, Reshape the grassland and reduce ground patterns.

## Diagnosis and scope

Continues devlog 0141. Gota asked to improve the mottled surface and distant curved bands, with M2TW screenshots under `resources/map/m2tw_map_ss/` as reference. Inspected resized Spanish Plain, Township and Pavia references. Their broad green/dry areas are useful guidance; no reference pixels were copied into game assets.

At an oblique camera approximating debug4/5, the old heightfield still showed curved creases with constant ground color and no texture normals. These bands were real slope breaks in the old ridged-noise and radial flattening formula, not solely texture filtering. Coverage-only and plain-shaded captures are saved. The camera is a reproduction of the general angle, not an exact match to the annotated screenshot.

Gota explicitly authorized height generation changes during this work: “You can certainly fix the height map generation, I also don't like the current height map”. This iteration therefore includes a simulation-affecting classic heightfield replacement. The experimental river height formula is unchanged. Battlefield dimensions remain 1,024 by 768 m with 2 m cells; enlargement and deployment coordination are still separate work.

## Implementation

The classic map has four broad asymmetric rises around an open central lowland, a shallow curving valley, and small residual height variation. Smooth analytic falloffs replace ridge cusps and abrupt radial flattening. The sampled interior height range is 2.56 to 23.10 m, maximum grade 8.79%, and maximum adjacent-cell second difference 0.0094 m. This makes long gentle slopes instead of the previous small rippled hills.

Removed the enlarged 34 m grass layer responsible for much of the felt-like surface. One 10 m grass sample scale supplies fine texture, while the coverage image supplies broad green/dry/soil distribution. Reduced coverage noise contrast and tied exposed earth more closely to dry regions. The relief stencil spans 96 m in each direction so local slope noise contributes less.

Removed the separate color-distance fade to flat constants; mip filtering now controls visible color detail. Normal detail still fades from 50 to 220 m. Normals were already present; stronger lighting or normals were not the remedy used here.

The first revised capture exposed repeated rows in the previously tiled soil texture. Extended rotated, variance-corrected sampling to soil and stony ground, using each source's linear color mean to keep distant color stable. Skip stone sampling where its blend weight is zero. Fine grass detail slightly breaks soil transition edges. No texture binaries or asset licenses changed; updated the terrain asset description for the single grass scale.

## Verification

- Opt-dev build and strict clippy across all targets passed, eight jobs and offline.
- All three terrain tests passed. Added a heightfield test for finite heights, useful relief, gentle grades and absence of abrupt cell ridges; existing mesh/crater tests still pass.
- Diff review confirms changes only in `src/terrain.rs`, `assets/shaders/terrain.wgsl`, and `assets/terrain/LICENSE.md`. Existing Terrain simulation methods, terrain dimensions, shared random functions and the river height formula are unchanged. Classic heights intentionally differ.
- Recorded fixed close/gameplay/wide/oblique views, a lower-angle landscape, 200,000 soldiers, a 12-second continuous zoom/orbit and crater deformation. Inspected consecutive sampled motion frames at distant and close transitions, and consecutive crater frames; this was not exhaustive playback of every frame. Ground detail remained attached and no sharp distance transition appeared in these samples.
- Runtime logs contain no shader validation errors or panics. Ten crater rebuilds touched one to four chunks and took 0.07 to 0.30 ms; these are observations, not a controlled benchmark.
- All builds and games were gated through `work/scripts/flanks-run.sh` in the host process namespace. Waited when another game was present. Stopped only processes started for this work.

## Simulation hash

Before and after height changes, ran the same 40-second classic scenario using `FL_TREE=/home/gota/ggando/gamedev/flanks-gfx`, `FL_TEST_FRONT=1`, `FL_UNITS=20000`, `FL_CAM_LOCK=1`, `FL_AI=0`, and `FL_THREADS=8`. Names: `terrain-v3-before` and `terrain-v3-after`. Each produced 19 fingerprints, without runtime errors.

Zero of 19 sampled ticks match. The earliest difference is tick 60. This is expected from changed ground heights and is an intentional simulation change requiring review when merging. The helper `hashcmp.sh` sorts tick strings lexicographically and reports tick 1020 first; an independent numeric sort gives tick 60. The shared helper was not edited.

Final sample tick 1140: before `n=34857`, hash `0b6797a77f530a1e`; after `n=34679`, hash `07c918c73c5a0d2b`.

## Short GPU observations

Three bounded runs used 200,000 stationary soldiers, with GPU clocks/load recorded every 500 ms and CPU load recorded before/after. Dropped the first five seconds and kept five opaque-pass samples per view. Median opaque-pass time: close 0.54 ms at 1,170–1,230 MHz; gameplay 0.41 ms at 1,935 MHz; wide 0.37 ms at 1,935 MHz. These are whole opaque-pass timings, not terrain-only cost. No matched plain-material or v2 run was made in this batch, so this is not a controlled performance comparison. Capture FPS overlays are not benchmark results.

## Artifacts and next review

- Before/after slider, motion/crater recordings, 200k view and plain-shading diagnosis: `tmp/shots/terrain-surface-v3/index.html`.
- Raw logs, hashes, clock CSVs, measurement JSON and helper scripts: `tmp/runs/terrain-surface-v3/`.
- Handoff: `work/handoffs/HANDOFF-terrain-v3-2026-09-26.md`.

The surface is deliberately calmer and the hills much broader. Some broad color variation remains by design. The finite map edge and sparse environment are still visible; this is a terrain shape/material iteration rather than a finished environment. Review the new landforms and quieter ground in motion before adding more detail or expanding the battlefield. The river experiment still needs a separate design pass.

The commit is local, with no push or PR. The shared devlogs symlink is intact; logs remain uncommitted and are backed up to `/data/ggando/flanks/devlogs/` at handoff.

## Review feedback after v3

Gota likes the new heightmap. Distant terrain now looks too dull and needs more sharpness. At close range, debug6.png still shows stain-like clumps and relief; Gota dislikes the current texture and wants a grassier surface. Inspected a resized debug6: the foreground still reads as flattened, patchy ground rather than distinct grass. The screenshot alone does not isolate how much comes from color versus normal detail.

The pasture source is Poly Haven Grass Ground (`https://polyhaven.com/a/grass_ground`), downloaded as 1K diffuse, OpenGL normal and roughness maps. Exact URLs are in `assets/terrain/sources.json`. Current pasture color is filtered and recolored, so the source preview is not the final rendered appearance. Next material iteration should retain the accepted landforms, evaluate a grassier source with its normal map at an appropriate scale, and restore more definition at distance without restoring the enlarged mottled layer. No source changes were made during this feedback review.

For continued work on this topic, update this entry instead of creating a new devlog each iteration.

## Grass replacement shortlist

Compared downloaded 1K color maps from ambientCG Grass001, Grass003, Grass004 and Grass005 against Poly Haven Grass Ground, Leafy Grass and Withered Grass. Preview CDN requests returned 403, so inspected actual color maps from the public texture downloads instead. Samples and a labeled contact sheet are in `tmp/shots/terrain-grass-candidates/`. The sheet shows original color with each full tile resized to the same image dimensions, not equal real-world scale or an in-game render.

First candidate: [ambientCG Grass004](https://ambientcg.com/view?id=Grass004). Dense green and dry blades, relatively even coverage, without the broad bare-soil patches in the current source. Best initial candidate for this mixed green/dry battlefield. Second: [Grass005](https://ambientcg.com/view?id=Grass005), cleaner and brighter short grass, with a possible maintained-lawn appearance. Third: [Grass003](https://ambientcg.com/view?id=Grass003), darker tangled grass for a lusher direction. These rankings are visual judgments from source maps, not results in the game.

Leafy Grass retains conspicuous leaves, twigs and exposed ground; Withered Grass has larger dead-grass clumps. Neither is the preferred base for the current feedback. Grass001 also has more dark clumping than the preferred candidates. ambientCG confirms CC0 covers assets and preview renders: https://docs.ambientcg.com/license/ .

The current shader maps a grass tile across 10 m. Grass004 and Grass003 specify approximately 1.4 m square. A replacement must be evaluated at its physical scale, with restrained blade-scale normals, rather than inherit the 10 m mapping. More distant definition will also need material coverage contrast/edges to be reconsidered; changing a fine texture alone does not establish that result. No game assets or code were changed, and no build or game was launched for this research.

## Grass 004 trial and physical scale comparison

Gota asked to try Grass 004 against the previous texture, then decide which to develop further. Gota also asked for the previous physical size and delegated choosing reasonable, unstretched scales.

Verified Poly Haven Grass Ground metadata at `https://api.polyhaven.com/info/grass_ground`: dimensions `[2509.999990463257, 2509.999990463257]`. The official Public-API swagger defines texture dimensions in millimetres, so the source covers 2.51 by 2.51 m. Its downloaded maps are 1,024 by 1,024 pixels. The 10 m tile used in v3 enlarged features by approximately 3.98 times in each direction. Grass004 specifies approximately 1.4 by 1.4 m and was also downloaded at 1K. Physical coverage and pixel resolution are separate quantities.

Compared three configurations at matching cameras:

| Configuration | Tile size | Grass normal strength |
|---|---|---|
| Previous appearance, commit fec2ff6 | Poly Haven, 10 m | 0.32 |
| Poly Haven at source scale | 2.51 m | 0.18 |
| Grass 004 trial | 1.4 m | 0.18 |

The two native-scale versions use identical normal strength, color palette, coverage, soil, lighting and the same neutral-detail preparation. The original 10 m baseline retains its original 0.32 normal strength, so that comparison changes both scale and normal strength. Both sources are processed into neutral detail rather than displayed as their original green/brown color images.

Chosen parameters: keep Grass004 at 1.4 m and grass normal strength 0.18. At the closest camera it supplies denser, more even fine detail; Poly Haven at 2.51 m is already substantially less blotchy than at 10 m but retains smaller clumps. At battle distance the distinction becomes modest because the fine texture is filtered. Native-scale material does not resolve the separate request for stronger distant landscape definition. Do not enlarge the blades to force them to remain visible at that distance. Visual preference between the sources remains for Gota to decide.

Committed the Grass004 trial as `064e0fc`, Use Grass 004 at natural scale. Only six asset/shader/preparation files changed; no Rust, heights, simulation, camera, coverage or lighting changes. Grass004 replaces the two pasture KTX2 files at the existing paths. Source JSON records the CC0 asset page, physical size, ZIP URL, archive size/hash and extracted map sizes/hashes. Preparation supports `--layers pasture`, with the source JPEGs extracted into the downloads directory. Updated the outdated filtering comment without changing the filter algorithm. GPU storage layout and texture resolutions/mip counts remain the same; disk sizes become 1,501,298 bytes for color and 5,330,392 bytes for normal/roughness.

Opt-dev build and strict all-target clippy passed. Both new KTX2 files passed Khronos validation with warnings as errors; Python compilation and git diff checks passed. Runtime capture logs had zero shader errors or panics. Captured closest gameplay zoom (15 m distance at low angle), low/high angle 40 m views, battle camera 280 m and wide 900 m for all three versions. Recorded 12-second continuous zoom/orbit clips for both native-scale materials and inspected consecutive sampled frames around the close transition; no abrupt detail transition was apparent in those samples. No exhaustive frame-by-frame inspection or performance benchmark is claimed. FPS varied with desktop/background state, so overlays should not be used to rank performance. All game launches used the shared wrapper and only our own games were stopped. Render-only diff means no new simulation hash run was required under AGENTS.md.

Review: `tmp/shots/terrain-grass004-trial/index.html`, with selectable left/right materials and a comparison slider. Logs, downloaded source maps/archive, previous/native shader snapshots, previous/new packed textures and capture helper: `tmp/runs/terrain-grass004-trial/`. The preceding Poly Haven implementation remains in commit fec2ff6, and its packed files are also preserved in the comparison run folder. Grass004 is active in the working tree; user acceptance is pending. No push or PR.

## Select Poly Haven and improve distant definition

Gota reviewed the Grass004 comparison and prefers the previous Poly Haven material. Grass004 was a trial, not the selected direction: its dense blade source at 1.4 m gave finer, more uniform foreground detail, but did not improve the broad distant appearance. The completed trial, scale comparison, source downloads, parameters and validation are documented above and preserved in commit 064e0fc and `tmp/shots/terrain-grass004-trial/`.

This entry is updated before changing the implementation, as requested. The selected material is Poly Haven Grass Ground. Retain the corrected 2.51 m tile size and restrained 0.18 grass normals from the comparison, following the instruction to avoid stretching either texture. Keep the accepted heightfield. The next change targets the overly diffuse green/dry/earth coverage visible at battle distance: make the regions more distinct and their transitions more defined, without bringing back the enlarged mottled turf layer. Exact changes, captures and validation will be appended after implementation.

The first distant-definition candidate remapped broad dryness through smoothstep(0.36, 0.70) and concentrated soil with smoothstep(0.62, 0.77). Gota rejected it immediately: the amplified warped bands resembled Jupiter. This candidate is discarded. Its stronger broad color contrast was the wrong direction, despite preserving the accepted heights. The original quiet coverage formula is restored. Next check is confined to texture filtering: increase anisotropy from 8 to 16 and test a restrained half-mip color bias, leaving normal-map filtering and coverage unchanged. This targets resolved surface detail rather than amplifying broad noise. Candidate captures remain in `/tmp/flanks-terrain-definition/`; they are not the selected result.

Gota rejected the filtering follow-up too: the whole appearance was unlike grassland_map_ref.png and the M2TW landscapes, not merely too soft. Revisited the actual reference and all three M2TW contact sheets. The current surface replaced source color and most variation with smooth, warped procedural coverage. Increasing that coverage's contrast made cloud bands; filtering changes could not correct its appearance. Both sharpening attempts are discarded.

Gota suggested Poly Haven Grass Path 2 (https://polyhaven.com/a/grass_path_2), which is already downloaded as the stony layer. It is a 1 m-wide dirt/pebble/grass-tuft scan. Next comparison retains original color maps at native scale, disables procedural coverage color/soil patches, and compares Grass Ground against Grass Path 2 with identical heights, lighting and weak normals. This is a material proof for review, not a claim that a texture-only change reproduces the complete reference environment. The distant mountains, vegetation, cast shadows and atmosphere in the reference remain separate missing context.

### Material study results and retained state

Tested original-color Grass Ground at 2.51 m against original-color Grass Path 2 at 1 m, using the same rotated sampling, grass normal strength 0.18, accepted heightfield and lighting. In both studies the shader bypassed procedural color/soil coverage and used source scan color directly; crater/steep-bank weights remained. Grass Ground packing temporarily skipped neutral grayscale preparation. This isolated what the source materials actually contribute.

Grass Path 2 shows clearer soil, pebbles and small tufts at close range. Covering the full battlefield with it makes a pale, dry field, not the requested meadow. At battle distance its fine grain is more apparent than the filtered pasture, but there is still a fairly uniform, grainy surface. It is a plausible exposed-ground/worn-area material, not an accepted full grassland replacement. Raw Grass Ground restores green/brown color variation close up but also becomes a largely uniform brown field at distance. These are diagnostic results, not a solved landscape.

Re-examining grassland_map_ref.png and the Spanish Plain, Township and Pavia images shows that the target depends on recognizable vegetation and soil regions, broken local boundaries, grass/rock/shrub forms and spatially varied lighting. The current grayscale-detail-plus-warped-color approach does not reproduce that material structure. Increasing the warped field's contrast was specifically rejected. A texture swap alone also did not reproduce the reference. Further work should establish a small reference-matched terrain area with deliberate grass/exposed-ground placement before spreading another procedural treatment over the whole map. Keep the accepted landforms.

Review page: `tmp/shots/terrain-natural-ground-study/index.html`. Original-color shader and texture snapshots, capture helper and logs: `tmp/runs/terrain-natural-ground-study/`. Captured near, battle and low landscape views for both sources, plus a 12-second Grass Path 2 zoom/orbit recording. Inspected consecutive sampled frames; no exhaustive motion or performance claim. Logs had zero runtime/shader errors. These studies remain separate and are not active game materials.

Retained implementation: commit `0e526cb`, Restore Poly Haven pasture at natural scale. Returns to the selected Poly Haven material at 2.51 m with 0.18 grass normals, original quiet coverage and original 8× anisotropic filtering. Neither the rejected coverage remap nor half-level mip bias remains. The original-color studies are also removed from the active shader/packing pipeline. Final diff contains only the pasture shader selection/scale, source/license records and packed assets; no Rust, terrain heights or simulation changes. Build and strict all-target clippy passed. Restored packed maps match fec2ff6 byte-for-byte and pass KTX2 validation. No hash run was needed for this render-only restoration. The distant grassland appearance remains unresolved and must not be described as fixed.

No game remains running from these comparisons. No push or PR. This updates the existing devlog rather than adding a new topic entry.

## Select original-color Grass Ground and propose distant ground layout

Gota prefers Grass Ground in the original-color material study and rejects Grass Path 2. Drop Grass Path 2 from the proposed grassland palette. This records the selected direction; no active textures or shader were changed during this proposal. The retained game implementation is still 0e526cb, and the original-color study remains a separate snapshot.

Re-inspected the study battle view, grassland_map_ref.png and the Spanish Plain screenshots. The study retains acceptable close detail but becomes an almost uniform ochre surface at battle distance. The reference surfaces retain grass and exposed-earth shapes across several scales, with rocks, bushes and shadows supplying additional depth and size cues. My diagnosis is that the distant ground is missing a deliberately composed material layout. Repeating a 2.51 m scan supplies fine detail; the previous broad warped noise supplied conspicuous clouds when amplified. Neither establishes the desired landscape by itself.

Proposed next stage: author a unique world-space ground layout, starting with a roughly 256 by 256 m representative area on the accepted heightfield. Use original-color Grass Ground at 2.51 m with restrained normals for close detail. Author masks for connected pasture, dry grass and occasional exposed earth, with unequal region sizes, gaps and broken local edges. Starting feature sizes are roughly 2 to 15 m within larger pasture areas, to be judged in the battle camera rather than treated as fixed correct values. Keep open grassy areas dominant. Avoid repeating blobs, warped bands and full-field random mottling. A mask decides where a material belongs; the material supplies its surface variation. Preserve this layout across viewing distances, allowing only the fine scan detail to filter away. Painted terrain layers are a standard implementation pattern; Unity documents the separation of terrain layer materials and painted coverage at https://docs.unity.com/en-us/engine/6000.5/manual/creating-environments/script-terrain/terrain-textures/class-terrain-layer . This is not a claim about M2TW internals.

First review should show the ground alone under unchanged lighting at near, battle and low landscape cameras, plus a continuous camera move. Assess distinct grass/earth structure, absence of cloud bands or obvious stamps, and close detail remaining at physical scale. The first useful proof is the battle view. Do not expand another unaccepted surface treatment across the whole battlefield. No height, simulation or river changes are proposed.

Sparse rocks, shrubs and grass clumps are a subsequent environment stage, using the same placement layout and preserving open deployment space. Shadows and distant atmosphere are later separate work; source inspection confirms directional shadows are currently disabled. They contribute to the reference, but should not be used to disguise an unresolved ground surface. No builds, games or source edits were needed for this proposal. Continue updating this entry for the topic.

## Author the distant ground surface, implementation start

Gota authorized steps 1 and 2. Bare ground must be judged under existing lighting, with no vegetation, scenery or shadows hiding an unresolved material. Keep accepted heights, simulation and river experiment intact. Replace the classic map's warped procedural color with authored non-repeating ground-color artwork and retain original-color Grass Ground at physical scale for close detail. Use the built-in image generation skill for the new project-created raster, then inspect and integrate it as an albedo asset. No reference screenshot pixels are used as textures. Review a representative central area in-engine before treating the approach as accepted. Grass Path 2 is excluded from the classic ground treatment; the separate river treatment remains outside this iteration.

Record source artwork, preparation and prompt alongside the material so the packed asset is reproducible from its saved source. Compare the bare surface to the retained baseline at matching cameras and inspect a camera move. Build and strict clippy checks follow only when the shared run guard reports no game. This is rendering-only: no heights or shared simulation random sources are to change.

### Authored ground trial result

Local commit `e0b06c7`, Add a unique grassland color layout, is active on feat/terrain-look. This is a visual candidate for Gota's review, not an accepted fix. The classic map now uses a unique overhead meadow albedo, with connected greener pasture and a drier eastern flank. Native-scale original-color Grass Ground is normalized by its linear mean and multiplied into the layout, so its subpixel detail filters toward one instead of erasing the larger surface arrangement. No grayscale preparation is used by the classic pasture. Grass remains 2.51 m per tile with normal strength 0.18. No new broad normal relief or negative mip bias. The alpha channel selects close grass/soil detail from the artwork's pale warm openings; it does not change the distant color. Existing crater exposure still overrides the surface with earth.

Used the built-in imagegen tool, not the CLI, for original project artwork. The first generated candidate failed the in-engine scale check: its marks resembled enlarged grass clumps. It is discarded and archived in the run folder. The second prompt explicitly described a kilometre-wide aerial survey with quiet continuous grass and a dry flank. Saved source: assets/terrain/grassland_layout.png, 1448 by 1086 pixels, despite requesting a larger image in the first prompt. The exact selected prompt is assets/terrain/grassland_layout.prompt.txt; dimensions, source checksum and 1024 by 768 m footprint are in grassland_layout.source.json. This artwork is generated for the project, not copied from M2TW or the mood reference. The single map spans the battlefield to avoid a rectangular trial boundary; review initially focused on the central battle area and the nearby dry flank.

prepare_layout.py validates source checksum/size, derives the detail mask and packs an sRGB KTX2 with linear alpha and a complete mip chain. Repacking reproduces identical bytes. Packed layout size: 4,776,885 bytes; additional original-color pasture: 2,577,946 bytes. The river keeps its previous neutral pasture, procedural coverage, stony layer and bank behavior. The classic material no longer loads or samples Grass Path 2. It uses the earth handles for the unused stone bindings so the material layout remains compatible with the river.

The comparison now retains a visibly greener central pasture and a distinct dry flank at battle distance. This is my observation, not a claim that Gota will find the surface pleasant or that it matches the reference. Bare terrain still exposes the sparse scene, smooth hill silhouettes and finite map edge. No vegetation, lighting, shadows, fog, height generation, unit rendering or simulation changes are included.

Review: tmp/shots/terrain-authored-ground/index.html. It opens at battle distance with a slider between the previously selected original-color Grass Ground study and the new layout; matching near and low landscape views, a continuous 12-second zoom/orbit and a crater recording are included. Artwork and exact prompt are linked there. Logs, helper scripts, rejected first artwork/captures, timing records and consecutive-frame contact sheets are in tmp/runs/terrain-authored-ground/. Inspected the continuous camera sequence as 24 consecutive half-second samples and the crater sequence as 12 one-second samples. No abrupt layout replacement or chunk seam was apparent in those samples; this is not an exhaustive temporal-aliasing inspection. The shader uses world-space mapping and mip filtering without a camera-distance switch in color.

Validation: opt-dev build and strict all-target clippy passed. Both new KTX2 assets pass validation with warnings treated as errors. Python compilation, source checksums, deterministic repacking and git diff checks passed. Fourteen capture/measurement logs contain zero shader/runtime error lines or panics. The river material smoke check rendered successfully. Crater recordings show exposed earth as the surface deforms; rebuild logs report 1 to 4 chunks in 0.07 to 0.23 ms. Diff inspection confirms rendering-only Rust changes in material bindings, texture loading and material creation; no terrain height or simulation changes and no new FL switch, so no new sim hash run was required.

Six bounded 14-second runs compared the previous raw Grass Ground study with the layout using the same binary, 200,000 stationary units and three cameras. Shader snapshots were restored after each run and final source equality was verified. Recorded GPU clocks/utilization every 500 ms and CPU load averages before/after; dropped the first five seconds and retained five opaque-pass samples per run. These are whole opaque-pass timings, not isolated terrain costs:

| Camera | Previous median | Layout median | Previous GPU clocks | Layout GPU clocks | Previous 1-minute CPU load before/after | Layout 1-minute CPU load before/after |
|---|---|---|---|---|---|---|
| 40 m | 0.81 ms | 0.54 ms | 630–1800 MHz | 1035–1245 MHz | 1.86 / 2.55 | 2.55 / 3.76 |
| 280 m | 0.40 ms | 0.42 ms | 1935–1950 MHz | 1935 MHz | 2.01 / 2.34 | 2.34 / 2.30 |
| 900 m | 0.36 ms | 0.39 ms | 1935 MHz | 1845–1935 MHz | 3.34 / 4.21 | 4.21 / 4.89 |

Battle-view cost is approximately 0.02 ms higher at comparable clocks. Close-camera clocks differ too much to infer a speed change, and no matched full-clock close-camera budget result is claimed. The wide-view clock difference also limits interpretation of the 0.03 ms difference. Do not use screenshot FPS overlays as performance comparisons.

No game remains running from this work. No push or PR. Devlogs remain uncommitted. This updates the same topic entry as requested.

## Accept the generated layout and record the generation recipe

Gota approves the current appearance ("This looks great") and asks to preserve this version. It is already saved in local commit e0b06c7, Add a unique grassland color layout, on feat/terrain-look. The tracked working tree is clean, so no duplicate or empty commit is needed. Keep this as the accepted surface baseline. The remaining green/brown boundary concern below is a separate refinement; no renderer or artwork edits were made during this acceptance/documentation update.

Generation summary:

1. Inspected grassland_map_ref.png and the M2TW Spanish Plain screenshots for color, material placement and scale. Used them as visual guidance while writing the request; no reference pixels were copied into the game asset or supplied as an edit target.
2. Used the built-in imagegen tool to create original overhead ground-color artwork. The first image contained too many grass-like marks. Once mapped over the battlefield, these resembled oversized clumps, so that image was discarded.
3. The selected request explicitly described a kilometre-wide orthographic aerial view: continuous muted olive-green pasture across the middle/left, a drier straw-olive right flank, occasional pale earth, and locally broken boundaries. It excluded visible individual blades, objects, shadows, haze, roads and repeated mottling. That landscape-scale wording produced the accepted image. The exact request is committed in assets/terrain/grassland_layout.prompt.txt. The built-in tool produced 1448 by 1086 pixels. The original PNG is committed as grassland_layout.png; source metadata and checksum are in grassland_layout.source.json. Re-running generation is not expected to reproduce identical pixels; the saved PNG is the reproducible source of the game asset.
4. prepare_layout.py checks that PNG against its source metadata, preserves its RGB color and derives an alpha mask from pale warm areas to choose close soil detail. It packs grassland_layout_color.ktx2 with sRGB color, linear alpha and a full mip chain. Repacking the saved PNG has been verified byte-identical. Command: python3 assets/terrain/prepare_layout.py --toktx /path/to/toktx . NumPy, Pillow and Khronos toktx are required.
5. The shader maps the artwork once over the 1024 by 768 m battlefield. Original-color Poly Haven Grass Ground repeats independently at 2.51 m with 0.18 normal strength. Its color is divided by the scan's linear mean and multiplied into the large ground-color map. At distance the fine scan averages toward one, leaving the authored ground layout visible. Brown Mud Dry supplies local soil detail and crater exposure. The alpha mask only chooses close detail; the green/brown boundary itself is already in RGB.
6. Checked matching battle, near and low-angle views, recorded a continuous camera move and crater updates, validated textures, built, ran strict clippy, and made bounded 200k-unit GPU observations. Detailed results are in the preceding section. No vegetation or lighting changes were used to obtain the accepted appearance.

### debug7 boundary feedback

Gota reports that the green/brown boundary can make the map read as a tiny patch of ground or a satellite image, creating a scale mismatch. Inspected resources/map/debug7.png after resizing from 1610 by 908 to 1440 by 812. The marked region has one long, locally ragged outline and detached green/brown islands. My visual diagnosis is that its coastline-like composition retains an aerial-image cue. The screenshot does not establish a regular 1 m grid artifact. The large color image has approximately 0.707 m per texel, but that sampling density alone does not explain the perceived scale. Fine grass detail does not alter the boundary at battle distance.

Proposed focused follow-up: preserve the accepted main regions and edit only a narrow transition corridor, adding uneven grass/earth interleaving and interruptions at a few metres in scale. Vary the transition width and reduce the isolated island/coastline shapes. Judge it against soldiers at the debug7 camera as well as close up. This requires changing the actual RGB boundary or representing it as material coverage; changing the existing alpha mask alone cannot fix the distant outline. Avoid blanket blur, map-wide noise, stronger normals or vegetation as substitutes. This remains a diagnosis and proposal, not a completed fix.

### Indexed screenshot folders

Renamed the eleven existing tmp/shots directories with four-digit indices, ordered by their pre-migration modification times. Unnumbered paths remain as relative symlinks so previous review links and helper scripts continue to work. New folders use the next unused index and no additional unnumbered alias. The current accepted review is tmp/shots/0011_terrain-authored-ground/index.html; the previous raw material study is 0010_terrain-natural-ground-study. tmp/shots/README.md records the convention and index. Next free index at this update: 0012; check the shared directory again before allocating it.

No game or build was needed for this documentation and file-organization update. No push or PR. Devlogs remain uncommitted and are backed up at session end.

## Refine the accepted green/dry boundary, implementation start

Gota asks for an intuitive explanation of the two-scale surface algorithm and authorizes the boundary refinement. The accepted baseline is e0b06c7. Its generated large-scale RGB artwork supplies persistent landscape structure, and normalized native-scale Grass Ground supplies close detail; the shader does not procedurally generate the artwork. The long irregular green/dry contour can read as either a coastline or a much smaller patch of turf. Refine the transition itself while preserving the green pasture and dry flank, accepted terrain heights, original grass scale and lighting. Use the built-in imagegen edit tool with the accepted artwork as the edit target. Compare the resulting material with soldiers visible at a wide camera similar to debug7, and at the landscape/close cameras. No vegetation or scenery additions.

Allocated indexed review folder tmp/shots/0012_terrain-boundary-refinement. Keep the accepted artwork and packed map available for matched before/after captures. Record the exact edit prompt and edited source if the candidate is retained. Update this entry with the result; do not create another devlog for this topic.

### Boundary refinement approach after inspection

Two built-in imagegen edits softened the outlined boundary but also introduced coarser texture marks outside the requested corridor. Neither generated image is used by the game. The accepted PNG and packed layout remain byte-for-byte unchanged. Exact requests and rejected outputs are archived with the refinement run evidence.

The retained candidate instead changes the material shader. Within the eastern meadow corridor, it estimates nearby grass/dry color and coverage using nine weighted samples spaced 16 m apart, filtered to an 8 m minimum footprint. It rebuilds the mixed region through intermediate grass/dry coverage. Small coverage interruptions use fixed 7 by 4 m and 2.5 m world-space scales, fading when unresolved; they are confined to mixed vegetation rather than applied across the map. A local luminance residual preserves the source surface grain. The first shader attempt omitted that residual and produced a visibly smooth strip; it was discarded. Outside the corridor or in unmixed grass/dry regions the original field is returned. All original source textures, terrain heights, native grass scale, normals, lighting and the river path are preserved.

The current result reduces the sharp color outline without a flat band. Some original land-cover shape remains visible, and whether its apparent scale is improved remains for visual review. Camera-matched captures begin with 20,000 soldiers. The wide camera approximates the debug7 framing, and closer views show the actual contact and greener side. Review: tmp/shots/0012_terrain-boundary-refinement/index.html. The exact edit prompts are linked there even though the final change is shader-only.

### Boundary refinement validation and saved state

Saved as local commit 2adeb91, Refine the grassland material transition, on feat/terrain-look. The accepted e0b06c7 baseline remains the preceding commit. Only assets/shaders/terrain.wgsl and assets/terrain/LICENSE.md change. The original PNG and packed ground-color texture were compared byte-for-byte with the accepted copies and remain identical. No Rust, heights, shared random sources, simulation logic, vegetation, lighting or FL switches changed. No simulation hash run is required for this shader-only change.

Review: tmp/shots/0012_terrain-boundary-refinement/index.html. Raw evidence and helpers: tmp/runs/0012_terrain-boundary-refinement/. The latter includes both rejected built-in imagegen outputs (rejected-edit-1.png and rejected-edit-2.png), their exact prompts, the discarded flat-blend shader, accepted/final shader snapshots, logs and GPU measurements. No generated edit replaced the accepted artwork. The final result is a controlled material-blending refinement, not a new image-generation result.

The wide camera uses distance 1050 m, pitch 0.9 and yaw 0.12, approximating debug7 rather than claiming an exact camera reconstruction. Boundary view: focus (260, 0), distance 220 m, pitch 0.65, yaw 0.35. The contact close-up uses focus (320, 0) and distance 25 m. All before/after captures use the same settings and begin with 20,000 soldiers. Combat can continue during capture, so unit state and counters may differ; this is not a simulation comparison. Captured and inspected a 12-second continuous zoom/orbit around the transition, including 24 consecutive half-second samples. No abrupt material replacement was apparent in those samples. This is not an exhaustive temporal-aliasing test. Both procedural interruption scales fade according to pixel footprint to avoid unresolved noise at very oblique/far views.

Opt-dev build and strict all-target clippy pass. Runtime WGSL compilation succeeds, and nineteen capture/measurement logs contain zero error or panic lines. Diff checks pass. Final shader restoration after each temporary baseline capture/measurement was verified. The pure river path does not call the new function; crater exposure still applies after material composition. No new crater or river simulation scenario was needed for this isolated shader change.

GPU observations use the same binary and 200,000 soldiers in six guarded 14-second runs. Camera focus is (320, 0), placing the close view on the transition. Drop the first five seconds, retain five opaque-pass samples per run, record clocks/utilization every 500 ms and CPU load before/after. These are whole opaque-pass timings, not isolated terrain cost. Clocks are not locked and differ between runs, so the table does not establish exact shader overhead or a matched full-clock close-view budget result.

| View/material | Median opaque pass | GPU clock range | 1-minute CPU load before/after |
|---|---|---|---|
| measure-280-accepted | 0.36 ms | 1845–1920 MHz | 1.53 / 2.11 |
| measure-280-layout | 0.45 ms | 1920–1920 MHz | 2.11 / 3.10 |
| measure-40-accepted | 0.62 ms | 1065–1200 MHz | 1.34 / 1.48 |
| measure-40-layout | 0.69 ms | 1185–1245 MHz | 1.48 / 2.06 |
| measure-900-accepted | 0.38 ms | 1560–1680 MHz | 2.17 / 2.33 |
| measure-900-layout | 0.45 ms | 1605–1725 MHz | 2.33 / 2.54 |

The battle-view observations are approximately 0.36 ms before and 0.45 ms after at 1845–1920 MHz and 1920 MHz respectively. The extra neighborhood sampling has a measurable cost. Source artwork dimensions, texture memory and native grass sampling are unchanged. No claim is made from screenshot FPS overlays.

Visual acceptance of this refinement is pending. Some original land-cover shape remains visible; the intended change is a less sharply outlined transition with preserved surface grain. The broad composition remains the accepted one. No game remains running from this work, and there is no push or PR. Devlogs remain uncommitted and are backed up at session end.

## Accept the refined texture baseline

Gota finds the refinement more natural and no longer coastline-like, and accepts the current texture as a working baseline that may be revisited later. Commit 2adeb91, Refine the grassland material transition, already contains the implementation and material documentation. The tracked working tree is clean. No duplicate commit or additional texture/shader edits are needed for this acceptance update. Current accepted review: tmp/shots/0012_terrain-boundary-refinement/index.html. This supersedes the earlier pending-review status and makes 2adeb91 the accepted baseline, replacing e0b06c7 for future comparisons.

Possible texture-level follow-ups are small and optional, not newly confirmed defects. First, use an explicit grass/dry-grass/exposed-soil mask when further material editing warrants it: the current close-detail mask is inferred from artwork colors, and the boundary shader also classifies nearby colors. An explicit mask would let the material detail correspond to the intended surface independently of color grading. Second, evaluate stability during slow movement at shallow camera angles before pursuing more sharpness; existing mipmaps and anisotropic filtering should be retained, and changes justified by an observed artifact. Third, consider precomputing the static neighborhood color/coverage estimates of the accepted boundary blend to reduce its extra texture reads. The pixel-footprint-dependent filtering and small coverage variation cannot simply be baked as one fixed screenshot, so require matching close/far/moving comparisons before substituting a cheaper implementation.

Recommendation: keep the visual baseline stable for now. Further texture work should address a visible close-material mismatch, motion artifact or measured cost, rather than add general noise, contrast or stronger normals. No such additional implementation is authorized by this request for suggestions. No game or build was run for this documentation update. Devlogs remain uncommitted and are backed up at session end.

## Texture follow-ups, responsibility and visual priority

Gota asks to explicitly record the three texture follow-ups and will have Claude handle the cost reduction later. Keep 2adeb91 as the accepted working baseline. No further texture implementation is started here.

| Follow-up | Purpose | Priority and next action |
|---|---|---|
| More deliberate grass/soil detail placement | Replace color-derived guesses with explicit grass, dry-grass and exposed-soil coverage so close textures and normals agree with the intended surface. | Deferred visual polish. No specific remaining mismatch has been demonstrated that warrants doing this before vegetation. Revisit if close shots reveal grass detail on bare soil, new materials need independent coverage, or color edits make the inferred mask unreliable. |
| Stability during slow camera movement | Check for crawling grain, shimmer or visible filtering transitions at shallow angles. Retain mipmaps and anisotropic filtering; adjust only when an artifact is observed. | Check during normal play and later visual reviews. No new defect is established and no separate broad filter rewrite is planned. |
| Reduce boundary-blend cost | Precompute the static neighborhood color/coverage estimates while preserving the accepted look and leaving view-dependent filtering/interruption behavior correct. | Claude will handle this later, as directed by Gota. Do not begin a competing optimization here. Compare against 2adeb91 with close, battle, wide and moving views, then measure at comparable GPU clocks. The observed 0.36 to 0.45 ms battle opaque-pass change is contextual evidence, not a promise that the full difference is recoverable. |

My recommendation is to prioritize a vegetation-placement pass over explicit grass/soil masks. The current bare surface has passed visual review. Better masks would mostly refine close contact/detail and improve authoring control; the references still differ more visibly through trees and shrubs, recognizable scale cues and the arrangement of open versus occupied ground. Start with a representative area containing deliberately grouped shrubs and a few tree groups, preserving broad deployment and maneuvering space. Avoid a uniform scatter across the entire field. Dense grass blades across the whole battlefield are not the proposed next step; any close grass would be a later, limited addition with its own cost check.

Vegetation is a separate scene-composition stage built on the accepted texture, not a remedy for an unaccepted ground surface. Placement, culling and draw cost still need validation against the 200k-unit target. This is a priority recommendation only; Gota has not requested vegetation implementation in this turn. No source edits, builds or games were needed. Devlogs remain uncommitted and are backed up at session end.
