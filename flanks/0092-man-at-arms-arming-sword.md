# Man-at-arms with arming sword

2026-09-23 — GPT-6 Astra.

Gota accepted the spearman v2 face for now and requested the same body/armor/heater shield with an arming sword. This changes the current branch's earlier falchion plan; falchion variety is deferred.

Created ignored `assets_dev/man_at_arms/textured_v2/` from the accepted spearman. Its shared geometry source is byte-identical; twenty retained components and all fifteen source texture tiles match. Removed the spear and sheathed sidearm hilt, lengthened the empty scabbard, and added a straight 0.792 m blade with a shallow geometric fuller, straight crossguard, leather grip and disc pommel. The unchanged right arm plus sword maps to arm_weapon 1 and points +Z.

Measured export: 2,980 L0 triangles, 1,992 source vertices, 3,648 exported vertices; 1.80 m human/overall height; one opaque material and one embedded 2048² RGBA atlas. Part triangles: body 1,422; sword arm 430; legs 320 each; shield arm 488. Source vertex counts: body 970; sword arm 241; legs 186 each; shield arm 409. Pivots remain hips (±0.115, 0.910, 0), sword shoulder (-0.224, 1.435, 0), shield shoulder (0.224, 1.435, 0).

Raw GLB inspection, embedded/external image equality, COLOR_0 VEC4, UV1 part/pivot coverage, one part per triangle and sword orientation checks pass. Reviewed full-unit, sword/grip and 60/20/8/3-pixel renders from the actual GLB. All twelve recorded accepted spearman/knight input files remain byte-identical.

README and validation live beside the asset. Claude handoff: `tmp/drafts/handoff-man-at-arms-arming-sword-gpt6-astra.md`. No source/renderer edits, Cargo runs, live Blender changes, commits, animation or lower LODs. Stop for this L0 equipment review.
