# Wall texture review

Written by GPT-6.

2026-10-23. Review 2 is built in `assets_dev/wall/textured_v1/`. Gota authorized texturing after discussing a possible later reduction from 10 m to 8 m thickness. The current delivery keeps 10 m. In-game width review with actual soldiers remains separate.

## Material

Created original stone with built-in imagegen: a four-quadrant source for rubble, paving, dressed blocks and plain carved granite. The exact prompt is `texture_prompt.txt`; the source is `stone_source.png`, 1254 x 1254. The generated source was copied into the project; the rebuild does not depend on the generator's output directory. No reference-game pixels were used in the asset.

Blender maps each sample at physical scale and bakes a shared 4096 x 4096 RGB base-colour atlas. Narrow wrap blending joins the source edges; periodic position coordinates keep subtle staining consistent at each 2 m module join. The low wall has mild ground darkening. One opaque material, roughness 0.9, no normal map, no baked scene lights. The atlas is about 30 MB as a PNG and embedded identically in each GLB; this is review output, not an engine memory/performance measurement.

The merlon proportions and footprint are unchanged. Face divisions at 0.45 m and 7.7 m permit dressed base and coping courses. The wall has 150 triangles, the landing 134. Both bounds are X [-1, 1], Y [0, 9.5], Z [-5, 5]. Merlon centre X = 0; 2 m pitch; normal walkway 8 m clear. The builder has a thickness parameter and samples in metres, so a later width change can preserve stone scale; only the current 10 m version was built and validated.

## Validation and review

Raw GLB inspection and `check_wall.py` pass. Zero open edges, nonmanifold edges, winding inconsistencies or degenerate triangles. The two ends match exactly. Early bisection gave sub-micrometre differences at the base-course vertices; snapping the generated coordinates to six decimals before export corrected the actual geometry. The texture checker confirms UVs inside the atlas, nonblack coverage at every triangle centre and byte-identical embedded PNGs. It uses glTF's exported image-coordinate convention.

Eight EEVEE views show individual outside/inside units, a 24 m run from both sides, close rubble and paving, and 40 m / 120 m Blender perspective views. The combined `review_sheet.jpg` and `review.html` show these for stone-size, colour, mortar, paving and weathering review. They are not in-game captures. The 2 m repetition remains visible in close views and belongs to review 3 after the stone treatment is reviewed. Do not present this as final repetition approval.

The prior shape iteration was archived on assets_dev main as the required provisional iteration commit, with its eight review images in LFS. The current texture iteration is not committed before review. Existing unrelated README and asset edits were preserved. No engine code, terrain or sim changed; no game, build or hash test was run. Checked the shared game runner before each Blender run.
