# Spearman arm articulation prototype

2026-09-23. Asset-side L0 review only, awaiting owner review and permission for
`src/` / renderer integration. No source files, accepted assets or earlier
build directories were edited; no commits.

Read `tmp/handoffs/HANDOFF-unit-animation-for-astra-2026-09-23.md`, the complete
asset spec and Blender setup notes. Inspected the current uncommitted shader
and follow-through code, loader, weapon-rebuild generator and offline arm
viewer. The existing renderer infers forearm membership from an elbow plane
and blends rotation angles across a continuous sleeve. Shoulder coverage is
not guaranteed by that representation. The readiness encoding also changes
from a negative stance band to positive attack progress: at zero wind-up,
`level` resets toward upright before `raise` grows. The prior offline viewer
uses older coefficients and a few isolated poses, so it cannot establish
continuity for the current complete animation.

Created `assets_dev/spearman/textured_v3/`, with an explicit upper arm (4), new
forearm (9), new hand (10), and existing weapon (8). This is a proposed contract
extension isolated from the approved spec and current runtime. Added closed
joint covers, including a shoulder socket assigned to the body. Distinguish
anatomical wrist from spear contact, chain every child from its parent's moved
joint, and keep attachment edges inside the covers across the motion.

Authored a bent guard and a 49.5 cm forward stab, holding the level spear 45 cm
from its butt. Merely interpolating endpoint joint rotations made the tip dip
12 cm; replaced that with two 33-sample angle tables. The resulting path stays
within 2.1 mm vertically during the thrust. Timeline uses the existing ten
30 Hz wind-up ticks and 0.6 s follow-through duration. The preview holds guard
between swings and never drops to carry at a phase boundary.

Measured from the GLB: L0 2,954 triangles; character 1.800 m; 100% part coverage;
one opaque material, one embedded 2048² RGBA atlas, COLOR_0 VEC4 and both UVs.
Ran the required `glb_inspect.py`. Body, face, shield and legs retain the prior
geometry; the revised atlas is rebaked from unchanged source texture tiles.

`check_motion.py` samples 2,504 carry/wind-up/strike/recovery poses using actual
exported geometry. It verifies source attachment probes against the GLB,
containment in each actual convex joint-cover mesh, shared-joint positions,
shaft contact, all triangle edge lengths, and phase continuity. Minimum inset:
shoulder 12.071 mm, elbow 10.607 mm, wrist 2.250 mm. Joint/contact/edge errors are
below 1e-12 m in float64 authoring math. No claim of game/GPU verification.

Deliverables: interactive `review.html`, `spear_stab.mp4`, `spear_stab.gif`,
`stab_contact_sheet.jpg`, playable `spear_stab_review.blend`, static
`spearman.glb`, `spear_stab.json`, and `motion_validation.json`. README describes
the exact new pivots, proposed IDs and engine integration changes. The current
loader rejects the new IDs; do not install this GLB before that integration.

Next accepted step: permission to integrate the new rig and sampled motion into
the loader and both renderer paths, preserving in-flight follow-through work.
Then verify in game. L1–L3, shieldwall motion, other units and full-body stepping
remain outside this review stage. No final asset promotion yet.

Saved-scene check: reload the actual playable .blend and compare each moving
part's world matrix against the authored motion over all 96 frames. This
caught glTF objects retaining quaternion mode while Euler channels were keyed;
the preview builder now explicitly selects Euler mode and linear keyframe
interpolation. The repaired scene passes with a matrix error below 1e-6.

Owner review and source promotion: Gota reviewed the motion positively, asked
for a handoff back to Claude, and authorized a commit. Promoted the geometry,
build/review/validation scripts, original texture tiles, and JSON pose tables
to tools/blender/spearman/. Output paths now default to the ignored
assets_dev/spearman/rebuild_v3/ directory, and the tools use the tracked shared
GLB inspector. A fresh rebuild passes the full motion and saved-scene checks
and reproduces the reviewed mesh attributes. The validator allows 1e-12 in
pose-table floats for Blender/system NumPy rounding differences.

Handoff: tmp/handoffs/HANDOFF-spearman-v3-for-claude-2026-09-23.md. The handoff
and this devlog remain uncommitted. Existing renderer edits and unrelated
files stay outside the source commit; installed unit assets remain unchanged.

Source commit: `320bc42` — `Add articulated spearman build and stab motion`. No footer.
