# Hill map first design
Written by GPT-6 Astra.

2026-10-15. Read handoff 120, the Big Grassland handoff and layout history, the original terrain proposal, the Grassland surface and planting decisions, plan 020, worktree/writing rules, and the eleven primary Hastings images. Inspected current graphics-tree terrain, camera, vegetation, formation packing, menu map enumeration, setup and selection-ring dimensions.

Created note `work/notes/089-hill-map-design-2026-10-15.md` and a review at `work/notes/vis/089-hill-design-v1/index.html`. The review includes a dimensioned PNG/SVG plan, centre profile, two CPU perspective studies at each of three height settings, measured design slopes and links to the supplied reference sheets. The reproducible drawing tool is `work/scripts/viz/hill_design_089.py`.

Proposed Hill: 1024 × 768 m, 880 m usable crest, centre z = −160, rise 36 m over 440 m, 400 m between deployment fronts. Big Hill doubles terrain distances and heights. Open approach grades on the 2 m design grid: median 9.66%, p95 10.38%, maximum 10.59%; centre mean 8.18%. The review compares 28/36/44 m rises. Formation arithmetic fits twenty 1,000-soldier blocks per side in two rows on Hill, and 100 per side in five rows on Big Hill, including 10 m block gaps. Deployment remains a proposal for Claude's later work.

The woods sit behind the reserve area and along the shoulders. Broken hedges and low scrub stay beside the main approach; the track crosses behind the defenders and descends the eastern shoulder. Proposed planting budgets are 480 trees/180 shrubs on Hill and 1600/500 on Big Hill; these are not measured engine counts. Trees and track width retain physical scale when the terrain doubles. Broad ground colour should follow Grassland's authored-layout approach, with fine grass at native scale.

Found stale assumptions in the handoff: the graphics tree is already on main (untracked `target-merge/` untouched), selection rings already use `terrain.grid_verts()`, and the demo's remaining size reference is explicitly Grassland. The menu reads `MapKind::ALL`. The significant implementation dependency is `build_terrain` using `is_large()` to select Big Grassland heights; shape dispatch must be independent of size for Big Hill.

The in-game height screenshot required before finalizing the drawing is outstanding. The note explicitly distinguishes this first layout from that later check; the CPU drawings do not stand in for a game run. No graphics branch, source edits, builds, game launches, Blender runs, commits or pushes. Stop at this design for review under the one-stage instruction. The next stage is the file-backed bare field, before planting and material polish.

## Visual HTML follow-up

Gota requested visuals in the HTML like the Grassland plan. Inspected `work/notes/023-vegetation-design-2026-09-27/index.html`: its presentation uses large landscape and species references with the placement diagram. Rebuilt the hill page around the same visual approach: a new landscape concept sheet first, then full-width supplied M2TW/Hastings approach and crest views, the two material/woodland reference sheets, the measured layout, and the height comparison.

Used the imagegen skill and built-in image generation tool for an original three-view landscape concept, saved as `work/notes/vis/089-hill-design-v1/landscape-concept-v1.png`, with its exact prompt beside it. The artwork is labelled as visual direction rather than measured geometry. All displayed images and height alternatives are embedded directly into the HTML (28.9 MiB), so displaying them no longer depends on relative reference paths or a complete sibling folder. Added click-to-enlarge viewing. Source: `index.template.html`; embedding script: `work/scripts/viz/hill_review_089.py`. No game assets or source code changed.

Verified eight embedded images decode, JavaScript syntax passes, and a separate headless Chrome with GPU rendering disabled loads every image and exercises all three height buttons and the image enlargement/close controls. Captured the page at `tmp/shots/hill-html-review/page-top.png`; browser results are `tmp/runs/hill-html-review/browser-check.json`. Closed the browser started for the check. The first sandboxed browser startup failed on a socket permission; the subsequent authorised headless run completed.
