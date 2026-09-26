# Spearman textured L0 and knight build promotion

2026-09-23 — GPT-6 Astra asset session.

Gota approved the textured knight v2 as the first repository version, then requested the next textured soldier using `resources/man_at_arms/menatarms_reference.png` and its visual prompt. The image equips the soldier with a spear and heater shield, unlike the original MAA contract's short sword/buckler. Gota clarified during work that this version is the **spearman**, with the same armor later used for a **falchion + heater shield MAA**.

## Knight repository work

Claude committed the texture pipeline and the approved knight while the asset work was running. `assets/units/knight.glb` is byte-identical to the reviewed `assets_dev/knight/textured_v2/knight_textured.glb`; it was not overwritten. The promoted build package under `tools/blender/` writes to ignored `assets_dev/knight/rebuild/`, has no dependency on ignored geometry scripts, and rebuilds all source material tiles.

An independent build from the promoted package matched all approved exported mesh attributes and pivot nodes exactly. The image checks passed and the source reproduced the 2,716-triangle L0. The shared GLB inspection helper now respects the accessor's `normalized` flag, so integer indices are not treated like normalized colors.

Commit **f3a5913**, `Add reproducible knight asset build scripts`, contains only eight build/documentation files under `tools/blender/`. No footer. Gota explicitly approved the staging/commit and requested a concise, plain commit message. No unrelated files were staged, and no push was performed.

## Spearman

Current review directory: `assets_dev/spearman/textured_v1/`. The earlier sword/buckler prototype under `assets_dev/man_at_arms/textured_v1/` is marked superseded and is not the requested MAA delivery.

Built a kettle hat with a sloping 35.2 cm brim, an open mail coif, face albedo, bare hands, short mail sleeves over quilted linen, plain surcoat, wool hose and low turnshoes. Surcoat/mail/quilt hems reach 62.2/58.5/53.5 cm above the ground. Fitted hose into the hip envelope and tucked the gambeson under the mail to remove visible intersections. Removed an internal cap exposed through the skirt split.

The spear is 2.50 m long, vertical and assigned with the right arm to `arm_spear` (ID 4). Its butt is 2 cm above the ground, so the whole asset reaches 2.52 m even though the human is 1.80 m. The left arm carries a medium heater shield with painted face, rawhide edging, wooden back and rear straps. A sheathed sidearm hangs at the hip. No symbols, plate armor, tall boots or gauntlets.

Fifteen original source tiles produce one 2048² RGBA color/AO/team atlas. No normal map, external model, copied reference pixels, image service, skeleton or animation clips. Original facial albedo replaced coarse black feature polygons. Iron was darkened to fit the reference's plain, worn equipment.

## Final measurements

2,961 triangles; 2,013 source vertices; 3,595 exported vertices. One L0 mesh/primitive, one opaque material, one embedded RGBA PNG. Every source vertex has exactly one full-weight group and all exported triangles have a single part ID.

| Part | ID | Source vertices | Exported vertices | Triangles | GLB pivot XYZ |
|---|---:|---:|---:|---:|---|
| body | 0 | 1011 | 1735 | 1445 | none |
| leg_l | 2 | 186 | 341 | 320 | (0.115, 0.910, 0) |
| leg_r | 3 | 186 | 341 | 320 | (-0.115, 0.910, 0) |
| arm_spear | 4 | 221 | 532 | 388 | (-0.224, 1.435, 0) |
| arm_shield | 5 | 409 | 646 | 488 | (0.224, 1.435, 0) |

Raw GLB checks pass for VEC4 `COLOR_0`, both UV sets, the ID/height pairs, all pivots, positive source triangle area, opaque material and atlas payload. The external and embedded atlas pixels match; zero mask alpha preserves fixed-color RGB. The exact spec export call is used. All final previews are rendered from the exported GLB after byte verification and re-import.

At a 50-degree camera, the nominal character-height targets are 60/20/8/3 px. Full sprite alpha bounds, including the spear, measure 70/23/7/1 rows at half-alpha. The thin shaft becomes subpixel at distance; authored far LODs need to preserve its visibility. These checks all use L0, not completed LODs.

## Integration finding and next step

The current loader derives scale from the complete mesh's top. A spear tip at 2.52 m would shrink this 1.80 m human to 71.4% of its intended display height. `work/handoffs/handoff-spearman-textured-l0-gpt6-astra.md` explains that character normalization should use body-part-0 maximum Y minus the ground while keeping full weapon bounds for culling. No engine workaround was made in the asset, and no source/renderer edits or Cargo runs were performed. Claude owns the loader correction and in-game verification.

The MAA variant will use a plain early falchion and heater shield after Gota reviews the spearman/shared armor. [Royal Armouries](https://royalarmouries.org/objects-and-stories/stories/the-hundred-years-war-1337-1453) documents falchions by the 13th century, and [Durham Cathedral](https://www.durhamcathedral.co.uk/explore/treasures-collections/our-most-famous-items/the-conyers-falchion) dates the Conyers example before 1272. This supports a broad 13th-century design, not a claim of dominance over arming swords in 1210–1240.

Stopped at spearman L0 appearance review. L1–L3, silhouette overlays, rotation acceptance and animation remain pending. No subagents, live Blender changes, synthetic desktop input, source changes or destructive commands. Spearman WIP, this devlog and the handoff remain uncommitted.
