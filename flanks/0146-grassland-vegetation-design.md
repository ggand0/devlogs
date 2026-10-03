# Grassland vegetation design
Written by GPT-6 Astra.

2026-09-27. Branch feat/vegetation, design stage only.

Read the vegetation handoff, graphics roadmap, worktree rules, writing guidelines, unit asset spec and Blender setup notes. Inspected vegetation.rs, terrain.rs, terrain shader, GLB texture loading and the deployment/spawn bounds. No source edits, builds, Blender sessions or game launches.

Gota supplied resources/map/m2tw_map_ss and grassland_map_ref.png while the proposal was in progress. Resized all sixteen screenshots into a contact sheet and inspected four individually at 1280 × 720: Pavia 19-59-19, Spanish Plain 20-02-57, Township 20-01-07 and 20-01-11. Viewed the generated reference at 1400 px wide. Original images remain untouched. Earlier external botanical searches were superseded by these project references.

The proposed direction is oak-dominated pasture with a lighter birch-like companion and hawthorn-like scrub. Birch replaces the handoff's suggested pine for this map, pending review. Shapes are art targets, not claimed species identifications from screenshots. The note records dimensions, triangle caps, texture sizes, a proposed vegetation GLB contract, chunked LOD rendering, alpha coverage, wind/shadow consistency and the first oak review gate.

Placement revealed a conflict in the handoff. The 1024 × 768 m field has two default 964 × 346 m deployment strips, covering 84.8% of its area. Large interior groves and long boundary hedges cannot coexist with keeping all deployment clear. The composition drawing therefore fits broken woodland into side margins, places small copses at the outer ends of the central gap, retains a 640 m open central corridor, and uses one short boundary hedge. Full crowns must respect exclusions. A broader scenery margin requires a separate map decision. No change to deployment or terrain is proposed in this branch.

Review artifacts: work/notes/023-vegetation-design-2026-09-27.md and work/notes/023-vegetation-design-2026-09-27/index.html. SVG plan embeds the existing layout unchanged and draws vector annotations over it. Reference images are resized private copies, not final assets. All counts are design budgets, not measured model or GPU results. The crossed far cards still require overhead and orbit review. New FL_VEG switch will require hash verification under the more specific AGENTS.md rule, despite the handoff's diff-only wording.

Stop here for direction feedback. Next accepted stage is one oak, L0/L1/card, placed legally and reviewed at 40/250/900 m with motion. No modeling or implementation started; nothing committed or pushed.
