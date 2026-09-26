# 0086: Textured knight L0 review (2026-09-23)

Gota reported that the first knight integrates well with no observed performance drop. New art target is between M2TW's dismounted feudal knight and Chivalry's Vanguard, guided by `resources/knight/knight_reference.png`. He explicitly excluded symbols and sigils.

Created a separate material review under `assets_dev/knight/textured_v1/`. The original `knight.glb`, `knight.blend` and `build_knight.py` are unchanged, verified by SHA-256 before and after. Worked on `feat/knight-model` without switching branches, editing source or renderer files, running Cargo, staging or committing. Existing concurrent source changes were left alone. No subagents, external image service, downloaded materials or live Blender operations were used. Background Blender ran with four CPU threads and the approved script prefix.

## Work

The knight has a single baked RGBA atlas with original mail, cloth, leather, metal, brass, shield paint and wood material tiles. `make_materials.py` deterministically generates the tiles. `build_knight_textured.py` unwraps the first UV layer, bakes color, team mask and local AO, packs the atlas and exports one material. No normal map or armature was added. The reference was used for art direction, not as texture pixels.

The shield cross and geometric mail accents are removed. Geometry adds leather forearm cuffs and taller boots, slightly lengthens the surcoat and tapers the great helm. Team-colored surfaces stay plain. Existing rigid-part IDs and pivots are preserved.

## Verified result

- L0: 2,602 triangles, 1.799999952 m height and zero ground offset. One opaque material, one mesh under `L0`, four pivot nodes.
- 1,744 authoring vertices with 100 percent single full-weight part coverage. Body 656, weapon arm 223, each leg 246, shield arm 373. Export has 3,439 vertices after splits.
- Exported hip pivots: (+0.115, 0.910, 0) and (-0.115, 0.910, 0). Shoulder pivots: (-0.224, 1.435, 0) and (+0.224, 1.435, 0).
- `TEXCOORD_0` is the packed atlas layout. `TEXCOORD_1` contains exactly (0, 0), (1, 1.435), (2, 0.910), (3, 0.910), (5, 1.435).
- `COLOR_0` is VEC4 with white RGB multipliers and team alpha values 0, 0.12, 0.16 and 1. White RGB prevents double coloring when glTF multiplies the atlas by vertex colors. No second color layer ships.
- One embedded 2048 × 2048 RGBA PNG, 5,666,840 bytes. Pixel-identical to the external atlas. Mask spans 0 to 255. 2,458,243 pixels have mask zero and still retain nonzero material RGB.
- Shared RGBA8 storage: 16 MiB without mips, about 21.3 MiB with a full chain. No in-game performance claim for this textured version.
- Front, back and blue-team renders use the re-imported GLB with the explicit team tint. The 50-degree L0 screen targets are 60, 20, 8 and 3 px. Measured half-alpha coverage is 60, 20, 8 and 2 rows.

The build asserts geometry and GLB contracts. The required `glb_inspect.py` was run, with output saved. `check_texture.py` independently checks the embedded image bytes, compares them to the external PNG and checks color alpha. `compose_review.py` writes the actual render sheets without retouching.

## Findings and engine handoff

Linear float bake data needs explicit sRGB encoding when written into a byte image. The first bake exposed the missing conversion as overly dark mail and leather. This was corrected in the build source. Also, Blender 5.2's exporter replaces the primary color layer with white, including alpha, when a second exported color layer overwrites its material association. Read the installed exporter to identify the cause, kept exactly one color layer and asserted its exported alpha. The Blender installation was not modified.

Textures use neutral RGB in team regions and a linear alpha mask. The preview formula is `atlas.rgb * mix(white, team_tint, atlas.a)` in linear color space. Team alpha is not opacity. The current shader's replacement blend would erase cloth detail at mask 1. The current loader also does not retain atlas UVs or images. This work is an asset preview, not a renderer integration.

Handoff is `work/handoffs/handoff-knight-textures-gpt6-astra.md`. It covers atlas loading, UV retention, fragment sampling, sRGB, tinting, mipmaps and the need to include atlas color in the visibility-based derived LOD calculation. White `COLOR_0.rgb` must not be used as the new far-color source.

Stop for the owner's material and shape review. L1 to L3, the silhouette overlay, rotation checks and full acceptance remain pending. No final asset or tools promotion yet.
