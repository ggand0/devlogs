# Knight textured shape revision

2026-09-23 — GPT-6 Astra asset session.

Gota accepted the first textured knight in game and requested four refinements: a great helm closer to the supplied reference, an optional generic medieval pattern, a lower mail skirt hem and tapered boots resembling the French M2TW dismounted feudal knight. Animation was a capability question only; no animation work was authorized for this pass.

Built a separate L0 revision in `assets_dev/knight/textured_v2/`. Read the full unit asset contract and toolchain notes. Used the generated knight contact sheet plus the English/French M2TW screenshots as visual references. No subagents, live Blender scene access, renderer/source edits, Cargo commands, branch changes or commits. Hashes confirm preservation of eight prior assets and build scripts from the original knight and textured v1.

## Result

- Rebuilt the great helm with an oval section, nearly vertical side plates and a shaped chin. Shell width 22.2 cm, crown/lower width 20.6 cm. Actual eye openings have a dark recess. The brow reinforcement follows individual shell planes, correcting its initial intersection with the convex face.
- Extended the mail skirt from a 70 cm ground clearance to 42.5 cm, with front/rear riding splits. Surcoat hem is 53.5 cm, previously 58 cm. The mail now shows clearly below the cloth.
- Replaced the rounded feet with longitudinally shaped insteps and tapered toes, 35.3 cm heel to point. Boot cuffs reach 44.7 cm. Matching polygon sections keep leather bands outside the shaft. Hidden chausses terminate inside the cuffs, eliminating ankle intersections found in the first close-up.
- Baked a small flax-colored diamond border into the surcoat. Thread areas use zero team-mask alpha; cloth around them remains team-tinted. Geometric bands in a [13th-century European textile](https://www.clevelandart.org/art/1928.650) informed the treatment, but the repeat is original and no artifact/image pixels were copied. No faction emblems or heraldic insignia.

## Measurements and verification

The delivered L0 has 2,716 triangles, up 114 (4.4%) from v1, within the 2,000–3,000 budget. Height is 1.80 m, ground zero. There are 1,895 authoring vertices and 3,662 exported vertices. One primitive/material, OPAQUE, one embedded 2048² RGBA PNG. No normal maps, armature or clips.

| Part | Source vertices | Exported vertices | Triangles | Pivot in GLB XYZ |
|---|---:|---:|---:|---|
| body / 0 | 791 | 1602 | 1028 | none |
| arm_weapon / 1 | 223 | 622 | 412 | (-0.224, 1.435, 0) |
| leg_l / 2 | 254 | 437 | 416 | (0.115, 0.910, 0) |
| leg_r / 3 | 254 | 437 | 416 | (-0.115, 0.910, 0) |
| arm_shield / 5 | 373 | 564 | 444 | (0.224, 1.435, 0) |

All source vertices have one full-weight group. All exported triangles have a single part ID. Actual GLB byte checks establish `COLOR_0` VEC4, white RGB and retained vertex alpha; packed `TEXCOORD_0`; intact `TEXCOORD_1` ID/height pairs and all four pivots. `glb_inspect.py` ran on both exports. The atlas payload matches its external PNG pixel for pixel and retains RGB where mask alpha is zero. Allocation remains 16 MiB RGBA8, about 21.3 MiB with full mips. V2 runtime performance is unmeasured.

The local texture checker decodes raw index integers separately: the existing shared inspection helper normalizes all uint16 values for its color display, which is unsuitable for mesh indices. The shared helper was not changed. This correction in the new checker was verified against positive triangle coverage in all five parts.

Review images use the exported GLB, re-imported into an empty background Blender scene. Front/back red, front/side blue, helm/hem/boot close-ups, v1 comparison and four screen sizes are included. Nominal 60/20/8/3 px checks measure 60/20/7/2 rows at 50% alpha due to edge antialiasing. All four use L0 geometry.

## Handoff and next gate

`work/handoffs/handoff-knight-textured-v2-gpt6-astra.md` gives Claude the paths, exact texture/part contract, measurements and review scope. The asset README contains reproduction commands. No integration change is made here; atlas alpha remains authoritative for per-pixel team tint, including the new trim.

Stop for Gota's L0 appearance review. No further LODs, silhouette overlay, joint rotation test or animation were undertaken. The longer skirt remains in the existing rigid body part; motion clearance is not checked. Final promotion into `assets/units/` and `tools/blender/` remains pending acceptance. Keep this devlog and the handoff uncommitted.
