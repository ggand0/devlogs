# First oak: authored model and grassland integration
Written by GPT-6 Astra.

2026-09-27. Graphics tree, branch `feat/vegetation`, based on main `6166f8b`. Review: `tmp/shots/0014_vegetation-oak-v1/index.html`. Asset iteration: `assets_dev/vegetation/oak_v1/`.

## Scope and decisions

Gota accepted the first-oak step after the design in devlog 0146. The current field remains the baseline despite the narrow scenery margins; he may ask Claude to make an experimental vegetation map later. He explicitly chose proceeding with an authored oak after discussing existing assets versus controlled Blender construction. This stage produces one species and one specimen, L0/L1/card, in the game at 40, 250 and 900 m. Wind, more variants/species, grove placement, the FL_VEG switch and the full population cost comparison are later stages.

Read the user rules, vegetation handoff, design note, unit asset spec, Blender setup, writing guidelines and current board. Inspected the current vegetation renderer, GLB image loader, Bevy 0.19 material/visibility code and Blender 5.2 export material code. No subagents. The main tree's source and Claude's claimed files were not edited. Main still lacks the shadow branch at this review, so the captures do not have cast tree shadows.

## Geometry and material

`build_oak.py` is the source of truth. It creates a tapered, slightly leaning trunk, eight irregular primary boughs, thirty terminal crown groups, and three leaf-spray atlas variants. The random seed is local to the offline Blender script. L0 has 1,352 wood triangles and 1,320 leaf-card triangles. L1 retains the trunk and primary boughs with 216 wood triangles and 420 leaf-card triangles, sampling the same crown centres with fewer larger cards. There is one variant root named `variant_0`.

Measured from the final GLB bytes:

| Mesh | Triangles | Exported vertices | Height | Width × depth |
|---|---:|---:|---:|---:|
| L0 | 2,672 | 5,224 | 12.120217 m | 11.919141 × 11.362256 m |
| L1 | 636 | 1,268 | 12.294296 m | 12.281463 × 11.490073 m |
| card | 4 | 8 | 14 m plane bounds | 14 × 14 m plane bounds |

The card dimensions include transparent padding. Both solid levels have minimum Y exactly zero; all node transforms are applied, the root is (0,0,0), and the export is +Y up. L1 differs in total height by 0.174 m. All vertices carry positions, normals, UV0, UV1 and VEC4 colour. Vertex alpha is 1 throughout. UV1 U stores bend strength and V stores 0 for wood or 1 for leaves; no wind runs yet. L0 has 2,640 foliage vertices and 2,584 wood vertices; L1 has 840 and 428. Unit part IDs and team colour do not apply to vegetation.

The built-in image_gen tool made one original transparent oak-leaf and bark atlas. It returned 1254 × 1254 RGBA despite the requested 2048-square size; the original was copied unchanged as `oak_source_atlas.png`, with its exact prompt, source path and provenance saved beside the build script. Three quadrants contain sprays and the fourth bark. No M2TW or external asset pixels enter the model. The near material has no separate normal atlas in this iteration; the authored mesh normals provide crown lighting.

The far cross is baked from two orthographic views of L0. It has a 1024 × 512 colour atlas and a 1024 × 512 tangent normal atlas. Colour is baked through emission to avoid baking a second sun; normal vectors are baked from the authored vertex normals into each plane's tangent frame. This gives live directional shading at distance. The two materials use alpha testing at 0.5, draw both faces, and retain the volume normals on card backfaces. All textures are embedded in the GLB, approximately 3.9 MB.

Technical corrections within this first iteration: flatten the first trunk ring onto the ground plane; align far baking and plane bounds with zero-height origin; preserve volume normals on backfaces; add far normal baking after a colour-only far image blended too strongly into the grass. The directional-light-only look still has a very dark crown lobe in the game. Treat its contrast as a visual-review point, not as resolved by the backface setting. The crown, bark detail and density have not been accepted by Gota.

## Runtime integration

Only `src/vegetation.rs` changes in the public tree. The loader checks the entire GLB before creating asset handles, validates the three levels, caps, attributes, alpha material and embedded textures, and builds indexed meshes. It looks first for `assets/vegetation/oak.glb`, then the private iteration file. Nothing was copied into the final assets folder pending review, and no new public binary was committed.

The oak stands at x = -495 m, z = -60 m, with the root sunk 0.08 m into the sampled terrain. Its crown remains inside the western 30 m scenery margin. Grassland has this one oak; Classic has no plants; River retains the existing procedural plants. Asset handles are cached while map entities respawn. The future populated version can merge these same indexed meshes into the existing terrain chunks; merging several plants is not exercised by one specimen.

Mipmaps average colour in linear light weighted by alpha, preserve cutout coverage, and include final rows/columns of non-power-of-two textures. Normal mips normalize the averaged vectors. Two regression tests exercise transparent-pixel colour bleed and the odd atlas edge case. No separate image crate or dependency was added.

Projected-height thresholds have hysteresis: L0 exits below 108 px and re-enters above 132 px; L1 exits to the cross below 16 px and returns above 20 px. Vertical cards cannot retain an overhead footprint, so steep views retain L1, with angular hysteresis at vertical ratios 0.80/0.84. Mesh changes run before visibility bounds update. Debug logging reports the selected level and estimated height.

## Validation and captures

`cargo build --profile opt-dev` passes. Strict `cargo clippy --profile opt-dev --all-targets -- -D warnings` passes. Both vegetation mip tests pass under the opt-dev profile. `glb_inspect.py` and `verify_oak.py` pass on the final GLB, including the far normal texture. Reports and build/test logs are copied into `tmp/runs/0014_vegetation-oak-v1/`.

The review page contains matched 40, 250 and 900 m views, an opposite-side 40 m view, a steep 900 m overhead view and a 22-second zoom/orbit recording. The camera focuses on (-495,-60), pitch 0.5 and yaw -0.7 for the main views. The opposite yaw is 2.4; overhead pitch is 1.3. All use 200k soldiers. The 900 m log selects level 2 at 11.7 px; the overhead log selects level 1 at 11.8 px. The recording logs the full 1→2→1→0→1 sequence with expected hysteresis thresholds.

Inspected the continuous recording through 88 sequential samples at 4 Hz, including both zoom directions and level transitions, rather than relying on one screenshot. The full video is available for playback in the review page; the tool interface does not play video directly. This is sampled motion inspection, not inspection of every video frame. The tree remains visible across the sequence; discrete level changes and the cross's flat geometry remain visible when magnified. No claim of a seamless crossfade or a final forest appearance.

The capture helpers only read the game window; no synthetic input was sent. Builds, Blender and launches were held while other games ran. Claude was conducting repeated shadow performance runs; Gota stopped that benchmark and explicitly released the machine for these captures. No other process was killed. No owned game remains at handover.

No GPU delta or under-1-ms claim: that belongs to the population and FL_VEG A/B stage. The screenshot fps overlay is incidental, not a benchmark. This diff changes only vegetation rendering and adds no FL switch, shared randomness, terrain heights, blocking or water changes. Diff review is the sim proof for this stage; no hash run was needed. The shadow branch still needs to be merged after its PR lands.

## Handover

Private iteration scripts, documentation, prompt/provenance and validation reports are committed as one folder with its README iteration row. PNGs, GLB, Blender scenes and render intermediates stay on disk, ignored. The public source diff stays available for review on feat/vegetation. No push or PR, no asset promotion or work on another species. Stop for Gota's review of this first oak.

Private iteration commit: `83ca726`, Add the first oak and its detail levels. Public source changes remain uncommitted for this review.
