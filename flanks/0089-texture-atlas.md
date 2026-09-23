# 0089: textured unit models (2026-09-23)

Branch `feat/knight-model`, commit b9b7a38. Astra's first textured knight (assets_dev/knight/textured_v1/knight_textured.glb, 2,602 tris, one 2048 px atlas) needed a texture path in the engine. The model file is not committed. Gota is still polishing it.

## What the engine does now

- The loader reads the model's one base colour texture from the GLB, decodes it as sRGB and builds a full mip chain at load, colour averaged in linear light (12 levels for 2048 px). Sampling is trilinear with 8x anisotropy.
- `TEXCOORD_0` is the atlas UV and rides the meshes as `ATTRIBUTE_UV_1`. Every unit mesh carries that channel, zero where there is no texture, so all levels share one vertex layout. `UV_0` stays (part id, pivot height).
- A textured model's vertex colour is forced to white with no team amount. The fragment draws `vertex colour * atlas rgb * mix(white, team, atlas alpha)`, then lighting, hit flash and death. The mask tints instead of replacing, so the cloth weave shows under the team colour.
- Each bucket entity carries its kind's atlas (`UnitAtlas`). The pulled path binds it at group 3 bindings 4 and 5 beside the storage buffers. The instanced path gets its own group 3 layout with the same two bindings. Untextured buckets and arrows bind one blank texel.
- Only buckets with an atlas compile the texture path (`UNIT_ATLAS` shader def, in both pipeline keys). Everything else keeps the old single-varying vertex colour path.
- The pulled vertex stays 48 bytes: vertex colour packed unorm8 and atlas UV packed unorm16.
- Derived far levels take their colour from the atlas. Each triangle is sampled at ten points, split into the part outside the team mask and the part inside, and turned into the vertex path's `mix(rgb, team, a)` form. That is exact where masked surfaces are grey, which the spec asks for.
- `FL_GLB_<KIND>=path` loads one kind from another file, for example `FL_GLB_KNIGHT=assets_dev/knight/textured_v1/knight_textured.glb`.

## Verified

- Close shots of both teams on both render paths: blue and orange tint the surcoat and shield with the weave intact, the helm slit and brass, belt, mail and boots read. tmp/shots/gait-natural/tex_arena_21s.png, tex_cpu_21s.png.
- Far levels at 120 m match the textured L0 at the same distance within antialiasing noise. The textured knight reads darker than the vertex colour one (mean soldier pixel 80,85,94 against 101,110,122 for the blue army), which is the asset: darker mail and boots, baked AO, a small team area.
- DIR fingerprints 19 of 19. `FL_GPU_CHECK=1` on DIR: 0 differences on 2,700 frames untextured and 8,400 textured.
- Build and clippy clean.

## Cost

200k knights, 40 m locked view, about 1,300 on L0. Unit pass × core clock, ms·MHz, two runs each, loadavg 9 to 12 from other jobs on the box:

| Build | ms·MHz |
|---|---|
| ccec1ea, before textures | 1,170 to 1,400 |
| b9b7a38, untextured knight | 1,050 to 1,160 |
| b9b7a38, textured knight, 8x anisotropy | 1,700 to 1,850 |
| same, anisotropy off (test only) | 1,430 to 1,590 |

The first version compiled the texture path for every bucket and cost 40 to 80 per cent more even untextured. Specializing on the atlas brought untextured back to the old cost. A textured L0 soldier costs about 45 per cent more than an untextured one, about 0.3 ms at full clock in this view. The GPU idles most of the frame, so the frame rate does not see it. The atlas is 21 MB of GPU memory with mips, shared by every knight.

## Open

- Which file the game loads by default. The loader still picks up `assets_dev/knight/knight.glb`, the untextured model. The textured one needs `FL_GLB_KNIGHT` until Astra writes the accepted version to that path.
- Team readability at distance: the textured knight's far levels are darker and less saturated than the code-built knight's. Asset side: more team area or lighter neutral cloth.
- Astra still owes authored L1 to L3. Derived far levels are a stopgap.

## v2 promoted (commit 316332b)

Gota accepted Astra's v2 (assets_dev/knight/textured_v2: narrower great helm, mail hanging below the surcoat, diamond trim on the hem, tapered boots, 2,716 tris). It is now `assets/units/knight.glb`, which the loader prefers over `assets_dev/`, so the game draws it with no override. Checked in game: the flax trim stays untinted through the atlas alpha, and the running legs clear the rigid mail skirt acceptably.

Stored with Git LFS (`.gitattributes`: `assets/units/*.glb`). The file is 5.9 MB, almost all of it the embedded atlas PNG, and every revision of every kind would otherwise stay in the public history for good. LFS hooks are installed in this clone, so a push uploads the object. Anything that builds a release must check out with LFS, or it gets a pointer file and the loader falls back to the code-built knight with an error.
