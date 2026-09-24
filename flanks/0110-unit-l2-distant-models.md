# 0110: Distant L2 meshes for the remaining units
Written by GPT-6 Astra.

2026-09-24. After the knight L2 review, Gota requested the man-at-arms, spearman and archer L2s autonomously while AFK.

Built closed meshes from simple volumes, preserving each unit's main masses, equipment and animated part chains. Counts are 248, 250 and 250 triangles, all at 1.80 m head height. The spear reaches 2.52 m. Each candidate appends L2 to its original L0 GLB without changing the original binary data, atlas, material or existing nodes. Spearman and archer sidecars remain byte-identical.

The atlas projection and visibility fit keep mean linear RGB changes below 0.32%, 0.98% and 1.79% respectively. Visible team shares are 33.59% to 33.50%, 32.57% to 32.33%, and 43.74% to 44.43%. Both shields are closed volumes with wood backs, leather rims and straps. Archer strings and the held arrow are omitted at L2, while the quiver arrow fan remains.

Raw GLB checks show VEC4 colours, whole-number part IDs, matching pivot heights and zero new L2 open, nonmanifold, same-winding or degenerate defects. All retained moving parts pass 0 to 60 degree joint sweeps. Temporal attack reviews cover 90 sword, 64 spear and 150 archer frames, with connected tested joints throughout. Reviews include 20-, 8- and 3-pixel comparisons at 50 degrees, three silhouette overlays and paired attack videos.

Promoted the builders and supporting tools to `tools/blender/lods/`. They use installed GLBs plus compact source component labels, without requiring WIP scenes. Independent rebuilds from those inputs are byte-identical to all three candidates. Python compilation and formatting checks pass. Final videos and review-page links were verified.

Combined review: `assets_dev/lod_l2_review/index.html`. Candidate GLBs are under `assets_dev/<kind>/lod_l2_v1/`. Detailed delivery note, per-part counts, pivot positions, hashes and validation limitations: `tmp/notes/unit-l2-other-models-for-claude-2026-09-24.md`.

Existing L0 topology defects and the archer's double-sided material remain unchanged; new L2 geometry is closed, with review renders explicitly enabling culling. This finishes the requested L2 stage. Candidates remain uninstalled for Claude's integration under the original handoff. No src/ or renderer edits, staging or commits were made. The approved knight candidate was unchanged.
